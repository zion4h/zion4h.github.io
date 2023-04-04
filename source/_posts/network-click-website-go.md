---
title: 输入网址按回车发生了什么？
date: 2023-04-03 19:28:30
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/wallpaperimg1005.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/wallpaperimg1005.jpg
categories: [IT, IT.网络]
tags: [http, dns, 网络]
toc: true
---
当我们在浏览器输入`https://www.youtbe.com`，并按下`Enter`键，便能进入油管慢慢泡视频。在这个过程中，究竟发生了什么？其实简单来说就是一个URL解析、DNS查询、TCP连接、HTTP连接、页面渲染以及断开连接，下面，我将按序详细描述整个过程。
<!-- more -->

## URL解析

一个完整的**URL**结构是`scheme://host.domain:port/path/filename`，其中

- **scheme** 应用层协议，如http、https或ftp等
- **host** 主机名（https的默认主机是www）
- **domain** 域名，如youtube.com
- **port** 端口，http默认80，https默认443
- **path** 服务器上的资源路径
- **filename** 文档或者资源的名字

## DNS查询

**DNS** *Domain Name Server*翻译过来是域名服务器，浏览器本身无法直接通过域名找到服务器，所以需要借助DNS将域名翻译成对应的能访问的IP地址。

浏览器按照以下步骤查询目标IP，找到则停止（我们以`www.youtube.com`为例）：

1. 浏览器缓存
2. 操作系统缓存
3. 路由缓存
4. **ISP**的**DNS**
5. 递归解析器，这里会进行递归查询，先询问根域名服务器，再询问**顶级域名服务器.com**，得到`youtube.com`这个域名的ip地址

## TCP三次握手

当我们获取ip后，要通过**TCP连接**目标服务器进行通信，这个建立连接的过程我们一般形象地称为**三次握手**。

![TCP connection](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/TCP-connection.png)

1. 第一步，客户端想要和服务器建立连接，因此它向服务器发送了一个带**SYN**的段，并告诉服务器它将从哪个序列号开始发送段
2. 第二步，服务器返回一个带**SYN**和**ACK**的段，其中**ACK**表示它响应第一步客户端发送来的段，而**SYN**则表示它可能从哪个序列号开始发送
3. 第三步，客户端确认服务器的响应，开始正常的数据传输

## HTTP报文响应

浏览器会帮我们构造**HTTP请求**，发送给服务器。**HTTP请求**由包含请求行、请求头和请求体三部分构成，请求行由**请求方法**、**URL**字段（如`/`）和**HTTP协议版本**字段3个部分组成；请求头是一组`key: value`；在请求头和请求体之间有一个空行`CRLF`，*请求体可以为空*。

![HTTP request](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/https-request.webp)

## 渲染页面

接收到服务器响应文本后，浏览器开始渲染页面：

![DOM tree](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/1200px-DOM-model.svg.png)

1. 根据HTML解析DOM树
2. 根据CSS解析CSS规则树
3. 结合DOM树和CSS规则树，生成渲染树
4. 根据生成的渲染树计算每个节点的信息
5. 根据每个节点的信息绘制画面给用户

![CSS rule](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/css-rule.svg)

## TCP四次挥手

TCP断开连接过程我们一般简称四次挥手，具体过程如下：

![TCP release](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/tcp-release.png)

1. 第一步，假设客户端决定断开连接（注意，服务端也可以选择断开），则它向服务器发送一个带FIN的段，并进入FIN_WAIT_1状态
2. 第二步，服务端响应第一步中客户端发出的断开连接请求，因此发送一个带ACK的段
3. 第三步，客户端接收到了服务端的响应，进入FIN_WAIT_2状态继续等待
4. 第四步，服务端在处理完断开连接的事务后（比如将没发送完的数据发送完），发送FIN段
5. 第五步，客户端接收到第四步服务端的请求，进入TIME_WAIT状态，并等待一段时间，通常为2MSL，大概是30秒~2分钟。

## 参考

[【1】根域名的知识](https://www.ruanyifeng.com/blog/2018/05/root-domain.html)
[【2】TCP Connection Termination](https://www.geeksforgeeks.org/tcp-connection-termination/#)