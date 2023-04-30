---
title: MIT 6.Null 命令行环境部署
date: 2022-12-17 12:48:33
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img001.jpg
categories: [编程, Labs, MIT 6.Null]
tags: 
    - shell
toc: true
---
长期使用 **Shell** 的程序员应该如何通过优化工作流来节约时间呢？我想可以从以下四点出发，一是提高任务并行度，通过 **Job控制** 和 **TMUX** 配合；二是简化或优化命令，通过 **Aliase** 和 **Dotfiles** 去配置实现；三是远程访问；四是美化 **Shell** 界面，良好的ui也能极大提高工作效率。
<!--more-->

## Job控制

当我们需要中断 **Job** 时，通常是因为命令完成时间过长（比如因为网络问题导致`brew update`卡住），通常会按下`<Ctrl c>`。这个组合键本质上是让 **Shell** 向当前进程传递信号`SIGINT`。

**Shell** 使用一种称为 **信号** 的 **unix通信机制** 向进程传递信息从而改变执行流程，因此信号是 **软件中断**。

### 杀死进程

**杀死进程** 有两种方式，键入`Ctrl c`从而发送`SIGINT`信号给进程，或者键入`Ctrl \`从而发送`SIGQUIT`信号给进程。

```python
#! /usr.bin/env python
import signal, time

def handler(signum, time):
    print("\ni got a sigint, but i am not stopping")

signal.signal(signal.sigint, handler)
i = 0
while true:
    time.sleep(.1)
    print("\r{}".format(i), end="")
    i += 1
```

![kill process](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/Kill-process.png)

当然，一种更优雅直接的方式是向进程传递`SIGTERM`信号，利用[kill](https://www.man7.org/linux/man-pages/man1/kill.1.html)命令向进程发送信号。

![kill -l](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/kill-15.png)

### 暂停或将进程放入后台

想要让运行的 **Job** 暂停，可以键入`Ctrl z`发送信号`SIGTSTP`，这里面的 **TSTP** 是 **Terminal Stop** 的缩写。而之后则可以使用[bg](https://man7.org/linux/man-pages/man1/bg.1p.html)或者[fg](https://www.man7.org/linux/man-pages/man1/fg.1p.html)命令将暂停的任务继续运行。

![前后台Job控制](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/fg-bg-demo.png)

想要查看更多信号相关信息，可以点击[here](https://en.wikipedia.org/wiki/Signal_(IPC))，或者直接键入`man signal`在终端查看。

## 终端多路复用器

一次性运行多个任务的时候，我们可以同时开多个终端窗口监视它们进度，但在命令行界面使用终端多路复用器是一种更加通用的解决方案，尤其在远程连接机器的时候。

![tmux demo](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/tmux.png)

最流行的终端多路复用器就是 **TMUX**，可以自己配置快捷键来快速创建和切换标签和界面。在 **TMUX** 中，一个 **Session** 就是一个独立的工作空间可以有多个窗口，而一个窗口就是一个可视标签同一个**Session**可以切割出多个窗口，一个窗口可以分为多个 **Pane**。

- Sessions
  - `tmux` 打开一个新的session
  - `tmux new -s NAME`打开一个新的session并命名
  - `tmux ls` 列出当前sessions
  - `<Ctrl b> d` 在session内部键入这个命令可以退出当前session
  - `tmux a` 进入上一个session，也可以用-t进入指定session
- Windows
  - `<Ctrl b> c` 创建一个新窗口
  - `<Ctrl b> N` 切换到第N个窗口
  - `<Ctrl b> p` 切换到上一个窗口
  - `<Ctrl b> n` 切换到下一个窗口
  - `<Ctrl b> ，`重命名当前窗口
  - `<Ctrl b> w` 列出所有窗口
- Panes
  - `<Ctrl b> "`水平切割窗口
  - `<Ctrl b> %` 竖直切割窗口
  - `<Ctrl b> <direction>` 朝特定方向移动pane
  - `<Ctrl b> z` 放大当前pane

这里有关于[tmux](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/)的快速教程。

## Aliases

**Shell** 支持 **Aliasing**，即用一个较短的指令作为一个长命令的别名从而减少我们的工作量，通常的用法如下：

```sh
alias alias_name="command_to_alias arg1 arg2"
```

比较常用过的设定有：

```sh
# User specific aliases and functions
alias ll="ls -lh"

alias gs="git status"
alias gc="git commit"
alias v="vim"

alias sl=ls

alias mv="mv -i"           # -i prompts before overwrite
alias mkdir="mkdir -p"     # -p make parent dirs as needed
alias df="df -h"           # -h prints human readable format

alias la="ls -A"
alias lla="la -l"
```

如果要将别名持久化，我们需要将其配置到 **shell** 启动文件中，比如 **.bashrc** 或者 **.zshrc**，这些文件又被称为 **Dotfile**。

## Dotfiles

许多程序都会用一个以`.`开头的文本文件作为配置文件，我们称这些文件为 **Dotfiles**，当我们输入ls时它们默认隐藏。**Shell** 在启动时会读取很多文件并加载其配置，根据 **Shell** 的不同，无论是登录还是交互都很复杂，[这里](https://blog.flowblok.id.au/2013-02/shell-startup-scripts.html)是关于这个问题的参考。

每个人都会对自己的 **Dotfiles** 有特别的配置，**6.null** 提供了助教们的配置:

- [Anish](https://github.com/anishathalye/dotfiles)
- [Jon](https://github.com/jonhoo/configs)
- [Jose](https://github.com/jjgo/dotfiles)
- [others](https://dotfiles.github.io)有一些很棒的资料可以参考。

## 远程访问

我们可以通过使用 *Secure Shell* 即 **SSH** 来远程连接计算机，通常输入这样的一条指令:

```sh
ssh foo@bar.mid.edu
```

这里我们尝试以用户 **foo** 去登录服务器`bar.mid.edu`，服务器既可以用 **URL** 表示也可以用 **IP** 表示。我们利用`ssh-keygen`生成密钥对，当有多套密钥对时可以用`ssh-agent`去管理。

```sh
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

为了不用每次都输入密码这么麻烦，我们可以将我们的公钥复制到远程计算机，因为 **ssh** 会检查`.ssh/authorized_keys`来决定哪些客户可以直接连接：

```sh
# 直接复制文件
cat .ssh/id_ed25519.pub | ssh foobar@remote 'cat >> ~/.ssh/authorized_keys'
# ssh-copy-id
ssh-copy-id -i .ssh/id_ed25519 foobar@remote
```

当我们想通过`ssh`复制文件时，可以用`scp`命令`scp path/to/local_file remote_host:path/to/remote_file`

## Shell & Frameworks

`TODO`
