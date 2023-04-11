---
title: 6.824 Lab1 MapReduce
date: 2022-12-01 12:00:00
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img264.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img264.jpg
categories: [编程, 编程.Labs, MIT 6.824]
toc: true
---
遵循[lab页](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)指导，并在本地部署好项目。我在观摩完***mrsequential***的代码后，再结合[MapReduce论文](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)，理清了基本思路。
<!--more-->

### 准备

首先，我们会有两个角色，`worker`负责处理任务，`coordinator`负责管理和分配任务，一般有一个`coordinator`和多个`worker`（现在先不考虑**单点故障**问题）。

- `SPOF` **单点故障**指系统中一点失效，就会让整个系统无法运作的部件。

单词计数任务的**map阶段**会将输入的多个文本转换成形如“*hello，1*”这样的键值数组，然后在**reduce阶段**会将这些键值数组整合并输出。另外，我们在代码设计过程中应注意：`coordinator`只负责分配和管理，而`worker`负责全部的计算。

### map

针对**lab1**，我们的单个**map任务**会接受一个文本文件，然后将其转换成中间键值对后，由给定的**nReduce**个**reduce任务**去接收计算，lab中已经给定为10。也就是说，我们的一个**map任务**，假设其编号为2，会将一个文本文件转化成10个中间文件，也就是*mr-2-0到mr-2-9*。而每个**reduce任务**都会将属于自己那份文件输入，比如编号为4的**reduce任务**会将中间文件`mr-x-4`的全部文件输入，最终输出`mr-out-4`。我们不需要做更后面的合并，测试脚本会帮我们完善这一步（可以看`test.sh`代码）。

### coordinator

上面是对lab1的任务处理流程进行梳理，接下来我们需要针对`coordinator`按需求进行设计。为了方便管理**reduce和map任务**的执行进度，我们需要定义一个新的结构体`task`，它包含任务类型、任务编号、是否完成，之前有将任务状态分为初始化，处理中和已完成三种状态，但后面看了助教的QA后，用`isDone`简单管理两种状态。之所以这样改进，是因为我们要让`coordinator`负责任务调度，如果将**inProcessing状态**传输回`worker`，那么`worker`也将会参与调度工作，比如等待一段时间，这是不太完善的。

### RPC设计

另一个需要设计的是Rpc，lab1中我们总共涉及**两种通信场景**，即`worker`向`coordinator`请求任务，以及`worker`向`coordinator`通知某个任务已完成。原本我是将整个`task`作为参数在RPC中进行传输的，现在分析整理了一下代码需求，发现我们只需要task类型、task编号、而**map**还需要reduce个数和map输入文件名、而**reduce**还需要map个数。而在任务完成时发送的信息，只包括task的类型和task的编号。

### 同步问题

最后，我们还需要处理同步问题。后面是漫长的修改，我控制并发的手段很单一，就是将`coordinator`访问共享变量的代码都用mutex包起来，最后用test-mr脚本测试的时候，总是在map并行测试这里失败，后面发现我的`worker`在处理**map和reduce**时用了`goroutine`，当时为了效率加的。

再引入`cond`，之前我的代码逻辑当检查完**map任务**发现有**map任务**没完成时会等待一段时间再检查，直到**map**完成后进入**reduce**，同理**reduce**完成后全部结束。引入`cond`可以让`coordinator`在处理完成任务时，再激活检查一下，而不再只是呆板的定时检查，这样更符合逻辑，当然定时检查同样不能少，这样当一个任务迟迟没有完成时`coordinator`也能及时发现。
