---
title: MIT 6.Null Vim
date: 2022-12-16 15:36:26
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img003.jpg
categories: [编程, Labs, MIT 6.Null]
tags: 
    - vim
    - shell
toc: true
---

据 [Stack Overflow](https://survey.stackoverflow.co/2022/#integrated-development-environment) 2022 年统计，[Visual Studio Code](https://code.visualstudio.com) 是全平台最流行的编辑器，而 [Vim](https://www.vim.org) 则是最流行的命令行文本编辑器。**Vim** 源于 1976 年的 **Vi 编辑器**，许多工具都支持 **Vim** 仿真模式（比如 **Visual Studio Code**），如果点进 **Vim** 官网，你会看到一个巨古老的界面，硬核气息可谓是扑面而来。

<!--more-->

**后人竟是我自己**：“写的不够好，vim 哲学没有好好描述，基本命令释义应该用中文，多一点实践场景，那种囊括大部分操作的 demo 展示，以后再修改”

## 入门

初学者可以用 **Vim** 自带的一个教学文档 `vimtutor`，它会让你动手实践来学习。**Vim** 的配置文件是 `.vimrc`。

## Vim 哲学

`Vim` 的终极目标是：**所思即所得**，让编辑速度和思考速度匹配。而为了达成这个目标，Vim 支持 **模态编程** 和 **命令组合**。

## 模态编程

由于我们在编程的时候，很少编辑长文本，反而常常将时间花在搜索、阅读和小规模的编辑上，为此 **Vim** 设计了多种运行模式。在不同的场景下进入不同的运行模式，从而保持心流。

* `Normal`
* `Insert`
* `Replace`
* `Visual`
* `Command-line`

**Vim** 默认和初始状态都是 **Normal**，我们可以通过 `<ESC>` 从任何模式切换回 **Normal**。在普通模式下，我们可以按 `i` 进入 **Insert**，也可以按 `r` 进入 **Replace**，使用 `v` 进入 **Visual-plain**，使用 `v` 进入 **Visual-line**，使用 `<Ctrl-V>` 进入 **Visual-block**，使用 `:` 进入 **Command-line**。当然，我们最常用的还是 **Normal** 和 **Insert** 模式。

## 基本命令

### 移动

* Basic movement: `hjkl` (left, down, up, right)
* Words: `w` (next word), `b` (beginning of word), `e` (end of word)
* Lines: `0` (beginning of line), `^` (first non-blank character), `$` (end of line)
* Screen: `H` (top of screen), `M` (middle of screen), `L` (bottom of screen)
* Scroll: `<Ctrl u>` (page up), `<Ctrl d>` (page down)
* File: `gg` (beginning of file), `G` (end of file)
* Line numbers: `number G` (number line)
* Misc: `% (corresponding item)`
* Find: `f{character}`, `t{character}`, `F{character}`, `T{character}`
* Search: `/{regex}`, `n` or `N` for navigating matches

### 选取

* `v` 普通选取
* `V` 按行选取
* `<Ctrl v>` 按块选取

### 编辑

* `i` 进入 **Insert** 模式
* `o` 或 `O` 在下一行或上一行插入空白行
* `d{motion}` 删除 e.g. `dw` 删除单词，`d$` 删除至行尾，`d0` 删除至行头
* `c{motion}` 修改
* `x` 删除当前字符，和 `dl` 等效
* `s` 修改当前字符，和 `cl` 等效
* `u` 撤销 `<Ctrl r>` 返回撤销
* `y` 复制（**yank** 缩写）
* `p` 粘贴

### 计数

我们可以将名词或动词同数字组成，从而简化重复操作

* `3w` 向前移动三个单词
* `5j` 向下移动五行
* `7dw` 删除七个单词

### 修饰符

我们对于成对的修饰符可以用 `a` 或 `i` 去操作其包含的内容：

* `ci(` 修改 `()` 内的内容
* `ci[` 修改 `[]` 内的内容
* `da'` 删除`''` 内的内容以及`''` 本身

### Command-line 命令

* `:q` quit
* `:w` save
* `:wq` save and quit
* `:e {name of file}` open file for editing
* `:ls` show open buffers
* `:help {topic}` open help

## 高级 Vim

### 查询和替换

**Vim** 本身有用`:s` 提供查询替换功能，但是具体怎么用可以看 [vim fandom](https://vim.fandom.com/wiki/Search_and_replace)，这边提供几个使用例子。

* `%s/foo/bar/g` 用 **foo** 去全局替换 **bar**
* `%s/\[.*\](\(.*\))/\1/g` 将 **markdown** 链接替换成普通 **url**
