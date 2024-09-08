---
title: MIT 6.Null 大杂烩
date: 2023-05-07 21:54:31
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img011.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img011.jpg
categories: [编程, Labs, MIT 6.Null]
tags:
    - shell
toc: true
---

本篇大杂烩包括键盘重映射、守护进程、FUSE、备份、APIs、命令行模板、窗口管理器、VPNs、Markdown 等。

<!--more-->

## 键盘映射

**键盘映射**有助于我们在不同系统之间无缝切换，这对程序员来说尤其便利，比如我就常常在 `Command` 在 Windows 上对应着 `Ctrl` 还是 `Alt` 上纠结。坦诚说，我到现在也没有对键盘进行重映射，一个直接理由是：我担心看不懂网络上地快捷键指南了。

一些常用的**映射配置**：

* `Capslock -> Ctrl or Esc` 这对洋人来说是个低频键，但对中国人来说其实在 macOS 上蛮常用来切换中英文的
* `PtrSc -> Play/Pause`
* `Ctrl -> Meta`（`Win` 或者 `Command`）

一些**自定义组合键事件**：

* 打开一个新网页或者终端
* 插入一个特别的文本，比如邮箱或者 ID

不同系统会各自有优秀的重映射软件：

* macOS [Karabiner-Elements](https://karabiner-elements.pqrs.org/)，[skhd](https://github.com/koekeishiya/skhd)
* Linux [xmodmap](https://wiki.archlinux.org/title/Xmodmap)，[Autokey](https://github.com/autokey/autokey)
* Windows 内置控制面板，[AutoHotKey](https://www.autohotkey.com/)，[SharpKeys](https://www.randyrants.com/category/sharpkeys/)

## 守护进程

**守护进程 daemons** 本质上就是躲在计算机后台偷偷运行的程序，当然这种说法是非贬义的，毕竟系统进程也是守护进程。通常来说，它们名字都会跟着一个名叫 “d” 的尾巴，比如在 `sshd` 或者 `systemd`。

这里提到的 `systemd` 是 Linux 用来管理守护进程的配置和运行的，你可以输入 `systemctl status` 来列出当前正在运行的守护进程。我们可以利用 `systemctl` 去启用 `enable`，禁用 `disable`，开启 `start`，暂停 `stop`，重启 `restart` 或者检查 `status` 服务状态，这些都是很常用的系统命令，不求记住但至少混个脸熟啊，可别 “天天打照面，次次都脸盲” 了。

如果你想_配置你自己的守护进程_，也可以通过 `systemd` 提供的接口设置。这里是一个自定义 Python 程序的示范：

```properties
# /etc/systemd/system/myapp.service
[Unit]
Description=My Custom App
After=network.target

[Service]
User=foo
Group=foo
WorkingDirectory=/home/foo/projects/mydaemon
ExecStart=/usr/bin/local/python3.7 app.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## FUSE

**FUSE** 是**用户空间文件系统**的英文缩写，简单来说就是以一个用户身份自己设计实现一个文件系统，当然本质上是在调用操作系统内核文件相关的系统调用命令。不过，出于安全方面的考虑，UNIX 并不支持 FUSE。

可能你会好奇，系统自带的文件系统挺好的，我干嘛要重新设计一个呢？下面有一些例子，可以参考：

* [sshfs](https://github.com/libfuse/sshfs) 通过 ssh 在本地打开远程节点的文件
* [rclone](https://rclone.org/commands/rclone_mount/) 安装 Dropbox、GDrive、Amazon S3 或 Google Cloud Storage 等云存储服务并在本地打开数据
* [gocryptfs](https://nuetzlich.net/gocryptfs/) 加密覆盖系统，它的文件和文件夹都是加密过的，然后通过这个系统明文访问，具体细节可以看官网演示
* [kbfs](https://keybase.io/docs/kbfs) 端到端加密的文件系统
* [borgbackup](https://borgbackup.readthedocs.io/en/stable/usage/mount.html)

## 备份

**备份 backups** 这个概念应当和**副本 copy** 区别开，上传到云盘也不算备份，尽管 Google Drive 和 Dropbox 很方便，但当数据被破坏时，它们也会把错误同步。

可以参考 [2019 年 missing 的备份话题章节](https://missing.csail.mit.edu/2019/backups/)

## APIs

很多在线服务都提供 **API** 接口供用户访问，我做过不少 Web 服务，这里不再赘述了。有的 API 需要权限，可以通过 [OAuth](https://www.oauth.com/) 扮演特定角色去访问。

## 命令行模板

如果你对命令一无所知，也可以使用 man 命令直接查看对应程序的使用手册。但大部分程序在设置 flag 时都遵循某些约定俗成的规则：

* `--help` 简略版的使用说明
* `--version` 或者 `-V` 版本号
* `--verbose` 或者 `-v` 打印输出，有的程序支持 `-vvv`，v 越多打印语句越详细
* `--quite` 只打印错误信息
* `-r` 通常来说具有破坏性的命令（比如删除）是不递归的，需要你手动指定**递归标志**

## VPNs

中国有防火墙的存在，所以我们经常通过 VPNs 的访问方式来与外网通信。但实际上在正常的网络访问中，“机场” 和 ISP 本身没什么区别，它们都能够截获你的流量和日志，但是你的通信内容是被 https 保护的。所以在使用 VPNs 时，我们只需要关心网络质量即可。

## Hammerspoon

[Hammerspoon](https://www.hammerspoon.org/) 是 macOS 的桌面自动化框架，它允许你编写挂接到操作系统功能的 Lua 脚本，从而和键盘、鼠标、窗口、显示器、文件系统等进行交互。

## Docker, Vagrant, Cloud

[Vagrant](https://www.vagrantup.com/) 允许你用代码对虚拟机配置进行描述，包括操作系统、服务和包等，然后键入 `vagrant up` 就能将虚拟机实例化。[Docker](https://www.docker.com/) 在概念上类似，但它使用轻量级容器而不是虚拟机。

你也可以在云端租用虚拟机，它有如下好处：

* 永远在线的**公网 IP**
* 机器可自由配置资源
* 以便宜的价格同时持有多台机器（比如一千台）

## GitHub

在 GitHub，你有两种方式参与到开源项目中：

* 提 **issue**，一般是报错和请求新特性，甚至都不需要你去读代码。
* 提 **pull request**，你可以 `fork` 某个项目到本地，新建一个分支，然后修 bug 或者实现一个新特性，最后创建一个 **pull request**，项目管理员会审核你的补丁，如果通过会加入到主干上。通常来说，大型项目都有贡献指南，标记那些对初学者较友好的 **issue**，有的甚至有指导计划来引导初学者。
