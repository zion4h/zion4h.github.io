---
title: 如何选择开源许可证？
date: 2023-03-31 00:24:42
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/wallpaperimg1002.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/wallpaperimg1002.jpg
categories: 
    - [编程, 开源]
toc: true
tags: 
    - 开源
---
**开源促进会 OSI** 是**开源定义 OSD** 的管理者，由 Bruce Perens 和 Eric S. Raymond 创立于 1998 年，我们寻常用的开源协议都需要该组织认可，要求符合 **OSD** 定义。
<!--more-->

- **OSI** [Open Source initiative](https://opensource.org/licenses/)
- **OSD** [Open Source definition](https://en.wikipedia.org/wiki/The_Open_Source_Definition)

## 常用开源协议

### GPL

1970s，[Richard Stallman](https://en.wikipedia.org/wiki/Richard_Stallman) 发起了**自由软件运动**，并撰写了 **GNU 通用公共许可协议** [GPL](https://en.wikipedia.org/wiki/GNU_General_Public_License)。GPL 是一个 [Copyleft 协议](https://zh.wikipedia.org/wiki/Copyleft)，而 Copyleft 协议又叫**传染性开源协议**，只要项目中有一部分带 Copyleft 协议，那么整个项目都必须带上 Copyleft 协议，从而*变相达到鼓励开源的目的*。

### MIT & BSD

与 GPL 相区别的是**宽松自由软件许可协议** [Permissive free software licence](https://en.wikipedia.org/wiki/Permissive_software_license)，其派生项目可不用再保持自由软件身份，常见的 [MIT 协议](https://en.wikipedia.org/wiki/MIT_License) 和 [BSD 协议](https://en.wikipedia.org/wiki/BSD_licenses) 都在此列。

顺带一提，这两个协议分别来自麻省理工大学和伯克利大学，里面的 MIT 协议是最自由的协议。由于 BSD 原始协议要求派生体在打广告时必须声明原始来源，因此我们通常用更简化的 **FreeBSD 协议**，同时 BSD 也是商用首选协议。

如 MIT 协议，有：

```bash
Copyright (C) <year> <copyright holders>

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
```

### LGPL

由于 GPL 的 Copyleft 特性，强制项目及其衍生品开源，因此需要再来一个允许闭源的 LGPL 来调和。

### Apache & MPL

[Apache 协议](https://en.wikipedia.org/wiki/Apache_License) 是自由软件协议，而由 Mozilla 基金会开发的 [MPL 协议](https://en.wikipedia.org/wiki/Mozilla_Public_License) 则更适用于专有软件。

如果实在感到迷惑，可以看下面**阮一峰**制作的一图流：
![Open Source License Choice](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/open-source-license.png)

## 参考

[【 1 】阮一峰：如何选择开源许可证？](https://www.ruanyifeng.com/blog/2011/05/how_to_choose_free_software_licenses.html)

[【 2 】五种开源协议的比较 (BSD，Apache，GPL，LGPL，MIT)](http://www.ha97.com/833.html)
