---
title: 如何创建一个随意折腾的 Linux 沙盒环境？
date: 2024-07-02 21:09:21
cover: https://i.imgur.com/WVccm9R.png
thumbnail: https://i.imgur.com/WVccm9R.png
categories: ['编程', '扫盲']
tags:
    - intro
toc: true
excerpt: 项目式学习（Project Based Learning）能够快速检验和提高一个人的编程水平，而学习过程中又不可避免地会编写运行一些 Demo。然而，如果直接使用 Windows 或者 MacOS 多少会面临环境部署问题，因此拥有一个能随意折腾地 Linux 沙盒对我来说开始变得尤为迫切。因此，在本篇博客中，我将介绍如何在 Windows 上轻松创建一个 Linux 沙盒。
---

### 介绍

项目式学习（[Project Based Learning](https://github.com/practical-tutorials/project-based-learning?tab=readme-ov-file#project-based-learning)）能够快速检验和提高一个人的编程水平，而学习过程中又不可避免地会编写运行一些 Demo。然而，如果直接使用 Windows 或者 MacOS 多少会面临环境部署问题，因此拥有一个能随意折腾地 Linux 沙盒对我来说开始变得尤为迫切。因此，在本篇博客中，我将介绍如何在 Windows 上轻松创建一个 Linux 沙盒。

### 部署 Linux 的方法

最直接的想法是在阿里云等平台直接租一台**云主机**，选择完套餐后再启动就是一个 Linux 系统，甚至配有公网 IP。但坏处也很明显，贵！

各种**在线沙盒**是退而求其次的想法，诸如 [COMPILER EXPLORER](https://godbolt.org/) 等，想用 C++ 的任一版本编译器都可，还不用部署，开盒即用还随用随扔。

最后就是各种本地跑的**虚拟机**了，厚重的如 VirtualBox、VMware，小巧的如 Docker，还有就是今天的主角 Vagrant 了！Vagrant 是由 HashiCorp 公司开发的工具，不仅简单易上手，启停和连接也非常迅速。

### Vagrant 安装与配置

使用 Vagrant 搭建本地开发环境，并准备一个 C++ 编译环境。

#### 前置条件

* 已安装 VirtualBox

#### 安装 Vagrant

1. 进入 [Vagrant 下载页面](https://developer.hashicorp.com/vagrant/install)

2. 根据系统架构选择下载安装包：32 位选择 i686，64 位选择 AMD64（x86\_64）

3. 验证架构：

   ```powershell
   PS E:\VagrantBoxes> wmic os get osarchitecture
   OSArchitecture
   64-bit
   ```

4. 安装后验证是否成功：

   ```shell
   vagrant --version
   ```

   ![验证安装成功与否](https://i.imgur.com/wm37wRH.png)

#### 环境准备

1. 添加基础环境，可到 [HashiCorp's Vagrant Cloud box catalog](https://app.vagrantup.com/boxes/search) 查看。

   ![添加基础环境](https://i.imgur.com/kYntQXC.png)

2. 初始化虚拟环境（如 `vb_cpp_memdiy`），会生成一个 `Vagrantfile`：

   ```shell
   vagrant init vb_cpp_memdiy
   ```

   ![初始化虚拟环境](https://i.imgur.com/KTk4b5U.png)

3. 在 `Vagrantfile` 中配置网络和 SSH。

   ![配置网络和 SSH](https://i.imgur.com/uJ3jwkg.png)

#### 启动环境并连接

1. 启动虚拟环境：

   ```shell
   vagrant up
   ```

   ![启动虚拟环境](https://i.imgur.com/mMCcKsx.png)

2. 通过 SSH 连接访问：

   ```shell
   vagrant ssh
   ```

   ![SSH 连接访问](https://i.imgur.com/PLCmcrs.png)

3. 退出环境：

   ```shell
   # 按 Ctrl+D 或输入
   logout
   ```

### 准备 C++ 编译环境

#### 添加 PPA 并安装工具

1. 添加 PPA 以获取更新版本的 CMake:

   ```shell
   sudo add-apt-repository ppa:kitware/ppa
   sudo apt-get update
   ```

2. 安装 CMake、GDB 和 G++：

   ```shell
   sudo apt-get install cmake gdb g++
   ```

   ![Clion 配置 remote host](https://i.imgur.com/aN5j2rW.png)

   ![Clion 配置 SFTP](https://i.imgur.com/aN5j2rW.png)

   ![Clion 配置 文件映射](https://i.imgur.com/X3x6w5C.png)

   ![Clion 配置 映射验证](https://i.imgur.com/TcQEenU.png)

   ![Clion 配置 上传文件到 remote host](https://i.imgur.com/ftDuwWC.png)

#### 解决依赖包问题

* 如果遇到依赖包缺失或版本冲突，升级并下载相应的包。

#### 调试设置

* 将调试调整为远程，避免调用本地的 MinGW（Windows）。

  ![Clion 配置 Debug](https://i.imgur.com/a0drG6y.png)

### 结论

通过以上步骤，你可以在本地使用 Vagrant 搭建一个易于控制的开发环境，并准备好 C++ 编译环境。
