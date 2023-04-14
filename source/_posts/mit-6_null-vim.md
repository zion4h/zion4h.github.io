---
title: MIT 6.Null Vim
date: 2022-12-16 15:36:26
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img003.jpg
categories: [编程, 编程.Labs, MIT 6.Null]
tag: [vim, shell]
toc: true
---
据[Stack Overflow](https://survey.stackoverflow.co/2022/#integrated-development-environment)2022年统计，[Visual Studio Code](https://code.visualstudio.com)是全平台最流行的编辑器，而[Vim](https://www.vim.org)则是最流行的命令行文本编辑器。**Vim** 源于1976年的 **Vi编辑器**，许多工具都支持 **Vim** 仿真模式（比如 **Visual Studio Code**），如果点进 **Vim** 官网，你会看到一个巨古老的界面，硬核气息可谓是扑面而来。
<!--more-->

## 入门

`vimtutor` Vim自带了一个教学文档，键入即可进入

## Vim哲学

**Vim** 有一套特殊且有趣的设计哲学，由于它是面向程序员设计的软件，而在编程时大部分时间都在读和修改而不是创作（看透了大伙本质上都是灵活CV工程师），因此 **Vim** 将`insert`和`manipulate`两种模式分离。**Vim** 被设计成可编程的，即它的操作命令可以组合使用产生各种奇效。当我们在使用 **Vim**时候应该避免使用鼠标，甚至是方向键，虽然我不是很理解因为我现在还不太能熟练使用 **Vim**。`Vim`的终极目标是：**所思即所得**，让编辑速度和思考速度匹配。

## 模态编程

**Vim** 认为程序员很少编辑长文本，反而常常将时间花在搜索、阅读和小规模的编辑上，为此 **Vim** 设计了多种运行模式：

- `Normal`
- `Insert`
- `Replace`
- `Visual`(plain, line, block)
- `Command-line`

Vim默认和初始状态都是`Normal`，我们可以通过`<ESC>`从任何模式切换回Normal。在普通模式下，我们可以按 **i** 进入`Insert`，也可以按 **r** 进入`Replace`，使用 **v** 进入`Visual-plain`，使用**V** 进入`Visual-line`，使用`<^v>`（`Ctrl-V`也写作`^V`）进入`Visual-block`，当然还有最通用的使用 **:** 进入`Command-line`。

## 基本编程

`Command-line`基本命令如下：

- `:q` quit
- `:w` save
- `:wq` save and quit
- `:e` {name or file} open file
- `:ls` show open buffers
- `:help` {topic} open help

- Basic movement: `hjkl` (left, down, up, right)
- Words: `w` (next word), b (beginning of word), e (end of word)
- Lines: `0` (beginning of line), `^` (first non-blank character), `$` (end of line)
- Screen: `H` (top of screen), `M` (middle of screen), `L` (bottom of screen)
- Scroll: `<^u>` (up), `<^d>` (down)
- File: `gg` (beginning of file), `G` (end of file)
- Line numbers: `:{number}<C-R>` or `number G` (number line)
- Misc: `% (corresponding item)`
- Find: `f{character}`, `t{character}`, `F{character}`, `T{character}`
- Search: `/{regex}`, n / N for navigating matches

我的常用组合键：

- `u` 撤销` <C-R> `取消撤销
- `dd` 删除行
- `o` 向下插入行 `O` 向上插入行

`/` 查询：在`normal`下按/即可查询，按 `n` 查找下一个，按 `N` 查找上一个，且支持正则表达式，如果加入`\c`表示大小写不敏感

`:s` 查找并替换，常用方式`:%s/foo/bar/g`可以查看[这里](https://vim.fandom.com/wiki/Search_and_replace#Additional_examples)看具体使用方式。

`:sp` 或者 `:vsp` 切割窗口

## Vim接口

### 移动

## Vim高级技巧
