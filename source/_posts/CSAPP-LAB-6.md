---
title: CSAPP Lab6 Writing Your Own Unix Shell
date: 2022-06-06 12:00:00
cover: /img/cover/img257.jpg
categories: [编程, 编程.Labs, CMU 15-213]
toc: true
---

## 准备

除了看[shlab.dvi (cmu.edu)](http://csapp.cs.cmu.edu/3e/shlab.pdf)和[Introduction to Computer Systems 15-213/18-243, spring 2009 (cmu.edu)](https://www.cs.cmu.edu/~213/lectures/15-ecf-signals.pdf)外，这个实验还需要仔细阅读书上第八章异常控制流部分，而书中未补全代码可在[csapp.cs.cmu.edu/3e/ics3/code/src/csapp.c](http://csapp.cs.cmu.edu/3e/ics3/code/src/csapp.c)搜索（比如Kill）。
<!--more-->

由于我基本是按照书上介绍内容照葫芦画瓢，实际的个人思考很少，个人认为比较重要的点是对照tshref.out（也可以用make rtestx输出）差异去不断修改。具体来说，我们按照原代码框架顺序编写eval、builtin_cmd、do_bgfg、waitfg和另外3个handler即可。

另外值得注意的是很多特殊变量不需要我们定义，比如SIGCONT，我的错误示范：

```c
/* signal.h中未定义信号 */
#define SIGCONT 18 /* 用在bg和fg指令 */
```

## eval、builtin_cmd、do_bgfg、waitfg

eval部分代码直接参照8-24，而其中锁的部分参考8.5.4和8-40，lab在Hints中也有提到需要在fork子进程前用sigprocmask锁住SIGCHLD信号，防止子进程提前终止导致父进程添加僵尸进程。当然，这里一开始我是完全没考虑并发冲突之类的。

```c
void eval(char *cmdline) {
    char *argv[MAXARGS];
    char buf[MAXLINE];
    int bg;
    pid_t pid;

    strcpy(buf, cmdline);
    bg = parseline(buf, argv);
    // 输入空命令行
    if (argv[0] == NULL)
        return;

    sigset_t mask_all, mask_one, prev_one;

    Sigfillset(&mask_all);
    Sigemptyset(&mask_one);
    Sigaddset(&mask_one, SIGCHLD);

    if (!builtin_cmd(argv)) {
        Sigprocmask(SIG_BLOCK, &mask_one, &prev_one);
        if ((pid = Fork()) == 0) {
            Sigprocmask(SIG_SETMASK, &prev_one, NULL);
            Setpgid(0, 0);
            Execve(argv[0], argv, environ);
        }

        // 添加子任务到jobs
        Sigprocmask(SIG_BLOCK, &mask_all, NULL);
        int state = bg ? BG : FG;
        addjob(jobs, pid, state, cmdline);
        Sigprocmask(SIG_SETMASK, &prev_one, NULL); // 解锁

        // 父进程等待前台job终止
        if (!bg) {
            waitfg(pid);
        } else {
            printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
        }
    }
    return;
}
```

然后，编写buildtin_cmd，注意这个方法属于主线程，因此我们用exit(0)可以跳出循环安全退出，当然具体代码参照8-24编写即可。

```c
int builtin_cmd(char **argv) {
    // quit
    if (!strcmp(argv[0], "quit")) {
        exit(0);
    }

    // jobs
    if (!strcmp(argv[0], "jobs")) {
        listjobs(jobs);
        return 1;
    }

    // bg | fg <job>
    if (!strcmp(argv[0], "bg") || !strcmp(argv[0], "fg")) {
        do_bgfg(argv);
        return 1;
    }

    return 0;     /* not a builtin command */
}
```

在编写do_bgfg的时候，要注意和参考输出做对比，这里我推荐test15，基本囊括了所有情况：

```c
void do_bgfg(char **argv) {
    int bg = !strcmp(argv[0], "bg");
    if (argv[1] == NULL) {
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    }

    pid_t pid;
    struct job_t *job;
    if (argv[1][0] == '%') {
        // job_id
        int jid = atoi(argv[1] + 1);
        if (jid == 0) {
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }

        if ((job = getjobjid(jobs, jid)) == NULL) {
            printf("%s: No such job\n", argv[1]);
            return;
        }
        pid = job->pid;
    } else {
        // pid
        pid = atoi(argv[1]);
        if (pid == 0) {
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }

        if ((job = getjobpid(jobs, pid)) == NULL) {
            printf("(%s): No such process\n", argv[1]);
            return;
        }
    }

    Kill(-pid, SIGCONT);
    if (bg) {
        job->state = BG;
        printf("[%d] (%d) %s", job->jid, pid, job->cmdline);
    } else {
        job->state = FG;
        waitfg(pid);
    }
}
```

waitfg我是参考8.5.7，循环阻塞式检测是否还有前台任务。

```c
void waitfg(pid_t pid) {
    sigset_t mask;
    Sigemptyset(&mask);
    while (fgpid(jobs) != 0) {
        sigsuspend(&mask);
    }
}
```

## Signal handlers

我们创建的任务会有多种运行场景，收到信号SIGINT强制进程结束（键入ctrl+c也可以发出相同信号），收到SIGSTOP或者SIGTSTP（等效于ctrl+z）而暂停进程，后两者本质差不多，但还是有区别[unix - What is the difference between SIGSTOP and SIGTSTP? - Stack Overflow](https://stackoverflow.com/questions/11886812/what-is-the-difference-between-sigstop-and-sigtstp)。当然我们需要发送的另一个指令SIGCONT可以让暂停进程继续运行，通常和SIGSTOP成对出现。

理解了信号的使用后，再借助8.4.3的内容，我们可以用信号去结束、回收、重启我们的子进程：

```c
void sigchld_handler(int sig) {
    int olderrno = errno, status;
    pid_t pid;
    sigset_t mask_all, prev_all;
    Sigfillset(&mask_all);
    while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) {
        Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
        if (WIFEXITED(status)) {
            // 正常返回
            deletejob(jobs, pid);
        } else if (WIFSIGNALED(status)) {
            // TSTP
            printf("Job [%d] (%d) terminated by signal %d\n", pid2jid(pid), pid, WTERMSIG(status));
            deletejob(jobs, pid);
        } else if (WIFSTOPPED(status)) {
            // STOP
            printf("Job [%d] (%d) stopped by signal %d\n", pid2jid(pid), pid, WSTOPSIG(status));
            struct job_t *job = getjobpid(jobs, pid);
            job->state = ST;
        }
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
    }
    errno = olderrno;
}
 
void sigint_handler(int sig) {
    pid_t pid;
    sigset_t mask_all, prev_all;
    Sigfillset(&mask_all);
    Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
    if ((pid = fgpid(jobs)) > 0) {
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
        Kill(-pid, SIGINT);
    }
}

void sigtstp_handler(int sig) {
    pid_t pid;
    sigset_t mask_all, prev_all;
    Sigfillset(&mask_all);
    Sigprocmask(SIG_BLOCK, &mask_all, &prev_all);
    if ((pid = fgpid(jobs)) > 0) {
        Sigprocmask(SIG_SETMASK, &prev_all, NULL);
        Kill(-pid, SIGTSTP);
    }
}
```

收官~