---
title: 6.Null-4 命令行环境部署
date: 2022-12-17 12:48:33
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img001.jpg
categories: [编程, 编程.Labs, MIT 6.Null]
toc: true
---

本文主要介绍如何在**shell**中运行多个进程，如何停止或暂停进程以及如何让进程在后台运行。之后还会讲解如何配置**dotfile**以及用**SSH**处理远程计算机。
<!--more-->

## Job控制

当我们需要中断**job**时，通常是因为命令完成时间过长（比如因为网络问题导致`brew update`卡住），可以执行`Ctrl-C`，它本质上是让**shell**向进程传递`SIGINT`信号。**shell**使用一种称为信号的**UNIX**通信机制向进程传递信息从而改变执行流程，因此<u>信号是软件中断</u>。

下面的**Python**程序会捕获`SIGINT`信号，我们如果想要停止它，可以使用`SIGQUIT`，即键入`Ctrl-\`。

```python
#!/usr/bin/python3
import signal, time

def handler(signum, time):
	print("\nI got a SIGINT, but I am not stopping")

signal.signal(signal.SIGINT, handler)
i = 0
while True:
	time.sleep(.1)
	print("\r{}".format(i), end="")
	i += 1
```

```shell
~/tmp ❯ ./sigin.py
11^C
I got a SIGINT, but I am not stopping
25^C
I got a SIGINT, but I am not stopping
43^\[3]    35733 quit       ./sigin.py
```

在终端中打印显示的`^`就是`Ctrl`。除了`SIGINT`和`SIGQUIT`外，还有`SIGSTOP`也很常用，键入`Ctrl-Z`可以让作业暂停。此外，我们还可以用[fg](https://www.man7.org/linux/man-pages/man1/fg.1p.html)或者[bg](https://man7.org/linux/man-pages/man1/bg.1p.html)分别继续前台或后台暂停的作业。

```shell
~ ❯ jobs
~ ❯ sleep 1000
^Z
[1]  + 36791 suspended  sleep 1000
~ ❯ nohup sleep 2000 &
[2] 36793
appending output to nohup.out
~ ❯ jobs
[1]  + suspended  sleep 1000
[2]  - running    nohup sleep 2000
~ ❯ bg %1
[1]  - 36791 continued  sleep 1000
~ ❯ jobs
[1]  - running    sleep 1000
[2]  + running    nohup sleep 2000
~ ❯ kill -STOP %1
[1]  + 36791 suspended (signal)  sleep 1000
~ ❯ jobs
[1]  + suspended (signal)  sleep 1000
[2]  - running    nohup sleep 2000
~ ❯ kill -HUP %1
[1]  + 36791 hangup     sleep 1000
~ ❯ jobs
[2]  + running    nohup sleep 2000
~ ❯ kill -HUP %2
~ ❯ jobs
[2]  + running    nohup sleep 2000
```

点击[here](https://en.wikipedia.org/wiki/Signal_(IPC))可以查看更多信号相关信息，或者直接键入`man signal`查看。

## 终端多路复用器

一次性运行多个任务的时候，我们可以同时开多个终端窗口监视它们进度，但在命令行界面使用终端多路复用器是一种更加通用的解决方案，<u>尤其在远程连接机器的时候</u>。

![tmux demo](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/tmux.png)

最流行的终端多路复用器就是**tmux**，可以自己配置快捷键来快速创建和切换标签和界面。在**tmux**中，一个**session**就是一个独立的工作空间可以有多个窗口，而一个窗口就是一个可视标签同一个**session**可以切割出多个窗口，一个窗口可以分为多个**pane**。

- Sessions
  - `tmux` 打开一个新的session
  - `tmux new -s NAME `打开一个新的session并命名
  - `tmux ls` 列出当前sessions
  - `<C-b> d` 在session内部键入这个命令可以退出当前session
  - `tmux a` 进入上一个session，也可以用-t进入指定session
- Windows
  - `<C-b> c` 创建一个新窗口
  - `<C-b> N` 切换到第N个窗口
  - `<C-b> p` 切换到上一个窗口
  - `<C-b> n` 切换到下一个窗口
  - `<C-b> ，`重命名当前窗口
  - `<C-b> w` 列出所有窗口
- Panes
  - `<C-b> " `水平切割窗口
  - `<C-b> %` 竖直切割窗口
  - `<C-b> <direction>` 朝特定方向移动pane
  - `<C-b> z` 放大当前pane

这里有关于[tmux](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/)的快速教程。

## 别名

**shell**支持***aliasing***，即用一个较短的指令作为一个长命令的别名从而减少我们的工作量，通常的用法如下：

```sh
alias alias_name="command_to_alias arg1 arg2"
```

比较常用过的设定有：

```sh
# Make shorthands for common flags
alias ll="ls -lh"

# Save a lot of typing for common commands
alias gs="git status"
alias gc="git commit"
alias v="vim"

# Save you from mistyping
alias sl=ls

# Overwrite existing commands for better defaults
alias mv="mv -i"           # -i prompts before overwrite
alias mkdir="mkdir -p"     # -p make parent dirs as needed
alias df="df -h"           # -h prints human readable format

# Alias can be composed
alias la="ls -A"
alias lla="la -l"

# To ignore an alias run it prepended with \
\ls
# Or disable an alias altogether with unalias
unalias la

# To get an alias definition just call it with alias
alias ll
# Will print ll='ls -lh'
```

如果要将别名持久化，我们需要将其配置到**shell**启动文件中，比如**.bashrc**或者**.zshrc**，这些文件又被称为***dotfile***。

## Dotfiles

许多程序都会用一个以`.`开头的文本文件作为配置文件，我们称这些文件为**dotfiles**，当我们输入ls时它们默认隐藏。**shell**在启动时会读取很多文件并加载其配置，根据**shell**的不同，无论是登录还是交互都很复杂，[这里](https://blog.flowblok.id.au/2013-02/shell-startup-scripts.html)是关于这个问题的参考。

<u>每个人都会对自己的**dotfiles**有特别的配置，除了助教们的配置外[Anish](https://github.com/anishathalye/dotfiles), [Jon](https://github.com/jonhoo/configs), [Jose](https://github.com/jjgo/dotfiles)，[这里](https://dotfiles.github.io)有一些很棒的资料可以参考。</u>



## 远程连接

我们可以通过使用*Secure Shell*即**SSH**来远程连接计算机，通常输入这样的一条指令:

```sh
ssh foo@bar.mid.edu
```

这里我们尝试以用户foo去登录服务器bar.mid.edu，服务器既可以用**URL**表示也可以用**IP**表示。我们利用`ssh-keygen`生成密钥对，当有多套密钥对时可以用`ssh-agent`去管理。

```sh
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

为了不用每次都输入密码这么麻烦，我们可以将我们的公钥复制到远程计算机，因为**ssh**会检查`.ssh/authorized_keys`来决定哪些客户可以直接连接：

```sh
# 直接复制文件
cat .ssh/id_ed25519.pub | ssh foobar@remote 'cat >> ~/.ssh/authorized_keys'
# ssh-copy-id
ssh-copy-id -i .ssh/id_ed25519 foobar@remote
```

当我们想通过`ssh`复制文件时，可以用`scp`命令`scp path/to/local_file remote_host:path/to/remote_file`



## Exercises

1.通过类似ps aux ｜ grep这样的命令去查找并杀死一个job。现在启动一个job，指令是sleep 1000，然后用Ctrl-Z暂停:

```sh
❯ sleep 10000
^Z
[1]  + 44133 suspended  sleep 10000
❯ bg
[1]  + 44133 continued  sleep 10000
❯ jobs
[1]  + running    sleep 10000
❯ pgrep -af sleep
44133
❯ pkill -HUP sleep
[1]  + 44133 hangup     sleep 10000
❯ jobs
~ ❯
```

2.
