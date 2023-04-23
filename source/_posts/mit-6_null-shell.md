---
title: MIT 6.Null 如何玩转Shell
date: 2022-12-12 13:22:58
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img000.jpeg
categories: [编程, 编程.Labs, MIT 6.Null]
tag: [shell]
toc: true
---

[6.Null](https://missing.csail.mit.edu)是 **MIT** 专为介绍常用计算机工具所开设的一门课程。我在过去的工作和学习过程中可能已经接触过 **git**、**vim** 等工具，也会使用一些简单的命令行指令比如`cd`、`cp`、`mv`等，但远远谈不上熟练。
<!--more-->
我仍然记得当我第一次怀着热情学习 **Linux** 时，首先上知乎，在大神们的指路下捧起了 **鸟叔的Linux私房菜** 这本书。结果我翻了不到两三页就开始犯困，学习热情消退地很快，整个人就像一个正通过新华字典学语文的洋人。更难以接受的是，网友们评价这本书学起来比较轻松，真的很难想象上古程序员们是度过了怎样枯燥的学习生涯。总而言之，这门课的定位就是一名合格的计算机领路人，我们不用再对着手册慢慢啃，或者干脆永远像新手一样用时谷歌，对着博客里的命令行代码一通照搬生抄，对于 **shell指令** 的概念也仅仅停留在这一长串命令是用来干这个或那个的阶段。

## 初识

课程使用 **Bourne Again SHell** 或者说 **bash**，但我更喜欢在 **Mac** 上用 **Zsh**（一种拓展bash），我使用的终端程序是 **iTerm2**。如果有美化需求，可以结合[调色卡](https://iterm2colorschemes.com)和[zsh主题](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)自定义。当然如果使用 **Windows 11**，我发现 **PowerShell(v7.4.0)** 的优化和配置也挺不错的。

```shell
~$ date
2022年12月12日 星期一 14时48分25秒 CST
~$ echo hello\ world
hello world
~$ echo "hello world"
hello world
```

`～`是`/Users/your-name`的缩写，`date`和`echo`都是程序，我们可以通过`\`输入转义字符或者直接用字符串，这些小程序就是我们常说的命令。当我们输入一个命令时，**Shell** 会到 **$PATH** 中定义的目录里查找与之匹配的程序并执行。

```shell
~$ echo $PATH
...:...:...:dir-x:dir-y:...:...
~$ which echo
echo: shell built-in command
~$ which date
/bin/date
~$ /bin/date
2022年12月12日 星期一 15时14分09秒 CST
```

## 路径

**Linux** 和 **macOS** 用`/`分隔路径并以`/`为根路径，**Windows** 上用`\`分隔且每个磁盘都有一个根，说实话后者的反斜杠真的反人类，后文默认采用 **Linux** 文件系统。任何以`/`开头的都是 **绝对路径**，其他都是 **相对路径**，相对是指相对于当前工作目录。我们用`pwd`（print work directory）命令可以打印当前目录的绝对路径，而用`cd`（change directory）可以进入其他目录，另外`..`表示父目录。

```shell
~$ pwd
/Users/huangzining
~$ cd /home
/home$ pwd
/home
/home$ cd ..
/$
```

当我们运行一个程序时，默认在当前目录中运行，比如`ls`指令默认打印当前目录的内容也可以打印你给出的指定目录。命令运行接收带-开头的标志来控制修改其行为，比如 **-h** 或者 **--help** 可以打印程序的帮助文本，阅读帮助文本我们可以知道`ls -l`表示打印长文本信息。

```shell
~$ ls -l go
total 0
drwxr-xr-x  10 huangzining  staff  320  8 10 22:08 bin
drwxr-xr-x   5 huangzining  staff  160  9 19 22:14 pkg
drwxr-xr-x   3 huangzining  staff   96  8  9 22:26 src
```

**d** 告诉我们这是一个目录文件，后面跟着三组形如 **rwx** 的字符，按照文件持有人、用户组和其他人拥有哪些权限。**rwx** 表示读写执行，短横线表示无对应权限，一般而言普通用户只能读和执行文件。

此外，我们常用的一些命令还包括`mv`文件移动（或重命名）、`cp`文件复制和`mkdir`创建目录。想要了解一个命令程序的完整手册可以使用`man`命令查看，按 **q** 退出。

## 输入输出流

当我们输入`echo`命令打印 ***hello word*** 时，键盘就是输入，屏幕就是输出，但 **shell** 允许我们重定向输入输出流。最简单的重定向是 `< file` 和 `> file`，分别对应输入流和输出流的重定向，还可以组合使用。

```shell
~/tmp$ echo hello > hello.txt
~/tmp$ cat hello.txt
hello
~/tmp$ cat < hello.txt
hello
~/tmp$ cat < hello.txt > hello2.txt
~/tmp$ cat hello2.txt
hello
~/tmp$ cat < hello.txt >> hello2.txt
~/tmp$ cat hello2.txt
hello
hello 
```

上面的 `>>` 表示追加，而 `>` 只能覆盖文件。当然，重定向输入输出流的核心是管道的使用，即利用 `｜` 将两个程序的输入输出连接起来。

```shell
~$ ls -l go | tail -n1
drwxr-xr-x   3 huangzining  staff   96  8  9 22:26 src
```

## root

在大多数类 **Unix** 系统上，都有一个超级用户 **root**，它可以创建、读取、更新和删除系统中的任何文件。一般我们通过`sudo`命令执行一些只能 **root** 才能执行的事情，或者切换成 **root** 用户。

```shell
~$ sudo xxx
~$ su
Password:
sh-3.2#
sh-3.2# exit
exit
~$
```

当出现`#`时，说明已进入 **root模式**。

## 练习

1.在**tmp**目录下创建一个**missing**目录:

```shell
~/tmp$ mkdir missing
~/tmp$ ls
missing
```

2.利用`man`熟悉`touch`命令:

```shell
~/tmp$ man touch
```

3.使用`touch`在 **missing** 内部创建一个名为 **semester** 的文件:

```shell
~/tmp$ touch missing/semester
~/tmp$ ls missing
semester
```

4.将一段内容写到 **semester** 文件中，每次写入一行：

```shell
~/tmp$ echo '#!/bin/bash' > missing/semester
~/tmp$ echo 'curl --head --silent https://missing.csail.mit.edu' > missing/semester
~/tmp$ cat missing/semester
curl --head --silent https://missing.csail.mit.edu
```

注意，`!`比较特殊即使用双引号包裹也会被执行，因此在作为纯文本时要用单引号。

5.直接执行文件:

```shell
~/tmp$ ./missing/semester
zsh: permission denied: ./missing/semester
~/tmp$ ll missing/semester
-rw-r--r--  1 huangzining  staff    51B 12 12 16:27 missing/semester
```

没有x权限，无法执行。

6.利用sh执行文件:

```shell
~/tmp$ sh missing/semester
HTTP/2 200
server: GitHub.com
content-type: text/html; charset=utf-8
last-modified: Mon, 05 Dec 2022 15:59:23 GMT
access-control-allow-origin: *
etag: "638e155b-1f37"
expires: Sun, 11 Dec 2022 08:21:47 GMT
cache-control: max-age=600
x-proxy-cache: HIT
x-github-request-id: FEC6:5295:A4B3D:AE839:6395911E
accept-ranges: bytes
date: Mon, 12 Dec 2022 08:30:02 GMT
via: 1.1 varnish
age: 0
x-served-by: cache-nrt-rjtf7700052-NRT
x-cache: HIT
x-cache-hits: 1
x-timer: S1670833802.399771,VS0,VE225
vary: Accept-Encoding
x-fastly-request-id: facf3a37ed8fe85eaa8ea754c22f5687009da86b
content-length: 7991
```

7.学习chmod

8.修改文件权限:

```shell
~/tmp$ chmod 755 missing/semester
~/tmp$ ll missing/semester
-rwxr-xr-x  1 huangzining  staff    51B 12 12 16:27 missing/semester
~/tmp$ ./missing/semester
HTTP/2 200
server: GitHub.com
content-type: text/html; charset=utf-8
last-modified: Mon, 05 Dec 2022 15:59:23 GMT
access-control-allow-origin: *
etag: "638e155b-1f37"
expires: Mon, 12 Dec 2022 06:15:51 GMT
cache-control: max-age=600
x-proxy-cache: MISS
x-github-request-id: FA3C:15BE:D7417:13394F:6396C4BE
accept-ranges: bytes
date: Mon, 12 Dec 2022 08:37:00 GMT
via: 1.1 varnish
age: 0
x-served-by: cache-tyo11980-TYO
x-cache: HIT
x-cache-hits: 1
x-timer: S1670834221.770088,VS0,VE165
vary: Accept-Encoding
x-fastly-request-id: d62b90ed680dd96c4821069bb3648d618c7a597b
content-length: 7991
```

通过查阅[shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))，我们知道#!实际上指代了#!interpreter [optional-arg]，常见的如：

- #!/bin/sh
- \#!/bin/bash
- \#!/usr/bin/pwsh
- \#!/usr/bin/env python3
- \#!/bin/false

9.利用|和>将由semester生成的last modified字段内容输出到home/last-modified.txt中:

```shell
~/tmp$ ./missing/semester|grep last-modified|cut -d ':' -f 2 > ~/last-modified.txt
~/tmp$ cat ~/last-modified.txt
 Mon, 05 Dec 2022 15
```
