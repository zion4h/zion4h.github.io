---
title: 搭建 C 语言编程环境指南
date: 2024-10-01 22:19:39
cover: https://i.imgur.com/HGBwT6T.jpeg
categories: [编程, C]
toc: true
---

C 语言是一个经典且高效的编程语言，广泛应用于操作系统、嵌入式系统等开发中。本文将详细介绍如何在 Linux 环境下搭建一个 C 语言开发环境，包括编译器安装、代码编辑器设置、Git 全局配置、Zsh 快捷键优化等，帮助你快速上手 C 语言编程。
<!--more-->
---

## 1. 安装编译器

C 语言的编译器主要有 GCC 和 Clang，推荐安装 GCC 作为主要编译器。

### 1.1 安装 GCC

可以通过以下命令在 Ubuntu 系统上安装 GCC：

```bash
sudo apt update
sudo apt install build-essential
```

检查安装是否成功：

```bash
gcc --version
```

你应该看到类似以下的输出，说明 GCC 安装成功：

```bash
gcc (Ubuntu 9.3.0-10ubuntu2) 9.3.0
```

### 1.2 安装 Clang（可选）

如果你想使用 Clang 编译器，也可以通过以下命令安装：

```bash
sudo apt install clang
```

之后同样可以通过 `clang --version` 来检查安装情况。

## 2. 设置 VPN 代理（可选）

如果你需要通过 VPN 访问外网，可以在虚拟机中设置 VPN 代理，以下是设置 VPN 的基础步骤：

### 2.1 配置 HTTP 和 HTTPS 代理

在终端中设置 HTTP 和 HTTPS 代理：

```bash
export http_proxy=http://192.168.168.1:1080/
export https_proxy=http://192.168.168.1:1080/
```

使用 `curl` 测试代理是否生效：

```bash
curl www.google.com
```

更多关于 VPN 配置的内容可以参考[这篇教程](https://miaochenlu.github.io/2020/08/21/UbuntuNet/)。

## 3. 配置代码编辑器

为了提高编码效率，你可以使用文本编辑器的插件和自动格式化工具。推荐使用 Clang-Format 来保持代码风格一致。

### 3.1 安装并配置 Clang-Format

使用 Clang-Format 格式化代码，首先确保已安装：

```bash
sudo apt install clang-format
```

然后通过[在线配置工具](https://zed0.co.uk/clang-format-configurator/)生成配置文件，或者使用以下配置作为起点：

```yaml
BasedOnStyle: LLVM
IndentWidth: 4
ColumnLimit: 100
SpaceAfterCStyleCast: true
UseTab: Never
AlignTrailingComments: true
BreakBeforeBraces: Allman
BraceWrapping:
  AfterControlStatement: true
```

将上述内容保存为 `.clang-format` 文件，放置于项目根目录，之后在代码编写完成后运行：

```bash
clang-format -i *.c
```

这将自动格式化所有的 `.c` 文件。

## 4. Git 配置

全局设置 Git 可以帮助你在不同的项目中保持一致的用户信息和提交规范。

### 4.1 设置全局用户名和邮箱

使用以下命令设置 Git 的用户名和邮箱：

```bash
git config --global user.name "huangzining"
git config --global user.email "21210240206@m.fudan.edu.cn"
```

之后你可以运行 `git config --list` 来检查配置是否正确。

## 5. 优化 Zsh 命令行

Zsh 是一个强大的 Shell，可以通过配置快捷键来提升工作效率。

### 5.1 配置 Home 和 End 键

在 Zsh 中可以通过 `bindkey` 命令绑定快捷键。例如，绑定 Home 和 End 键以便快速跳转到行首和行尾：

```bash
bindkey '\e[1~' beginning-of-line
bindkey '\e[4~' end-of-line
```

要使这些配置生效，你可以将它们添加到 `.zshrc` 文件中。

### 5.2 配置常用快捷键

同样的，你可以为删除键（Delete）、PgUp 和 PgDn 等常用按键绑定功能：

```bash
bindkey '\e[3~' delete-char
bindkey '\e[6~' end-of-history
bindkey '\e[5~' insert-last-word
```

更多关于 Zsh 配置的内容可以参考[这篇文章](https://blog.csdn.net/gatieme/article/details/104170950)。

## 6. 常用插件和工具

在日常开发中，除了基础的编译器和编辑器配置，Git 插件也是非常有用的工具。你可以考虑在 Git 提交时自动格式化代码、生成 changelog 或通过插件增强 Git 的交互性。

### 6.1 安装 Git 插件

安装 Git 插件来提升开发效率，如 Git commit 插件。根据你的工作流程，可以搜索相关插件并进行配置。

## 结论

通过以上步骤，你已经完成了基本的 C 语言开发环境搭建。从编译器安装到环境变量配置，再到编辑器和 Shell 的优化设置，这些都能帮助你在 C 语言的开发过程中更加高效。接下来，你可以根据自己的需求进行更多个性化的配置，开始你的 C 语言项目吧！

---

这样的一篇博客可以帮助读者从无到有地搭建 C 语言的开发环境。如果有特定的内容需要补充或修改，请告诉我！
