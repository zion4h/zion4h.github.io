---
title: 如何选择开源许可证？
date: 2023-03-31 00:24:42
cover: /img/cover/img1002.jpg
categories: 
    - 编程
    - 编程.开源
toc: true
tags: 
    - osi
excerpt: 在open source initiative(OSI)中，有介绍各种五花八门的开源协议，采用不同的协议代表赠予了不同程度的权力。
---

# OSI

在***open** source initiative*([OSI](https://opensource.org/licenses/))中，有介绍各种五花八门的开源协议，采用不同的协议代表赠予了不同程度的权力。

## GPL

1970s，[理查德·斯托曼](https://zh.wikipedia.org/wiki/%E7%90%86%E6%9F%A5%E5%BE%B7%C2%B7%E6%96%AF%E6%89%98%E6%9B%BC)发起了自由软件运动，并撰写了**GNU通用公共许可协议**([GPL](https://zh.wikipedia.org/wiki/GNU%E9%80%9A%E7%94%A8%E5%85%AC%E5%85%B1%E8%AE%B8%E5%8F%AF%E8%AF%81))，它是一个[Copyleft协议](https://zh.wikipedia.org/wiki/Copyleft)。**Copyleft协议**又叫传染性开源协议，只要项目中用到了带Copyleft协议的部分，那么整个项目都必须带上Copyleft协议，从而变相达到鼓励开源的目的。

## MIT & BSD

与之区别的是**宽松自由软件许可协议条款**([Permissive free software licence](https://zh.wikipedia.org/wiki/%E5%AF%AC%E9%AC%86%E8%87%AA%E7%94%B1%E8%BB%9F%E9%AB%94%E6%8E%88%E6%AC%8A%E6%A2%9D%E6%AC%BE))，其派生项目可以不必保持自由软件身份，常见的[MIT协议](https://zh.wikipedia.org/wiki/MIT%E8%A8%B1%E5%8F%AF%E8%AD%89)和[BSD协议](https://zh.wikipedia.org/wiki/BSD%E8%AE%B8%E5%8F%AF%E8%AF%81)都在此列。顺带一提，这两个协议分别来自麻省理工大学和伯克利大学，而MIT协议是最自由的协议。

```bash
Copyright (C) <year> <copyright holders>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```

## LGPL

由于GPL协议的Copyleft特性，强制项目及其衍生品开源，因此需要再来一个LGPL，允许闭源。

## Apache & MPL

[Apache协议](https://zh.wikipedia.org/wiki/Apache%E8%AE%B8%E5%8F%AF%E8%AF%81)是自由软件协议，而由Mozilla基金会开发的[MPL协议](https://zh.wikipedia.org/wiki/Mozilla%E5%85%AC%E5%85%B1%E8%AE%B8%E5%8F%AF%E8%AF%81)则更适用于专有软件。

如果实在感到迷惑，可以看下面阮一峰制作的一图流：
![license.png](license.png)

# 参考

[【1】****如何选择开源许可证？****](https://www.ruanyifeng.com/blog/2011/05/how_to_choose_free_software_licenses.html)