---
title: 单源最短路径问题
date: 2023-05-21 20:34:23
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img014.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img014.jpg
categories: [编程, 算法]
tags:
    - 算法
    - 最短路径
toc: true
---
当你掏出手机打开高德地图，搜索从“五角场”到“江浦公园”的最短路线，**单源最短路径**就是高效解决这种问题的算法。当然，我其实瞬间就想到了很多其他的点子，比如直接在卫星地图上测量实际距离，又或者将一些路标（比如地铁口、学校、医院等）之间的最短路线（或者最优路线，虽然二者有时不相等，比如“*我喜欢绿化更高的出行线路，它让我心情好*”）提前记录好，然后只需要测量起始点到最近路标的最近路线即可。
<!--more-->
好吧，有点跑题，总而言之这是一篇关于单源最短路径的算法学习文章，我们会接触到 **Bellman-Ford 算法**、**DAG 最短路算法**、**Dijkstra 算法**等。

## 松弛操作

在介绍**松弛操作**之前，我们要先对问题模型进行**初始化**，即如何表示最短路径？我们可以对图中的每个结点，维持两个属性，$v.d$ 表示**距离**，$v.\pi$ 表示**前驱节点**，这样按图索骥即可得到最短路。另外，我们在计算最短路前，应该对所有节点进行初始化，即让图中所有节点距离源点都**无穷远**，再让前驱节点指向**空**。

```python
def initialize_single_source(G, s):
    for v in G:
        v.d = INF
        v.pi = None
    s.d = 0
```

而松弛一条边 `<u, v>` 意味着检测当前 `v` 的最短路是否可以更新，即用 $u.d$ 加上 `u` 到 `v` 的距离同 $v.d$ 进行比较。有点像一个抽象的三角形，但这个三角形有可能两边之和小于第三边，如果成立则更新 `v` 的前驱和距离。

```python
def relax(u, v, W):
    w = W((u, v)) 
    if u.d + w < v.d:
        v.d = u.d + w
        v.pi = u
```

本文介绍的所有方法都将调用 **initialize_single_source**，然后重复对边进行松弛。

## Bellman-Ford 算法

**Bellman-Ford 算法**在我看来内蕴一种暴力美学，在 $|G.v| - 1$ 次循环地对**全部边**进行松弛后，再**检测**下是否有**权重为负值的环路**，结束。对了，这个检测也是对全部边松弛一次。这感觉真的有点秒，就好像有人给你做了 n 次“马杀鸡”后告诉你最短路找到了。

```python
def bellman_ford(G, W, s):
    initialize_single_source(G, s)
    for i in range(1, len(G)):
        for u, v in E:
            relax(u, v, W)
    for u, v in E:
        w = W((u, v))
        if u.d + w < v.d:
            return False
    return True
```

## DAG 最短路算法

**DAG** 最短路算法适用于**有向无环图**，即使有负权重边，但因为没有负权重的环路，最短路必然存在。我们先对 DAG 进行**拓扑排序**，然后按序取结点，对**从该结点出发的毗邻边**做一次松弛操作。

```python
def dag_shortest_paths(G, W, s):
    points = topologically_sort(G)
    initialize_single_source(G, s)
    for u in points:
        for v in adj(G, u):
            relax(u, v, W)
```

**DAG 最短路算法**可以用来找 **PERT 图**的**关键路径**，即 DAG 中最长路径，一种简单方法是**将权重取反**。

## Dijkstra 算法

>Dijkstra 算法总是选择集合 V - S 中“最轻”、“最近”的节点来加入到 S 中，该算法使用的贪心策略。

**Dijkstra 算法**为了保证这个**贪心策略**能稳定计算出最短路，要求所有**边非负**。其基本原理是设定两个结点集 `V` 和 `S`，每次都从 `V` 中挑选一个结点加入到 `S` 中，可能看算法逻辑更清晰。

```python
def dijkstra(G, W, s):
    initialize_single_source(G, s)
    S = []
    Q = G.V
    while Q:
        u = extract_min(Q)
        S.append(u)
        for v in adj(G, u):
            relax(u, v, W)
```

代码中的 `Q` 是一个**优先队列**，而 `extract_min` 方法从 `Q` 中找出距离最短的点，从队列移除并返回该结点。

顺带八卦一下，**dijkstra** 中的 **ij** 其实是 **y** 上添了俩点，但当时打印不出来这个字母于是用 **i 和 j** 做了替换。

## 题目

[2699. 修改图中的边权](https://leetcode.cn/problems/modify-graph-edge-weights/)
