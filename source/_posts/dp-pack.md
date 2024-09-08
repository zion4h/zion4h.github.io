---
title: 背包 DP
date: 2022-12-06 21:29:32
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img006.jpg
categories: [编程, 算法]
tags: 
    - leetcode 
    - algorithm
toc: true
---

背包问题探讨了如何用有限背包去装入尽可能多或者尽可能贵的物品，或者说用**有限空间换取最大价值的问题**。当然，问题是多样化的，我们需要将问题中涉及到的资源抽象对应到背包的空间和价值两个维度上。

<!--more-->

## 01 背包

有 **N** 件物品和一个容量为 **V** 的背包，放入第 i 件物品会消耗 **ci**，获得价值 **wi**，而 **01 背包**问题的特点是每种物品有且仅有一件。其状态转移方程为：

```javascript
F[i,v] = max {F[i-1, v], F[i-1, v - c_i] + w_i}
```

根据上面的转移方程可以写出如下代码（网上随意 copy 了一份，看看思路即可），**每当遇到一个新的物品，判断加入它到背包是否能让之前的收纳规划优化**：

```python
n, v = map(int, input().split())
goods = []
for i in range(n):
    goods.append([int(i) for i in input().split()])

# 初始化，先全部赋值为 0，这样至少体积为 0 或者不选任何物品的时候是满足要求  
dp = [[0 for i in range(v+1)] for j in range(n+1)]

for i in range(1, n+1):
    for j in range(1,v+1):
        dp[i][j] = dp[i-1][j]  # 第 i 个物品不选
        if j>=goods[i-1][0]:   # 判断背包容量是不是大于第 i 件物品的体积
            # 在选和不选的情况中选出最大值
            dp[i][j] = max(dp[i][j], dp[i-1][j-goods[i-1][0]]+goods[i-1][1])

print(dp[-1][-1])
```

而将上述代码优化后可以得到：

```python
dp = [0] * (N + 1)

for i in range(1, N+1):
    c, w = goods[i][0], goods[i][1]
    for j in range(V, c-1, -1):
        dp[j] = max(dp[j], dp[j - c]+w)
```

## 完全背包

即将 01 背包问题的条件修改为每种物品有无穷件，其状态转换方程为：
$$
F \[i,v] = max \\{ F \[i-1,v], F \[i,v-c\_i]+w\_i \\}
$$
注意和前一个转换方程对比，不难发现，其区别的关键点在于，\*\* 我们在完全背包考虑加入一件物品时，之前可能已经加入过无数件它的复制品了，而 01 背包时则从没加入过该物品。\*\* 分析后，不难得到代码如下：

```python
dp = [0] * (n + 1)

for i in range(1, N+1):
    c, w = goods[i][0], goods[i][1]
    for j in range(c, V+1):
        dp[j] = max(dp[j], dp[j - c]+w)
```

另外一种解决完全背包问题的思路是，虽然每种物品都是无限的，但在有限容量的限制下，我们最多只能取 $\lfloor V/c\_i \rfloor$ 件，因此可以转换为 01 背包问题。更近一步，可以用二进制优化。

## 多重背包

即将 01 背包问题的条件修改为每种物品有若干件，其状态转换方程为：
$$
F \[i,v] = max \\{ F \[i-1,v], F \[i-1,v-k \times c\_i]+k \times w\_i|0<=k<=M\_i \\}
$$
对于 $M\_i$ 件第 i 种物品，我们考虑用二进制的方法将其转换为 01 背包问题，令这些系数分别为 $1,2,2^2 ...2^{k-1},M\_i−2^k+1$，比如 13 可以被分为 `1,2,4,6` 四件物品。

总的来说，面对背包问题，首先状态转换方程，即思考前 i-1 件物品满足某种情况后，面对第 i 件物品，我们应该如何去优化我们规划。

## 分组背包

同样由 01 背包的背景而来，**N** 个物品被划分成了 **K** 组，同一组的物品最多只能选择一件。此时，我们的思考方式变成了，**每当新的一组物品，我们选本组中最大的一件或者一件都不选**，有状态方程：

$$
F\[i,v] = max\\{F\[k-1,v], F\[k-1,v-c\_i]+w\_i|item\ i \in group\ K\\}
$$

优化后的伪代码如下：

![fake code](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/dp-pack-fake-code.png)

三重循环不可随意更替，内部第二重循环的逆序是因为在第三重循环会更新 $F \[v]$ 的值，而优化时只能在 $i-1$ 的基础上优化。

## 参考资料

【 1 】 崔添翼的 [背包九讲](https://github.com/tianyicui/pack/blob/master/V2.pdf)

【 2 】 三叶 LeetCode 的 [背包 DP 子栏](https://github.com/SharingSource/LogicStack-LeetCode/wiki/背包-DP)
