---
title: 最小生成树
date: 2023-06-10 17:06:03
cover: https://i.imgur.com/EI7uBgh.jpeg
thumbnail: https://i.imgur.com/EI7uBgh.jpeg
categories: [编程, 算法]
tags:
    - 算法
    - MST
toc: true
---
**最小生成树**源于最短电路连接，我们希望连接所有组件的针脚，同时又巴不得连线最短。由于连接了全部点，同时无环，因此又叫最小树。
<!--more-->
最小树的生成算法其实很简单，如下：

```python
def generic_mst(G):
    A = []
    n = len(G)
    while len(A) < n - 1:
        e = findSafeEdge(A, G.E)
        A.append(e)
    return A
```

这里面的 `findSafeEdge` 意为寻找一条**安全边**，**Kruskal** 和 **Prim** 算法都各自定义了这条**安全边**的具体规则。

## Kruskal 算法

简单来说，先将所有边按权重排序，再由小到大遍历所有边，将两端点非同一棵树（借助并查集）的边加入。这样一来，每次加入的安全边永远是**权重最小的连接两个不同分量的边**。

```python
def mst_kruskal(G):
    A = []
    n = len(G)
    dsu = Dsu(n)
    for v in G.V:
        dsu.pa[v] = v
    sort(G.E)
    for e in G.E:
        u, v = e
        if dsu.find(u) != dsu.find(v):
            A.append(e)
            dsu.union(u, v)
    return A
```

**Kruskal** 算法时间为 `O(ElgV)`。

## Prim 算法

**Prim** 和 **Dijkstra** 算法相似，保持**集合 A** 的边总能构成一棵树，每次从连接集合 A 和集合 A 之外的点的所有边中选取一条边。

```python
def mst_prim(G, r):
    for u in G.V:
        d[u] = inf
        pi[u] = None
    d[r] = 0
    Q = heapq.heapify(G.V)
    while Q:
        u = heapq.heappop(Q)
        for v in G.adj[u]:
            if G.W(u, v) < d[v]:
                pi[v] = u
                d[v] = G.W(u, v)
```

**Prim** 算法的时间取决于最小优先队列 `Q` 的实现，如果使用**二叉最小优先队列**，则时间复杂读为 `O(ElgV)`，如果改用**斐波那契堆**来实现优先队列，那么算法时间 `O(E+VlgV)`。

一个有趣的点在于，最**小生成树算法**最早由 **Boruvka** 发明，**Kruskal** 是 1956 年，而 **Prim** 是 1930 年。
