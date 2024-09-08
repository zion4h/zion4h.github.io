---
title: 并查集
date: 2023-06-06 21:26:54
cover: https://i.imgur.com/Jhp4n7Z.jpeg
thumbnail: https://i.imgur.com/Jhp4n7Z.jpeg
categories: [编程, 算法]
tags:
    - 算法
    - 并查集
toc: true
---

算法常见题之**并查集**或者不相交集合数据结构 **DSU**，当有一组集合 n 个元素，假设我们经常需要：（1）判定某个元素属于哪个集合，（2）将两个集合合并成一个集合。那么，我们只需要维护一个**并查集**即可。

<!--more-->

为什么不直接在每个元素对象中添加一个描述从属集合的属性呢？因为合并时，时间复杂度是 `O(n)`，而并查集只需要很少时间。

## 分析

并查集包含三种操作，创建集合，**合并两个集合**，**查找元素的从属集合**。我们一般会从一个集合中挑出一个**代表元素**来代表整个集合，如果把集合看作一棵树，那么查找操作可以看作是查找根节点，而合并操作则可以看作是将一棵树的根节点指向另一棵树的根节点。

## 模板代码

模版代码（使用了**路径压缩**）：

```python
class Dsu:
    def __init__(self, size):
        self.pa = list(range(size))

    def find(self, x):
        if self.pa[x] != x:
            self.pa[x] = self.find(self.pa[x])
        return self.pa[x]

    def union(self, x, y):
        self.pa[self.find(x)] = self.find(y)
```

## 启发式合并

**启发式合并**是对上面代码的改进，简单来说，如果我们每次都将**深度**或者**点数**更小（二选一）的集合树连接到一棵更大的集合树下，那么之后的查找操作时间将会缩短。

```python
class Dsu:
    def __init__(self, size):
        self.pa = list(range(size))
        self.size = [1] * size

    def find(self, x):
        if self.pa[x] != x:
            self.pa[x] = self.find(self.pa[x])
        return self.pa[x]

    def union(self, x, y):
        x, y = self.find(x), self.find(y)
        if x == y:
            return
        if self.size[x] < self.size[y]:
            x, y = y, x
            self.pa[y] = x
            self.size[x] += self.size[y]
```

一般来说，用压缩路径即可。[Tarjan](https://www.researchgate.net/publication/220430653_Worst-case_Analysis_of_Set_Union_Algorithms) 证明了如果不使用启发式合并、只使用路径压缩的最坏时间复杂度是 $O (m \log n)$，而[姚期智](https://epubs.siam.org/doi/abs/10.1137/0214010?journalCode=smjcat) 证明了不使用启发式合并、只使用路径压缩，在平均情况下，时间复杂度依然是 $O (m\alpha (m,n))$。

[CLRS 的阅读攻略😂](https://www.cnblogs.com/clemente/p/9902220.html) 在学习章节之前可以先看看这个总结，节约时间，搞清楚重点。

## 问题

[547. 省份数量](https://leetcode.cn/problems/number-of-provinces/)

[1319. 连通网络的操作次数](https://leetcode.cn/problems/number-of-operations-to-make-network-connected/)
