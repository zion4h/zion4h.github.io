---
title: MIT 6.NULL 版本控制 (Git)
date: 2023-04-18 14:20:22
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img005.webp
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img005.webp
categories: [编程, Labs, MIT 6.Null]
tags: 
    - shell
    - git
toc: true
---
***Version Control Systems* (VCSs) 版本控制系统** 是专门用来跟踪源码变动的工具。

**VCSs** 会用快照将文件夹和它里面的内容变化记录下来，每一份快照都完整包含了那一时刻文件夹及其子文件和子文件夹的状态，以及快照创建人信息和快照捎带信息。
<!-- more -->
版本控制有什么用？即使只是独立开发，通过 VCS 我们也能轻松了解某段代码的设计和修改目的，从而解放大脑，而在多人合作的场景下则需求更迫切：

- *谁写的这个模块？*
- *A 文件的第 2306 行代码是谁写的？什么时候写的？为什么这么写？*
- *之前某个功能单元是能正常运行的，但现在无法运行了，我该去哪里找问题？*
- *甲部门开发 x 功能，同时让 Z 部门开发 y 功能*

## Git 数据模型

![Git by xkcd](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/git-xkcd.png)

这个漫画讽刺了 Git 的接口设计，太过抽象容易让人困惑，以至于最后大家都像念魔咒一样用它。但如果我们自底向上，先去了解 Git 的底层设计，也就是 **Git 数据模型**，理解起 **Git** 的运作来会更加的轻松。

理解 **Git 数据模型**，意味着理解 `tree`、`blob`、`commit`。在 **Git 数据模型** 中，我们将文件称为 **blob**，目录则称为 **tree**。显然，**tree** 可以同时包含 **tree** 和 **blob**，而 **blob** 并不能。还有一个我们不能忽视的角色是快照，它们被称为 **commit**，每个 **commit** 都包含以下内容：

- `parent`
- `author`
- `message`
- `snapshot` 这里指代最顶层 **tree** 的快照

### Object

如果用伪代码表述的话，可以表示上述三者为：

```javascript
// a file is a bunch of bytes
type blob = array<byte>

// a directory contains named files and directories
type tree = map<string, tree | blob>

// a commit has parents, metadata, and the top-level tree
type commit = struct {
    parents: array<commit>
    author: string
    message: string
    snapshot: tree
}
```

我们用 `object` 去统称 `tree`、`blob`、`commit`，则所有 `object` 都可以按如下方式统一管理：

```javascript
type object = blob | tree | commit

objects = map<string, object>

def store(object):
    id = sha1(object)
    objects[id] = object

def load(id):
    return objects[id]
```

简而言之，**Git** 会对 `object` 通过 **SHA-1 哈希值** （40 个 16 进制数）进行内容寻址。

### Reference

尽管所有的 `commit` 都被哈希值唯一标记，但正常人类是没办法记住 40 个 16 进制数的，因此我们需要借助索引去记忆，它们指向实际的 **commits**，比如我们常用索引 `master` 就指向主分支上的最新的 **commits**。

```javascript
references = map<string, string>

def update_reference(name, id):
    references[name] = id

def read_reference(name):
    return references[name]

def load_reference(name_or_id):
    if name_or_id in references:
        return load(references[name_or_id])
    else:
        return load(name_or_id)
```

在 **Git** 中，索引 `HEAD` 指向当前位置。

### Snapshots

每个 `commit` 都会记录最顶层 `tree`，形如：

```sh
<root> (tree)
|
+- foo (tree)
|  |
|  + bar.txt (blob, contents = "hello world")
|
+- baz.txt (blob, contents = "git is wonderful")
```

而 **commits** 的 DAG （有向无环图）则构成了 **Git** 的历史记录，通俗来讲就是，每个 `commit` 只需要知道自己的父亲是哪些 **commits**，也就是能正确回答我从哪里来这个问题就好（嗯~非哲学范畴）。

假如考虑这样一个场景，我们要对源码进行 1. 修复 Bug；2. 添加新功能，我们可以用新建两个分支同步执行：

```sh
o <-- o <-- o <-- o
            ^
             \
              --- o <-- o
```

我们接下来需要合并 1 和 2，但不可避免的会遇到合并冲突，这需要我们手动选择保留和删除冲突代码。

```sh
o <-- o <-- o <-- o <---- o
            ^            /
             \          v
              --- o <-- o
```

### Repositories

现在，我们终于能够正确定义一个 Git 仓库了！在硬盘上，它是一堆由 `reference` 和 `object` 构成的数据，我们可以从 **Git 数据模型** 角度去理解。我们输入的 **Git 命令** 都在操作 **commit 的 DAG**，本质上是在添加和修改`object`和`referance`。

## 暂存区

**Git** 还有一个暂存区的概念，它跟数据模型正交，属于创建提交的一部分。简单来说，暂存区允许我们告诉 **Git** 下一次快照需要包含哪些修改，这和通常的 VCS 直接保存当前状态有所区别，这能够暂存区的快照更干净、更聪明。

另外，如果你创建了一个新文件，**Git** 并不会跟踪它，此时它的状态是 **Untracked files**，除非你用`git add`命令将其加入 **暂存区**。

一个高质量的提交消息很重要，可参考

- [A Note About Git Commit Messages](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)
- [How to Write a Git Commit Message](https://cbea.ms/git-commit/)

## Git 命令行接口

可以阅读 [Pro Git](https://git-scm.com/book/zh/v2) 以了解更多细节。

### 基础

- `git help <command>` 帮助文档
- `git init` 将当前文件夹初始化为一个 git 仓库，并将数据都放到 **.git 文件夹** 中
- `git status` 查看当前状态
- `git add <filename>` 添加文档到暂存区
- `git commit` 创建一个`commit`
- `git log` 展开历史日志
- `git log --all --graph --decorate` 以 **DAG** 方式展开历史日志
- `git diff <filename>` 展示 **暂存区** 中该文件的具体变动
- `git diff <revision> <filename>` 展示不同快照间该文件的变动
- `git checkout <revision>` 更新 **HEAD 索引** 到指定快照

### 分支与合并

- `git branch` 展示分支
- `git branch <name>` 创建分支
- `git checkout -b <name>` 创建分支并切换 **HEAD** 到该分支
  - 等效于 `git branch <name>; git checkout <name>`
- `git merge <revision>` 将指定分支合并到当前分支
- `git mergetool` 一个工具，用来解决合并冲突的
- `git rebase` 变基 TODO

### 远程访问

- `git remote` 展示远程仓库
- `git remote add <name> <url>` 将一个远程仓库添加到当前 **Git 仓库**
- `git push <remote> <local branch>:<remote branch>` 发送`objects`到远程仓库，并更新远程仓库`reference`
- `git branch --set-upstream-to=<remote>/<remote branch>` 设置本地分支和远程分支的联系关系
- `git fetch` 从远程仓库获取`objects`/`references`
- `git clone` 将远程仓库下载到本地

### 撤销

- `git commit --amend` 修改一条`commit`的内容或者捎带信息
- `git reset HEAD <file>` 取消暂存文件
- `git checkout -- <file>` 丢弃改变

## Git 高级命令

- `git config` 可以参考 [git-config](https://git-scm.com/docs/git-config)
- `git clone --depth=1` 浅克隆，丢掉完整的历史，只保留一个快照
- `git add -p`
- `git rebase -i`
- `git blame` 查看谁最后编辑的
- `git stash`
- `git bisect`
- `.gitignore` 指定哪些 **untracked files** 需要忽略

### Fast-Forward Merge

**Fast-Forward Merge（快进式合并）**其实并不需要我们做额外操作，它是 Git 本身内置的。

![快进式合并](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/20230503163121.png)

### Three-Way Merge

如果在合并时使用 `--no-ff` 参数，Git 就会采用 **Three-Way Merge（三方合并）**。所谓三方合并是同“先 `diff`，再手工决定”的两方合并相区别的，它会根据原始文档内容判断到底该保留谁的，简单来说是舍旧迎新策略。

![三方合并](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/20230503165154.png)

### Squash Merge

**Squash Merge（压缩式合并）**本身和普通 `merge` 没什么两样，但是它能让整个 log 更加干净。我们正常合并会让新 `commit` 拥有两个父节点，但很多时候我们的分支只是做了一些很细小的修改，如果直接 `merge` 会让整个 log 看着非常乱，而压缩式合并能够解决我们这种需求。当然，它是原理本身不难猜到，就是将分支改动在主干上重放，然后需要手动 `commit`。

![压缩式合并](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/20230503171038.png)

### Rebase

## Resources

- [Pro Git](https://git-scm.com/book/en/v2) 很重要的一本书
- [Oh Shit, Git!?!](https://ohshitgit.com/) 教你如何处理常见 Git 错误
- [Git for Computer Scientists](https://eagain.net/articles/git-for-computer-scientists/)
- [Git from the Bottom Up](https://jwiegley.github.io/git-from-the-bottom-up/) 详细描述 Git 自底向上实现
- [How to explain git in simple words?](https://xosh.org/explain-git-in-simple-words/)
- [Learn Git Branching](https://learngitbranching.js.org/?locale=zh_CN) Git 教学游戏

## 实践

### 添加一个 animal.py

```python
import sys

def default():
    print('hello')

def main():
    default()

if __name__ == '__main__':
    main()
```

### 加入到暂存区

`git add animal.py`

### 提交

`git commit`

### 查看

![git branch -vv](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/git-branch-vv.png)

### 添加猫猫功能

![add cat feature](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/20230423163903.png)

### 添加狗狗功能

![add dog feature](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/20230423164506.png)

### 合并分支

![合并猫狗](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/20230423165417.png)

![冲突代码](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/0C0uJtMNir.png)

代码冲突解决后记得将冲突文件添加到暂存区，最后键入 `git merge --continue`，解决！

![over](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/20230423165822.png)

## 参考

[【1】Git 合并那些事——认识几种 Merge 方法](https://morningspace.github.io/tech/git-merge-stories-1/)
