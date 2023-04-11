---
title: 6.Null-2 shell脚本编程和常用工具
date: 2022-12-12 22:21:35
cover: /img/cover/anato-finnstark-anato-finnstark-anato-finnstark-web-petit.jpg
categories: [编程, 编程.Labs, MIT 6.Null]
toc: true
---

本文主要叙述了如何以脚本方式去使用**bash**，并介绍了大量常用的**shell工具**。很多工具并非系统自带，因此需要手动下载甚至手动配置，如果是**macOS**的话，使用`brew`一键安装即可。
<!--more-->

## shell脚本

大部分**Shell**都有自己的脚本语言，我们仅描述最基本的**bash**脚本。为什么需要学习**shell**脚本，因为在使用**Shell**时，除了执行命令和用管道将它们连接外，常常会有批量执行命令的需求并希望使用条件或者循环等控制流。

```shell
~/tmp ❯ ls
~/tmp ❯ foo=bar
~/tmp ❯ foo = bar
zsh: command not found: foo
~/tmp ❯ echo "$foo"
bar
~/tmp ❯ echo '$foo'
$foo
```

我们可以在**bash**中直接赋值变量，但注意<u>在**shell**脚本中，空格符会将参数拆分</u>。另外，字符串可以用单引号`'`和双引号`"`定义，但二者不等效，<u>单引号是纯文本，双引号会替换变量值</u>。

现在我们写一个脚本文件***mcd.sh***，`$1`表示这个脚本接收的第一个参数：

```sh
mcd() {
    mkdir -p "$1"
    cd "$1"
}
```

**bash**使用特殊变量来引用参数、错误代码和其他变量。下面是常见变量，或者查看更全面的[特殊变量介绍](https://tldp.org/LDP/abs/html/special-chars.html)。

- `$0` - 脚本名
- `$1` 到 `$9` - 脚本接收的参数
- `$@` - 脚本接收的全部参数
- `$#` - 参数数目
- `$?` - 上一条命令的返回码
- `$$` - 处理当前脚本的进程id
- `!!` - 完整的上一条命令包括参数
- `$_` - 上一条命令的最后一个参数

命令会使用**`STDOUT`**返回输出或者用**`STDERR`**返回错误，并且会返回一个`Return Code`（也就是`$?`）。这个返回码为**0**时表示一切正常，否则表示发生了错误。返回码能够与逻辑与`&&`和逻辑或`||`一起运算，另外同一行命令还能够用分号`;`分隔。

```shell
~/tmp ❯ false || echo "Oops, fail"
Oops, fail
~/tmp ❯ true || echo "Will not be printed"
~/tmp ❯ true && echo "Things went well"
Things went well
~/tmp ❯ false && echo "Will not be printed"
~/tmp ❯ true ; echo "This will always run"
This will always run
~/tmp ❯ false ; echo "This will always run"
This will always run
```

一种常见的场景是希望<u>将命令的输出作为变量获取</u>，我们可以通过命令置换***command substitution***完成。当我们输入`$(CMD)`时，它会先执行`CMD`，并用结果取代原本的`$(CMD)`。比如，当我们想要遍历文件夹时，可以使用`for file in $(ls)`，**shell**会先调用`ls`然后迭代访问这些其返回结果。

另外一个鲜为人知的类似功能是进程替换***process substitution***，常用在希望<u>将值用文件而不是**STDIN**传递</u>时。当我们输入`<(CMD)`时，会先执行`CMD`，然后将结果保存在一个临时文件中，并用这个临时文件名去替换原本的`<(CMD)`。比如，`diff <(ls foo) <(ls bar)`会现实**foo**和**bar**两个文件夹之间的差异。

```sh
#!/bin/bash

echo "Starting program at $(date)" # Date will be substituted

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    # When pattern is not found, grep has exit status 1
    # We redirect STDOUT and STDERR to a null register since we do not care about them
    if [[ $? -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
```

代码中比较了前一条命令的返回码是否为**0**，如果想要执行其他比较，可以参考[test手册](https://www.man7.org/linux/man-pages/man1/test.1.html)和[这里](http://mywiki.wooledge.org/BashFAQ/031)，执行比较的时候注意加双中括号。

**Shell**支持通配符***Wildcards***，<u>可以用`?`和`*`分别匹配一个或任意数量字符</u>，此外**Shell**还支持用<u>大括号`{}`去扩展命令中的公共子字符串</u>。

```shell
~/tmp ❯ mkdir foo bar
~/tmp ❯ mkdir {foo,bar}/{x..z}
~/tmp ❯ tree .
.
├── bar
│   ├── x
│   ├── y
│   └── z
├── foo
│   ├── x
│   ├── y
│   └── z
├── mcd.sh
└── test.sh
```

当脚本变得很复杂时，我们可以借助[shellcheck](https://github.com/koalaman/shellcheck)帮助检测sh或者bash的脚本错误。

除了**bash**脚本，我们还可以写一个**python脚本**，它通过**python**解释器而不是**shell**来执行。

```python
#!/usr/bin/python3
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```

```shell
~/tmp ❯ chmod 755 script.py
~/tmp ❯ ./script.py a b c
c
b
a
```

第一行是[shebang](https://en.wikipedia.org/wiki/Shebang_(Unix))，它告诉内核使用**python**解释器来执行这个脚本。这里不得不强调一下脚本和shell函数的区别，**shell**函数必须用使用和**shell**相同的语言，而脚本可以用任何语言，这就是为什么脚本都需要加上一行**shebang**。



## shell工具

#### 查看命令使用方式

一般用`-h`或`--help`，也可以用`man`直接查看使用手册，但有时候手册过于臃肿，我们可以借助工具[TLDR](https://tldr.sh/)将说明简化。

#### 查看文件

直接使用内置命令`find`即可：

```shell
# Find all directories named src
find . -name src -type d
# Find all python files that have a folder named test in their path
find . -path '*/test/*.py' -type f
# Find all files modified in the last day
find . -mtime -1
# Find all zip files with size in range 500k to 10M
find . -size +500k -size -10M -name '*.tar.gz'
```

我们还可对找到的文件执行操作：

```sh
# Delete all files with .tmp extension
find . -name '*.tmp' -exec rm {} \;
# Find all PNG files and convert them to JPG
find . -name '*.png' -exec convert {} {}.jpg \;
```

尽管`find`很强大，但它的语法用起来也很麻烦，因此我们通常用工具[`fd`](https://github.com/sharkdp/fd)，它更简单更快而且更友好：

```shell
~/tmp ❯ fd "t"
bar/t/
foo/t/
script.py
test.sh
```

#### 查看代码

查看代码最通用的自然是 [grep](https://www.man7.org/linux/man-pages/man1/grep.1.html)，`grep`有很多标志符，比较常用的是`-C`用来获取匹配行的上下文，以及`-v`用来反转匹配（比如返回所有不匹配某个`pattern`的行）。当你想要从多个文件中快速查找时，可以用`-R`递归查询。

针对`grep -R`也有很多优化查询工具，比如[ack](https://github.com/beyondgrep/ack3), [ag](https://github.com/ggreer/the_silver_searcher)和[rg](https://github.com/BurntSushi/ripgrep)，下面的结果高下立判。

```shell
~/tmp ❯ grep -R "s"                                                         17s
./script.py:#!/usr/bin/python3
./script.py:import sys
./script.py:for arg in reversed(sys.argv[1:]):
./test.sh:#!/bin/bash
./test.sh:echo "Starting program at $(date)" # Date will be substituted
./test.sh:echo "Running program $0 with $# arguments with pid $$"
./test.sh:    # When pattern is not found, grep has exit status 1
./test.sh:    # We redirect STDOUT and STDERR to a null register since we do not care about them
./test.sh:        echo "File $file does not have any foobar, adding one"
~/tmp ❯ rg "s"
test.sh
1:#!/bin/bash
3:echo "Starting program at $(date)" # Date will be substituted
5:echo "Running program $0 with $# arguments with pid $$"
9:    # When pattern is not found, grep has exit status 1
10:    # We redirect STDOUT and STDERR to a null register since we do not care about them
12:        echo "File $file does not have any foobar, adding one"

script.py
1:#!/usr/bin/python3
2:import sys
3:for arg in reversed(sys.argv[1:]):
```

#### 查找shell命令

通过`history`我们可以浏览之前输入的命令，除了不断按上箭头，我们还可以通过`history | grep find`找到包含“**find**”子串的命令。

此外，我们也可以用`Ctrl+R`去向后搜索，它其实是和[fzf](https://github.com/junegunn/fzf/wiki/Configuring-shell-key-bindings#ctrl-r)绑定的。当然，由于我用的是zsh，自然使用 ***history-based autosuggestions***插件了。

## 练习

1.读man ls并一个ls命令来列出所有文件

```shell
~/tmp ❯ ls -ahlt --color
total 24
drwxr-xr-x+ 116 huangzining  staff   3.6K 12 13 16:19 ..
drwxr-xr-x    7 huangzining  staff   224B 12 13 16:03 .
drwxr-xr-x   28 huangzining  staff   896B 12 13 15:46 bar
drwxr-xr-x   28 huangzining  staff   896B 12 13 15:46 foo
-rwxr-xr-x    1 huangzining  staff    80B 12 13 15:27 script.py
-rw-r--r--    1 huangzining  staff   484B 12 13 15:11 test.sh
-rw-r--r--    1 huangzining  staff    40B 12 12 22:46 mcd.sh
```

2.编写一个marco保存当前目录路径，然后执行polo进入之前保存的路径

```shell
~/tmp ❯ cat marco.sh
marco() {
    target=~/tmp/test.txt
    if [[ ! (-e $target) ]]; then
	touch $target
    fi
    echo $(pwd) > $target
}
~/tmp ❯ cat polo.sh
polo() {
    cd $(cat ~/tmp/test.txt)
}
```

注意修改执行权限并用source将这两个命令定义给shell。

3.捕获脚本的全部输出打印到文件，记录运行次数。

```sh
 #!/usr/bin/env bash

for (( i=1;; i++ )); do
    n=$(( RANDOM % 100 ))
    if [[ n -eq 42 ]]; then
        echo "Something went wrong"
        >&2 echo "The error was using magic numbers"
        echo "run $i times"
        exit 1
    fi
    echo "Everything went according to plan"
    done
```

