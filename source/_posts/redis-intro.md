---
title: Redis 基础扫盲
date: 2023-03-01 21:29:32
cover: /img/cover/img1000.jpg
categories: 
    - IT
    - IT.数据库
toc: true
tags: 
    - redis
excerpt: Redis是一个NoSQL数据库，开源（遵守BSD协议），使用ANSI C语言编写，Key-Value存储系统，基于内存运行，能够在大部分POSIX系统上运行。
---

# Redis 是什么

首先，我们要对Redis建立一个最基本的印象，K/V，NoSQL，内存数据库，单线程。另外，Redis还是开源的，基于BSD协议（这个协议允许商用），用ANSI C语言编写而成，能够在大部分POSIX系统上运行，当然最常用的就是Linux和OS X系统了。

- **[五种开源协议的比较(BSD，Apache，GPL，LGPL，MIT)](http://www.ha97.com/833.html)**
- ****[ANSI C标准 vs GNU C标准](https://developer.aliyun.com/article/953604)****
- **[Linux 黑话解释：什么是 POSIX？](https://linux.cn/article-14201-1.html)**

# Redis数据类型

由于C语言本身缺乏对一些复杂结构的定义，因此Redis自定义了一些数据类型，自成体系。最基本的数据类型包括String、List、Hash、Set、ZSet，再深入一点，就是Bitmap、Stream、HyperLogLog和Geospatial。

****[Redis data types](https://redis.io/docs/data-types/)****

## String

尽管C语言本身就有定义字符串，但是Redis为了存储二进制文件重新自动定义了一个String，准确的说是用SDS简单动态字符串实现的，String是Redis最常用也是最基本的数据类型，注意区分数据类型和数据结构，SDS是数据结构。

## List

List可以简单当做一个双链表结构，支持双端插入和取出，不过一旦深入，就会发现它的底层并不简单。List基本单元是quicklist，而quicklist的基本单元是ziplist。ziplist自己又是一个特殊双链表，它使用连续块并且没有维护双指针，另一方面它保存前一个entry的长度和当前entry的长度，进而推断出下一个元素和上一个元素起始位置。为什么要这么嵌套循环设计呢？答案是为了节省内存，因为Redis本身是基于内存的，如果基于传统链表实现，那么肯定会产生大量内存碎片，非常浪费。

## Hash

Hash是标准的HashTable架构，即依靠挂链来解决冲突，如果K/V长度较小，Redis会用ziplist取代hashtable对象，其他的感觉没什么好说的。

## Set

Set可以由intset或hashtable构成，原因和Hash类似，当一个集合只包含整数并且数量少于512个的时候，Redis会使用intset。

## ZSet

ZSet也是集合，我们在插入元素的同时要声明该元素的分数，以便它内部比较排序。然后，再谈论下它的底层结构，一般我们用ziplist或者skiplist构成，只有在元素个数少于128并且元素长度小于64字节时使用ziplist。

跳表是在链表基础上发展而来的，为了提升查询效率，我们可以在单链表之上再加一层链表，但只保留一半，每个元素除了指向后一个元素还指向下一层的“自己”，其可行性是建立在元素有序排列的基础之上的。当然，看到这里，我们不难想到，跳表稍微复杂点的地方是插入和删除，会以类似查询的方式对相邻元素更新。

简单来说，记住下面这张图即可

![Untitled](Untitled.png)

## Bitmap

位图对象也是为了节省内存而设计的，如果光看手册可能不知道它该用在什么地方，其实最常见的使用场景就是海量数据统计，比如判断用户是否在线，我们可以根据用户ID来获取位图偏移量，对该比特位上设置用户当前登录状态。

其他应用场景：用户每月签到情况、连续签到用户总数（这里有用到BITOP）

## HyperLogLog

HyperLogLog是用来做基数统计的，使用的时候只需要pfadd和pfcount便能添加和统计一个集合中不重复元素的数目。一想到基数统计，我们很容易想到要用集合，但由于Redis为了节省内存，所以并没有用Set，也没用BitMap，而是用了概率算法，其表现惊人，存储1个亿数据大概只需要12K内存，准确的说在误差只有0.81%的情况下，可以存2^64条数据。

一般来说，电商的UV、PV都会用这个统计流量。

## Geospatial

Geospatial对象是专门用来存储真实地理位置的，所以要输入经纬度。专门设计出这样一个数据对象也是为了快速计算某个位置附近的其他地标，Redis将整个地球转换成了一个二维平面，然后将这个平面不断切割，每个坐标都位于一个唯一的小格子里，经过这个过程后一对经纬度就可以转换为一串二进制数，便于存储和计算。


## Bitmap

位图对象也是为了节省内存而设计的，如果光看手册可能不知道它该用在什么地方，其实最常见的使用场景就是海量数据统计，比如判断用户是否在线，我们可以根据用户ID来获取位图偏移量，对该比特位上设置用户当前登录状态。

其他应用场景：用户每月签到情况、连续签到用户总数（这里有用到BITOP）

## HyperLogLog

HyperLogLog是用来做基数统计的，使用的时候只需要pfadd和pfcount便能添加和统计一个集合中不重复元素的数目。一想到基数统计，我们很容易想到要用集合，但由于Redis为了节省内存，所以并没有用Set，也没用BitMap，而是用了概率算法，其表现惊人，存储1个亿数据大概只需要12K内存，准确的说在误差只有0.81%的情况下，可以存2^64条数据。

一般来说，电商的UV、PV都会用这个统计流量。

## Geospatial

Geospatial对象是专门用来存储真实地理位置的，所以要输入经纬度。专门设计出这样一个数据对象也是为了快速计算某个位置附近的其他地标，Redis将整个地球转换成了一个二维平面，然后将这个平面不断切割，每个坐标都位于一个唯一的小格子里，经过这个过程后一对经纬度就可以转换为一串二进制数，便于存储和计算。

## Stream

# Redis持久化

Redis作为一个内存数据库，一旦宕机，里面的数据就会全部丢失掉，而之前对Redis的查询请求就会直接转移到MySQL中。而且，要让Redis恢复，又要从MySQL中慢慢地读取，这样效率非常低。上述的两个问题，就是Redis持久化的意义所在。

Redis有两种持久化方式，一个是RDB，适合冷备，一个是AOF。

- [冷备、热备、双活、两地三中心](https://zhuanlan.zhihu.com/p/34791057)

## RDB

RDB就是Redis DataBase，为了让Redis能够边复制边写入，注意Redis是单线程，借助了COW，即写时复制Copy On Write技术。另外，由于Redis是由C语言写的，需要用到fork方法。当主线程接收到写入操作时，会将该数据块复制到一旁以便备份线程访问。

## AOF

AOF是Append-only file，还是没明白什么意思，实际上AOF的持久化原理是将Redis的动作记录下来，而当宕机恢复的时候，直接“重放”AOF记录下的动作就可以快速还原。

在Redis4.0后，组合了RDB和AOF，组合方式也很容易想到，用AOF持久化上一次持久RDB到现在为止的动作。

# 数据过期清理策略

Redis的数据过期清理策略是no-eviction，但如果键值对们将内存撑爆了会怎样，Redis会直接罢工的，这样的话即使请求可以一直进行但工作其实已经暂停了。

常见的过期清理策略其实不难想象，LFU频率、LRU最近最少未使用、Random随机，如果设置了过期时间还可以使用TTL策略。

如果我们选择了LRU，Redis会维持一个系统时间，并且给每个键都设置一个时间值，这个时间在访问时会刷新。

# 队列

Redis本身定义了list数据对象，也能满足一些简单队列需求。这里单独提队列，是指的阻塞队列和延迟队列。直接取空list，取元素失败后会sleep一段时间再唤醒重新取，而采用blpop则能阻塞获取元素。延迟队列的实现也很简单，用zset存储消息，而把时间戳作分数，这样我们利用zrangebyscore在N秒之前对数据轮询。

# 发布订阅模式

这个模式有两个角色，一个负责发布频道，一个负责订阅频道，后者持续接收消息。

```bash
# A is a SUB
redis 127.0.0.1:6379> SUBSCRIBE runoobChat

Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "runoobChat"
3) (integer) 1

# B is a PUB
redis 127.0.0.1:6379> PUBLISH runoobChat "Redis PUBLISH test"

(integer) 1

redis 127.0.0.1:6379> PUBLISH runoobChat "Learn redis by runoob.com"

(integer) 1

# A will show message below:
1) "message"
2) "runoobChat"
3) "Redis PUBLISH test"

1) "message"
2) "runoobChat"
3) "Learn redis by runoob.com"
```

# 本文参考

[[1] Redis Documentation](https://redis.io/docs/)

**[[2] Redis 面霸篇：从高频问题透视核心原理](https://mp.weixin.qq.com/s/wrrXz4GoILd5hsbrYACTmA)**
