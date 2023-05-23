---
title: 算法（持续更新）
date: 2013/7/13 20:46:25
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img004.jpg
toc: true
categories: [编程, 算法]
tags: 
    - leetcode
    - algorithm
---
> The palest ink is better than the best memory.
>
> 好记性不如烂笔头。
<!--more-->

- [Markdown 中输入数学公式及 LaTex 常用数学符号整理](http://liyangbit.com/math/jupyter-latex/)

## Trie 字典树

**字典树** 又叫 **前缀树**，洋文叫 **Trie**。**Trie** 本质上是一个多路查询树，它也可以被看作一个树形态的确定有限状态自动机 **DFA**，

- [Trie](https://en.wikipedia.org/wiki/Trie)
- [DFA](https://en.wikipedia.org/wiki/Deterministic_finite_automaton)

![Trie](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/Trie_example.png)

```rust
structure Node
    Children Node[Alphabet-Size]
    Is-Terminal Boolean
    Value Data-Type
end structure
```

上面代码展示了一个 **Trie** 的基本数据结构，除此之外我们还需要实现**查询**、插入、删除功能。

```java
class Trie {
    boolean isLeaf;
    Map<Character, Trie> children;
    public Trie() {
        isLeaf = false;
        children = new HashMap<>();
    }
    
    public void insert(String word) {
        Trie cur = this;
        for (char c : word.toCharArray()) {
            if (!cur.children.containsKey(c))
                cur.children.put(c, new Trie());
            cur = cur.children.get(c);
        }
        cur.isLeaf = true;
    }
    
    public boolean search(String word) {
        Trie cur = this;
        for (char c : word.toCharArray()) {
            if (!cur.children.containsKey(c))
                return false;
            cur = cur.children.get(c);
        }
        return cur.isLeaf;
    }
    
    public boolean startsWith(String prefix) {
        Trie cur = this;
        for (char c : prefix.toCharArray()) {
            if (!cur.children.containsKey(c))
                return false;
            cur = cur.children.get(c);
        }
        return true;
    }
}
```

## 约瑟夫环

**约瑟夫环** 通常用来求解这样一个问题：*一圈 **n** 人每次取第 **m** 个，被取到则离开圈子，问最后一个取到的是原先第几个人？*

如果直接模拟整个过程的话，每次都要考虑越过多少虚空位置（已经被删除），会很麻烦，因此我们要用递归思想。假设将上述问题抽象成 `f(n, m)`，并且我们已经知道 `f(n-1, m)` 的结果为 `k`，那么显然在抽取一个人之后，顺位 k 个人即能得到我们想要的结果。用数学来表示，即：

$$f(n, m) = (m + f(n - 1, m)) \\% n$$

[剑指 Offer 62. 圆圈中最后剩下的数字](https://leetcode.cn/problems/yuan-quan-zhong-zui-hou-sheng-xia-de-shu-zi-lcof/)

```java
class Solution {
    public int lastRemaining(int n, int m) {
        if (n == 1) return 0;
        
        int x = lastRemaining(n - 1, m);
        return (x + m) % n;
    }
}
```

[leetcode 390. 消除游戏](https://leetcode.cn/problems/elimination-game/)

## 二分法

二分法的核心在于取 `mid = l+r+1 >> 1` 后，遇到双中点情况时，我们取右侧中点。而为了修正这一举动防止死循环，我们遇到判断分支时，会对中点左移情况补偿（即加强左移）。具体表现为，预测中点往右侧时，我们令 `l = mid`, 预测中点往左移时，我们令 `r = mid - 1`，往左更进一步。另一方面，由于原中点驻留在 `l` 处，故将等于分支也归到此处。

或者采取邪道记法，`mid = l+r+1 >> 1` 取 **1**，而 `mid = l+r >> 1` 取 **0**。

[leetcode 367. 有效的完全平方数](https://leetcode.cn/problems/valid-perfect-square/)

```java
class Solution {
    public boolean isPerfectSquare(int num) {
        long l = 0, r = num;
        while (l < r) {
            long mid = l + r + 1 >> 1;
            long t = mid * mid;
            
            if (t == num) {
                return true;
            } else if (t > num) {
                r = mid - 1; // 当前 mid 过大，中点应该左移
            } else {
                l = mid; // 当前 mid 过小，中点应该右移
            }
        }
        return false;
    }
}
```

## 洗牌算法

洗牌算法最著名的当属 [Knuth shuffle](https://rosettacode.org/wiki/Knuth_shuffle) 算法，其原理是将“牌”分为有序无序两部分，每次从未打乱部分中选择一个元素加入到已打乱部分。

具体来说，假设有数组 `[1,2,3,4,5]`，那么随机抽一个和最末尾元素交换，比如得到 `[1,5,3,4,2]`，然后再从前四个继续，比如得到 `[1,4,3,5,2]`，再继续。

为什么这样洗牌是公平的呢，所谓公平指 **每个元素出现在每个位置的概率相等**。考虑上述设定，元素 **2** 出现在最后一个位置的概率是 **1/5**，而元素 **5** 出现在倒数第二个位置的概率则是 `4/5 x 1/4 = 1/5`，以此类推。

本人更啰嗦的讲解版本，每次都从一堆牌中随机抽取一张，可以用哈希表构建一个映射。一开始所有映射都指向未被抽取的卡牌，在抽牌过程中我们将哈希表中指向未抽取卡牌的映射放在前半部分，指向已抽取卡牌的映射放在后半部分。抽牌时，每次我们都从前半部分抽取一个映射，它必然指向一个未抽取卡牌。在我们获取这个映射并得到卡牌之后，我们让这个映射指向哈希表前半部分的最末元素所指向的映射（夺舍了）。这意味着该映射又指向了一个新的未抽取卡牌，以此类推，从而维持哈希表前半部分所有映射都指向未被抽卡牌的一致性。

[leetcode 384. 打乱数组](https://leetcode.cn/problems/shuffle-an-array/)

```java
class Solution {
    int[] init;
    int n;
    Random random = new Random();
    
    public Solution(int[] nums) {
        init = nums;
        n = nums.length;
    }
    
    public int[] reset() {
        return init;
    }
    
    public int[] shuffle() {
        int[] ret = init.clone();

        for (int i = 0; i < n; i++)
            swap(ret, i, i + random.nextInt(n - i));
        return ret;
    }

    void swap(int[] a, int l, int r) {
        int t = a[l];
        a[l] = a[r];
        a[r] = t;
    }
}
```

[leetcode 519. 随机翻转矩阵](https://leetcode.cn/problems/random-flip-matrix/)

```java
class Solution {
    Map<Integer, Integer> map = new HashMap<>();
    Random rand = new Random();
    int cnt, r, c;

    public Solution(int m, int n) {
        r = m;
        c = n;
        cnt = m * n;
    }
    
    public int[] flip() {
        int x = rand.nextInt(cnt--);
        int idx = map.getOrDefault(x, x);
        map.put(x, map.getOrDefault(cnt, cnt));
        return new int[]{idx / c, idx % c};
    }
    
    public void reset() {
        cnt = r * c;
        map.clear();
    }
}
```

## 香农熵

[香农熵](https://en.wikipedia.org/wiki/Entropy_(information_theory))又称为“进制猜想”，可以将其转换为 *猜测多维空间的某一点在何处* 的问题，即将待测点均匀分布在一个多维空间，而目标点能够用坐标系轻易标出，只要满足 $N^c >= buckets$。

在经典**可怜小猪**这题中，由于小猪可以保留一列，所以有 $N=k+1$。

[leetcode 458. 可怜的小猪](https://leetcode.cn/problems/poor-pigs/)

```java
class Solution {
    public int poorPigs(int buckets, int minutesToDie, int minutesToTest) {
        int k = minutesToTest / minutesToDie;
        return (int)Math.ceil(Math.log(buckets) /  Math.log(k + 1));
    }
}
```

## 数学

### 因数分解

```java
static void printDivisors(int n)
{
    for (int i = 1; i <= Math.sqrt(n); i++)
    {
        if (n % i == 0)
        {
            if (n/i == i)
                System.out.print(" " + i);
            else 
                System.out.print(i + " " + n/i + " ");
        }
    }
}
```

### 质数分解

```java
public static void resolvePrime(int n) {
    for (int i = 2; i <= n; i++) {
        while (n % i == 0) {
            System.out.println(i);
            n /= i;
        }
    }
}
```

### 快速幂

快速幂常用来求解基于模数下的 `a 的 b 次方`问题，其原理是将 b 按二进制位拆解从而降低计算复杂度。一个二进制数可以看作不同位上的 2 的 k 次方之和（i.e.，5 = 4 + 1），从而将 a 的 b 次方看作无数以 a 为底，2 的 k 次方为幂的数的积。

[leetcode 372. 超级次方](https://leetcode.cn/problems/super-pow/)

```java
class Solution {
    static final int MOD = 1337;
    public int superPow(int a, int[] b) {
        return dfs(a, b, b.length - 1);
    }

    int dfs(int a, int[] b, int u) {
        if (u == -1) return 1;
        return qpow(dfs(a, b, u - 1), 10) * qpow(a, b[u]) % MOD;
    }

    int qpow(int a, int b) {
        a = a % MOD;
        int ans = 1;
        while (b > 0) {
            if ((b & 1) == 1) ans = ans * a % MOD;
            a = a * a % MOD;
            b >>= 1;
        }
        return ans;
    }
}
```

### 按位取反

值得注意的是，`i & -i` 取最低比特 1 是很常见的技巧，另外有 `Integer.highestOneBit(num)` 的辅助函数可以直接用。

```java
class Solution {
    public int findComplement(int num) {
        int x = 0;
        for (int i = num; i != 0; i -= i & -i) x = i;
        return ~num & (x - 1);
        // return (~num) & ((Integer.highestOneBit(num)) - 1);
    }
}
```

### 极坐标

注意一点，用 `atan(dy/dx)` 函数只能求出 [-90˚，90˚] 而用 `atan2(dy, dx)` 可以求出 [-180˚, 180˚]。另外，极坐标求角度范围可以等效为**循环队列**问题，通过**倍长队列和滑动窗口组合**解决。

[leetcode 1610. 可见点的最大数目](https://leetcode.cn/problems/maximum-number-of-visible-points/)

```java
class Solution {
    public int visiblePoints(List<List<Integer>> points, int angle, List<Integer> location) {
        int a = location.get(0), b = location.get(1);
        int cnt = 0;
        double pi = Math.PI, t = angle * pi / 180;
        List<Double> q = new ArrayList<>();
        for (List<Integer> p : points) {
            int x = p.get(0), y = p.get(1);
            if (a == x && b == y) {
                cnt++;
                continue;
            }
            q.add(Math.atan2(y - b, x - a) + pi);
        }
        Collections.sort(q);
        
        int n = q.size(), max = 0;
        for (int i = 0; i < n; i++) q.add(q.get(i) + 2 * pi);
        for (int r = 0, l = 0; r < n * 2; r++) {
            while (l < r && q.get(r) - q.get(l) > t) l++;
            max = Math.max(max, r - l + 1);
        }
        cnt += max;
        return cnt;
    }
}
```

### 闰年规则

闰年 Leap year 规则如下：

1. 400 的倍数为闰年
2. 100 的倍数但非 400 的倍数为平年
3. 4 的倍数但非 100 的倍数为闰年

### 阶乘

[leetcode 172. 阶乘后的零](https://leetcode.cn/problems/factorial-trailing-zeroes/)

```java
class Solution {
    public int trailingZeroes(int n) {
        if (n == 0) return 0;
        int cnt = 0;
        while (n > 0) {
            cnt += n / 5;
            n /= 5;
        }
        return cnt;
    }
}
```

## 并查集

**并查集**首先将所有元素单独构成集合，每个集合只有一个根结点，后续合并集合时本质上是对两个集合的根结点进行条件判断。并查集的核心方法是 `union` 和 `find`，其中 `find` 方法返回所属集合根节点，并缩短查询路径将根节点设为父节点，而 `union` 将两个集合合并，并确定新集合的父节点。

[并查集原理](https://zhuanlan.zhihu.com/p/93647900)，简单来说并查集可以抽象为天下群英会帮主 pk 赛。

[leetcode 765. 情侣牵手](https://leetcode.cn/problems/couples-holding-hands/)

```java
class Solution {
    int[] p = new int[70];

    void union(int a, int b) {
        p[find(a)] = p[find(b)];
    }

    int find(int x) {
        return x == p[x] ? p[x] : (p[x] = find(p[x]));
    }

    public int minSwapsCouples(int[] row) {
        int n = row.length, m = n / 2;
        for (int i = 0; i < m; i++) p[i] = i;
        for (int i = 0; i < n; i += 2) union(row[i] / 2, row[i + 1] / 2);
        
        int cnt = 0;
        for (int i = 0; i < m; i++) 
            if (i == find(i)) cnt++;
        return m - cnt;
    }
}
```

## 图

### DAG 拓扑排序

拓扑排序首先记录点的入度，并根据所给有向边记录其后续节点，然后将入度为零的点入队。对其后续节点做处理后，将后续节点的入度减一，并判断入度为零时入队。

[leetcode 851. 喧闹和富有](https://leetcode.cn/problems/loud-and-rich/)

```java
class Solution {
    public int[] loudAndRich(int[][] richer, int[] quiet) {
        int n = quiet.length;
        int[][] dag = new int[n][n];
        int[] din = new int[n];
        for (int[] r : richer) {
            dag[r[0]][r[1]] = 1;
            din[r[1]]++;
        }

        LinkedList<Integer> q = new LinkedList<>();
        int[] ans = new int[n];
        for (int i = 0; i < n; i++) {
            ans[i] = i;
            if (din[i] == 0) q.add(i);
        }
        while(!q.isEmpty()) {
            int t = q.poll();
            for (int i = 0; i < n; i++) {
                if (dag[t][i] == 1) {
                    if (quiet[ans[t]] < quiet[ans[i]]) ans[i] = ans[t];
                    if (--din[i] == 0) q.add(i);
                }
            }
        }
        return ans;
    }
}
```

**最短路**、**最小生成树**、**线段树**

[涵盖所有存图方式的模版 by 三叶](https://mp.weixin.qq.com/s?__biz=MzU4NDE3MTEyMA==&mid=2247488007&idx=1&sn=9d0dcfdf475168d26a5a4bd6fcd3505d&chksm=fd9cb918caeb300e1c8844583db5c5318a89e60d8d552747ff8c2256910d32acd9013c93058f&mpshare=1&scene=23&srcid=0311tjKy74JijYzXhHo8Qob7&sharer_sharetime=1646964421353&sharer_shareid=1221771780968b30ef07c3f22cd356ed%23rd)

## 字符串

### 正则化处理

[leetcode 537. 复数乘法](https://leetcode.cn/problems/complex-number-multiplication/)

```java
class Solution {
    public String complexNumberMultiply(String num1, String num2) {
        String[] ss1 = num1.split("\\+|i"), ss2 = num2.split("\\+|i");
        int a = parse(ss1[0]), b = parse(ss1[1]);
        int c = parse(ss2[0]), d = parse(ss2[1]);
        int A = a * c - b * d, B = b * c + a * d;
        return A + "+" + B + "i";
    }
    int parse(String s) {
        return Integer.parseInt(s);
    }
}
```

`TODO` **kmp** 、**回文串**

[leetcode 28. 找出字符串中第一个匹配项的下标](https://leetcode.cn/problems/find-the-index-of-the-first-occurrence-in-a-string/)

## 动态规划

### 股票

做动态规划时，注意条件设置为当 x 状态，能够获取的最大利益。

[leetcode 123. 买卖股票的最佳时机 III](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-iii/)

```java
class Solution {
    public int maxProfit(int[] prices) {
        int n = prices.length;
        int[][][] dp = new int[n][2][3];
        dp[0][1][1] = -prices[0];
        dp[0][1][2] = -prices[0];
        for (int i = 1; i < n; i++) {
            for (int j = 1; j <= 2; j++) {
                dp[i][0][j] = Math.max(dp[i - 1][0][j], dp[i - 1][1][j] + prices[i]);
                dp[i][1][j] = Math.max(dp[i - 1][1][j], dp[i - 1][0][j - 1] - prices[i]);
            }
        }
        return dp[n - 1][0][2];
    }
}
```

### 模式匹配

[leetcode 10. 正则表达式匹配](https://leetcode.cn/problems/regular-expression-matching/)

```java
class Solution {
    public boolean isMatch(String s, String p) {
        int n = s.length(), m = p.length();
        boolean[][] f = new boolean[n + 1][m + 1];
        f[0][0] = true;
        for (int j = 1; j <= m; j++) 
            if (p.charAt(j - 1) == '*' && j > 1) f[0][j] = f[0][j - 2];
        for (int i = 1; i <= n; i++) {
            char a = s.charAt(i - 1);
            for (int j = 1; j <= m; j++) {
                char b = p.charAt(j - 1);
                if (a == b) {
                    f[i][j] = f[i - 1][j - 1];
                } else if (b == '.') {
                    f[i][j] = f[i - 1][j - 1];
                } else if (b == '*' && j > 1) {
                    char c = p.charAt(j - 2);
                    if (a != c && c != '.') f[i][j] = f[i][j - 2];
                    else f[i][j] = f[i][j - 2] || f[i - 1][j - 2] || f[i - 1][j];
                }
            }
        }
        return f[n][m];
    }
}
```

`TODO` **kmp**

### 背包 DP

[背包九讲](http://cuitianyi.com/Pack/ )

[背包九讲 v2](https://github.com/tianyicui/pack/blob/master/V2.pdf)

[三叶背包](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU4NDE3MTEyMA==&action=getalbum&album_id=1751702161341628417&scene=173&from_msgid=2247486107&from_itemidx=1&count=3&nolastread=1#wechat_redirect)

```javascript
f[i][v] = max{f[i-1][v], f[i-1][v - c[i]] + w[i]}
```

### 背包、完全背包、多重背包、混合背包

```javascript
for i = 1..N
    for v = 0..V
        f[v] = max{f[v], f[v - cost] + weight}
```

多重背包首先可以看作 **01 背包**，这样时间复杂度肯定很高，然后对相同物件进行二进制优化

## 双指针

[leetcode 15. 三数之和](https://leetcode.cn/problems/3sum/)

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        Arrays.sort(nums);
        int n = nums.length;
        List<List<Integer>> ans = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            int a = nums[i];
            if (a > 0) break;
            if (i > 0 && nums[i] == nums[i - 1]) continue;
            int l = i + 1;
            int r = n - 1;
            while (l < r) {
                int b = nums[l], c = nums[r];
                int sum = a + b + c;
                if (sum == 0) {
                    ans.add(Arrays.asList(a, b, c));
                    while (l < r && nums[l] == nums[l + 1]) l++;
                    while (l < r && nums[r] == nums[r - 1]) r--;
                    l++;
                    r--;
                }
                else if (sum > 0) r--;
                else l++;
            }
        }
        return ans;
    }
}
```

[leetcode 2055. 蜡烛之间的盘子](https://leetcode.cn/problems/plates-between-candles/)

```java
class Solution {
    public int[] platesBetweenCandles(String s, int[][] queries) {
        char[] arr = s.toCharArray();
        int n = arr.length, m = queries.length;
        int[] sum = new int[n + 1], l = new int[n], r = new int[n];
        for (int i = 0, j = n - 1, p = -1, q = -1; i < n; i++, j--) {
            if (arr[i] == '|') p = i;
            if (arr[j] == '|') q = j;
            l[i] = p;
            r[j] = q;
            sum[i + 1] = sum[i] + (arr[i] == '*' ? 1 : 0); 
        }
        int[] ans = new int[m];
        for (int i = 0; i < m; i++) {
            int a = r[queries[i][0]], b = l[queries[i][1]];
            if (a != - 1 && a <= b)
                ans[i] = sum[b + 1] - sum[a + 1];
        }
        return ans;
    }
}
```

[leetcode 76. 最小覆盖子串](https://leetcode.cn/problems/minimum-window-substring/)

```java
class Solution {
    public String minWindow(String s, String t) {
        char[] chars = s.toCharArray();
        char[] chart = t.toCharArray();
        int n = chars.length, m = chart.length;
        int[] hash = new int[128];
        for (char c : chart) hash[c]--;

        String ans = "";
        int cnt = 0;
        for (int l = 0, r = 0; r < n; r++) {
            hash[chars[r]]++;
            if (hash[chars[r]] <= 0) cnt++;
            while (cnt == m && hash[chars[l]] > 0) {
                hash[chars[l++]]--;
            }
            if (cnt == m && (ans == "" || r - l + 1 < ans.length())) {
                ans = s.substring(l, r + 1);
            }
        }
        return ans;
    }
}
```

## 格雷编码

格雷编码的**头尾连续性**可以通过对称实现。

[leetcode 89. 格雷编码](https://leetcode.cn/problems/gray-code/)

```java
class Solution {
    public List<Integer> grayCode(int n) {
        List<Integer> ans = new ArrayList<>();
        ans.add(0);
        for (int i = 0; i < n; i++) {
            int m = ans.size();
            for (int j = m - 1; j >= 0; j--) {
                ans.add(ans.get(j) | 1 << i);
            }
        }
        return ans;
    }
}
```

## 克隆

实现**图的深度克隆**，注意节点构建顺序。

[leetcode 133. 克隆图](https://leetcode.cn/problems/clone-graph/)

```java
class Solution {
    Map<Node, Node> map = new HashMap<>();
    public Node cloneGraph(Node node) {
        if (node == null) return node;

        if (map.containsKey(node)) return map.get(node);

        Node r = new Node(node.val);
        map.put(node, r);
        for (Node t : node.neighbors) 
            r.neighbors.add(cloneGraph(t));
        return r;
    }
}
```

[宫水三叶的刷题日记](https://github.com/SharingSource/LogicStack-LeetCode/wiki)

动态规划有两个核心：最优子结构和重叠子问题，当局部最优解（预设）能够得到全局最优解时我们使用动态规划。

## 记忆化搜索

首先构造一个 dfs，利用 dfs 从局部最优解得到全局最优解，此时极有可能超时，然后再在 dfs 基础上引入记忆化，将不变参作为外部变量。通常我会将其命名为 memo，或者另设一个从字符串到对应 value 的哈希表映射来缓解存储压力。

记忆化搜索的优化过程如下：dfs -> 记忆化搜索 -> dp -> 状态机 DP
