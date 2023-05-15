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
本篇大杂烩包括键盘重映射、守护进程、FUSE、备份、APIs、命令行模板、窗口管理器、VPNs、Markdown等。
<!--more-->
## 键盘映射

键盘映射有助于我们在不同系统之间无缝切换，这对程序员来说尤其便利，比如我就常常在command 在win 上对应着 ctrl还是alt上纠结。坦诚说，我到现在也没有对键盘进行重映射，一个直接理由是：我担心看不懂网络上地快捷键指南了。

一些常用地映射配置：

- Capslock -> Ctrl or Esc 这对洋人来说是个低频键，但对中国人来说其实在mac上蛮常用来切换中英文的
- PtrSc -> Play/Pause
- Ctrl -> Meta（Win 或者 Command）

一些自定义组合键事件：

- 打开一个新网页或者终端
- 插入一个特别的文本，比如邮箱或者ID

不同系统会各自有优秀的重映射软件：

- macOS [Karabiner-Elements](https://karabiner-elements.pqrs.org/)，[skhd](https://github.com/koekeishiya/skhd)
- Linux [xmodmap](https://wiki.archlinux.org/title/Xmodmap)，[Autokey](https://github.com/autokey/autokey)
- Windows 内置控制面板，[AutoHotKey](https://www.autohotkey.com/)，[SharpKeys](https://www.randyrants.com/category/sharpkeys/)

## 守护进程

守护进程daemons本质上就是躲在计算机后台偷偷运行的程序，当然这种说法是非贬义的，毕竟系统进程也是守护进程。通常来说，它们名字都会跟着一个名叫“d”的尾巴，比如在sshd或者systemd。

这里提到的systemd是Linux用来管理守护进程的配置和运行的，你可以输入systemctl status来列出当前正在运行的守护进程。我们可以利用systemctl去启用enable，禁用disable，开启start，暂停stop，重启restart或者检查status服务状态，这些都是很常用的系统命令，不求记住但至少混个脸熟啊，可别“天天打照面，次次都脸盲”了。

如果你想配置你自己的守护进程，也可以通过systemd提供的接口设置。这里是一个自定义Python程序的示范：

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

## 备份

## APIs

## 命令行模板

## 窗口管理器

## VPNs

## Markdown

## Hammerspoon

## Booting + Live USBs

## Docker, Vagrant, VMs, Cloud, OpenStack

## Notebook编程

## GitHub
