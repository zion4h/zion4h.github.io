---
title: 如何创建一个随意折腾的 Linux 沙盒环境？
date: 2024-07-02 21:09:21
cover: https://i.imgur.com/wm37wRH.png
thumbnail: https://i.imgur.com/wm37wRH.png
categories: ['编程', '扫盲']
tags:
    - intro
toc: true
excerpt: 在本篇博客中，我将介绍如何使用 Vagrant 搭建本地开发环境，并准备一个 C++ 编译环境。Vagrant 是由 HashiCorp 公司开发的工具，能够简化虚拟环境的管理。
---

## 开始

### 介绍

在本篇博客中，我将介绍如何使用 Vagrant 搭建本地开发环境，并准备一个 C++ 编译环境。Vagrant 是由 HashiCorp 公司开发的工具，能够简化虚拟环境的管理。

### 一、云主机与虚拟机的比较

#### 云主机

- 公网 IP

#### 虚拟机

- VirtualBox、VMware
- Docker
- Vagrant：本地便宜且易于控制，启停和连接方便快捷

#### 临时 Web 模拟

- 随用随扔

### 二、Vagrant 安装与配置

#### 前置条件：已安装 VirtualBox

1. **安装 Vagrant**
   - 进入 [Vagrant 下载页面](https://developer.hashicorp.com/vagrant/install)
   - 根据系统架构选择下载安装包：32 位选择 i686，64 位选择 AMD64（x86_64）
   - 验证架构：

     ```powershell
     PS E:\VagrantBoxes> wmic os get osarchitecture
     OSArchitecture
     64-bit
     ```

   - 安装后验证是否成功：

     ```shell
     vagrant --version
     ```

     ![验证安装成功与否](https://i.imgur.com/wm37wRH.png)

2. **环境准备**
   - 添加基础环境，可到 [HashiCorp's Vagrant Cloud box catalog](https://app.vagrantup.com/boxes/search) 查看。

     ![添加基础环境](https://i.imgur.com/kYntQXC.png)

   - 初始化虚拟环境（如 `vb_cpp_memdiy`），会生成一个 `Vagrantfile`：

     ```shell
     vagrant init vb_cpp_memdiy
     ```

     ![初始化虚拟环境](https://i.imgur.com/KTk4b5U.png)

   - 在 `Vagrantfile` 中配置网络和 SSH。

     ![配置网络和 SSH](https://i.imgur.com/uJ3jwkg.png)

3. **启动环境并连接**
   - 启动虚拟环境：

     ```shell
     vagrant up
     ```

     ![启动虚拟环境](https://i.imgur.com/mMCcKsx.png)

   - 通过 SSH 连接访问：

     ```shell
     vagrant ssh
     ```

     ![SSH 连接访问](https://i.imgur.com/PLCmcrs.png)

   - 退出环境：

     ```shell
     # 按 Ctrl+D 或输入
     logout
     ```

### 三、准备 C++ 编译环境

1. **添加 PPA 并安装工具**
   - 添加 PPA 以获取更新版本的 CMake:

     ```shell
     sudo add-apt-repository ppa:kitware/ppa
     sudo apt-get update
     ```

   - 安装 CMake、GDB 和 G++：

     ```shell
     sudo apt-get install cmake gdb g++
     ```

     ![Clion 配置 remote host](https://i.imgur.com/aN5j2rW.png)

     ![Clion 配置 SFTP](https://i.imgur.com/aN5j2rW.png)

     ![Clion 配置 文件映射](https://i.imgur.com/X3x6w5C.png)

     ![Clion 配置 映射验证](https://i.imgur.com/TcQEenU.png)

     ![Clion 配置 上传文件到 remote host](https://i.imgur.com/ftDuwWC.png)
2. **解决依赖包问题**
   - 如果遇到依赖包缺失或版本冲突，升级并下载相应的包。

3. **调试设置**
   - 将调试调整为远程，避免调用本地的 MinGW（Windows）。

     ![Clion 配置 Debug](https://i.imgur.com/a0drG6y.png)

最后，通过以上步骤，你可以在本地使用 Vagrant 搭建一个易于控制的开发环境，并准备好 C++ 编译环境。
