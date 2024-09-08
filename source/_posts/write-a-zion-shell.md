---
title: 如何用 C 语言编写一个 Shell
date: 2024-07-14 19:45:43
cover: https://i.imgur.com/bcF5qZr.jpeg
thumbnail: https://i.imgur.com/bcF5qZr.jpeg
categories: ['编程', '扫盲']
tags:
    - intro
toc: true
excerpt: Shell，或者说 Unix shell，本质上是一个命令行解释器。它是一个计算机程序，允许用户通过命令行与操作系统交互。Shell 有两种界面形式：命令行界面 (CLI) 和图形用户界面 (GUI)（例如 Windows 中的文件浏览器）。常见的 CLI 有 `sh`、`zsh`、`bash` 等，其中 `sh` 是第一个流行的 shell，`bash` 是 Linux 系统自带的 shell，而 `zsh` 则是最受欢迎的 shell 之一。
---

## Shell 具体指的是什么？

**Shell**，或者说 **Unix shell**，本质上是一个命令行解释器。它是一个计算机程序，允许用户通过命令行与操作系统交互。Shell 有两种界面形式：命令行界面 (CLI) 和图形用户界面 (GUI)，例如 Windows 中的文件浏览器。常见的 CLI 有 `sh`、`zsh`、`bash` 等，其中，`sh` 是第一个流行的 shell，`bash` 是 Linux 系统自带的 shell，而 `zsh` 则是最受欢迎的 shell 之一。

当你登录到一个类 Unix 系统时，shell 会自动在这个会话中运行。你也可以输入以下命令查看当前 shell 和系统内置的 shell 列表：

```sh
vagrant@ubuntu-xenial:~$ echo $SHELL
/bin/bash
vagrant@ubuntu-xenial:~$ cat /etc/shells
# /etc/shells: valid login shells
/bin/sh
/bin/dash
/bin/bash
/bin/rbash
/usr/bin/tmux
/usr/bin/screen
```

### Shell 的生命周期

作为一个程序，shell 也有自己的生命周期：

* **初始化：** 读取与该 shell 配套的配置文件，例如`.zshrc`、`.condarc` 等。
* **解释：** 循环读取命令行并执行命令。
* **终止：** 检测到异常或者接收到终止命令，释放资源然后退出。

## 如何编写一个 Shell？

### 实现难点

Shell 并不负责命令行的真正执行，而是通过 `execvp` 系统调用来执行命令。根据手册，`execvp` 是一个更加友好的 `exec` 方法，其中 `v` 表示支持一组可变参数，`p` 表示无需提供目标程序的完整路径。

![手册描述截图](https://i.imgur.com/xZAtGwG.png)

虽然 shell 像是一个搬运工，但在执行某些特殊命令时，仍然需要单独处理。例如 `cd` 命令，它允许你更改当前目录。然而，“当前目录” 是一个进程的属性，如果让子程序执行，shell 的 “当前目录” 不会发生变化。同理，还有 `exit` 命令。

### 代码细节

**初步框架：**

```cpp
int main() {
    ZionSH *zionSh = new ZionSH();
    zionSh->initialize();
    int terminate = zionSh->interpret();
    return terminate;
}
```

```cpp
void ZionSH::initialize() {
    std::cout << "Zion Shell initializing..." << std::endl;
}

int ZionSH::interpret() {
    char *line;
    char **args;
    int status;
    do {
        std::cout << "ZionSh>";
        line = read_line();
        args = split_line(line);
        status = execute(args);
        // 释放存储空间
        free(line);
        free(args);
    } while (status);
}
```

在执行 `exec` 时，它会用目标程序替换当前程序，除非发生错误。因此，我们需要用 `fork` 函数生成子程序，并通过得到的进程 ID 判断是子进程还是父进程。

```cpp
int ZionSH::execute(char **args) {
    int status;
    pid_t pid = fork();
    if (pid == 0) {
        if (execvp(args[0], args) == -1) {
            fprintf(stderr, "Fail to execute the input command.\n");
            return EXIT_FAILURE;
        }
    } else if (pid < 0) {
        fprintf(stderr, "Fail to fork.\n");
        return EXIT_FAILURE;
    } else {
        do {
            waitpid(pid, &status, WUNTRACED);
        } while (!WIFEXITED(status) && !WIFSIGNALED(status));
    }
    return 0;
}
```

主进程需要等待子进程运行结束后再退出，这里用到的 `waitpid` 和 `WIFEXITED`、`WIFSIGNALED` 可以通过手册查询。

需要参考的指令如下：

* [fork](https://man7.org/linux/man-pages/man2/fork.2.html) ![fork 方法](https://i.imgur.com/AYPgnU0.png)
* [waitpid](https://man7.org/linux/man-pages/man2/waitpid.2.html) ![waitpid 方法](https://i.imgur.com/6NqfqWM.png)
* [chdir](https://man7.org/linux/man-pages/man2/chdir.2.html) ![chdir 方法](https://i.imgur.com/GTgwgBm.png)

### 代码实现

```cpp
#include "../headers/zionSh.h"

int cd(char **args);
int help(char **args);
int exit(char **args);

int (*sp_func[]) (char **) = {
    &cd,
    &help,
    &exit
};

std::string sp_func_name[] = {
    "cd",
    "help",
    "exit"
};

void ZionSH::initialize() {
    std::cout << "Zion Shell initializing..." << std::endl;
}

int ZionSH::interpret() {
    char *line;
    char **args;
    do {
        std::cout << "ZionSh>";
        line = read_line();
        args = split_line(line);
        int status = -1;
        for (int i = 0; i < num_sp_func(); i++) {
            if (sp_func_name[i] == args[0]) {
                status = (*sp_func[i])(args);
                if (status == 0) {
                    return EXIT_SUCCESS;
                }
            }
        }
        if (status == -1) {
            status = execute(args);
            if (status == EXIT_FAILURE) {
                return EXIT_FAILURE;
            }
        }
        // 释放存储空间
        free(line);
        free(args);
    } while (true);
}

char *ZionSH::read_line() {
    int curr = 0;
    int buffSize = BUFF_SIZE;
    char *line = static_cast<char *>(malloc(buffSize * sizeof(char)));
    int c;

    if (!line) {
        fprintf(stderr, "allocation error!");
        exit(EXIT_FAILURE);
    }

    while (true) {
        c = getchar();
        if (c == EOF || c == '\n') {
            line[curr] = '\0';
            return line;
        }
        line[curr] = (char) c;
        curr++;
        if (curr >= buffSize) {
            buffSize += BUFF_SIZE;
            void *newBuffer = realloc(line, buffSize);
            if (!newBuffer) {
                fprintf(stderr, "reallocation error!");
                exit(EXIT_FAILURE);
            }
            line = static_cast<char *>(newBuffer);
        }
    }
}

char **ZionSH::split_line(char *line) {
    char **args = static_cast<char **>(malloc(ARG_SIZE * sizeof(char *)));
    if (!args) {
        fprintf(stderr, "Fail to allocate.\n");
        exit(EXIT_FAILURE);
    }
    char *token;
    int num_args = 0;
    while (true) {
        token = strtok(line, TOK_DELIM);
        if (token == nullptr) break;
        args[num_args] = token;
        num_args++;
        if (num_args % ARG_SIZE == 0) {
            char **ex_args = static_cast<char **>(realloc(args, (num_args + ARG_SIZE) * sizeof(char *)));
            args = ex_args;
            if (!args) {
                fprintf(stderr, "Fail to reallocate.\n");
                exit(EXIT_FAILURE);
            }
        }
    }
    args[num_args] = nullptr;
    return args;
}

int ZionSH::execute(char **args) {
    int status;
    pid_t pid = fork();
    if (pid == 0) {
        if (execvp(args[0], args) == -1) {
            fprintf(stderr, "Fail to execute the input command.\n");
            return EXIT_FAILURE;
        }
    } else if (pid < 0) {
        fprintf(stderr, "Fail to fork.\n");
        return EXIT_FAILURE;
    } else {
        do {
            waitpid(pid, &status, WUNTRACED);
        } while (!WIFEXITED(status) && !WIFSIGNALED(status));
    }
    return 0;
}

int cd(char **args) {
    if (args[1] == nullptr) {
        fprintf(stderr, "No path for cd\n");
    } else if (chdir(args[1]) == -1) {
        fprintf(stderr, "Error change dir: %s\n", strerror(errno));
    }
    return 1;
}

int help(char **args) {
    std::cout << "help statements.";
    return 1;
}

int exit(char **args) {
    return EXIT_SUCCESS;
}

int ZionSH::num_sp_func() {
    return sizeof sp_func_name / sizeof(std::string);
}
```

一个基本的 shell 程序框架就实现好了，它能够初始化、读取和解释命令行输入，并通过 `fork` 和 `execvp` 系统调用来执行命令。此外，它还包含了几个特殊命令的处理，如 `cd` 和 `exit`。
