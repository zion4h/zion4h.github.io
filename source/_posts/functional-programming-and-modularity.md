---
title: 函数式编程和模块化
date: 2022-12-11 12:00:00
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img009.jpg
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img009.jpg
toc: true
categories: 
    - [编程, 函数式编程]
tags: 
    - typescript 
    - 函数式编程
---

本文涉及项目源于**张天戈**老师的高级软件开发技术课设，语言使用 Typescript，任务是对论文 [_Why Functional Programming Matters_](https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf)，中间有反复参考**函数式编程**的 [术语](https://github.com/shfshanyue/fp-jargon-zh) 和 Typescript 的 [文档](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-func.html)。

<!--more-->

这篇论文年代久远（据闻是 **1984** 年！），当时的 John Hughes 对软件发展提出了一个前瞻性的观点：\* 编程会越来越复杂，对于代码复用或者说模块化编程的需求会越来越旺盛。\* 恰好函数式编程的两个特性，**高阶函数**和**惰性求值**在模块化编程方面优势巨大，而这也解释了论文标题的来源。

## 前言

函数式编程的特点是不包含赋值语句，**变量的值一旦给定就不再更改**。这意味着一条函数式程序在执行过程中不会产生任何副作用，即只帮助计算结果而不影响其他任何数据。此外，这种属性还解放了函数执行的顺序，因为程序是**引用透明**的。

但仅仅是没有赋值，没有副作用，没有控制流这些还不够，有的人甚至听起来会觉得有点无聊。因为给**结构化编程语言**的参数加上 `const` 也能达到类似的效果，但后者显然能够做到更多东西。结构化编程能够对小模块编码，其通用模块能够重复使用，每个模块还可以独立测试，_那我们为什么还需要用函数式编程？_

我们在模块化的过程中，通常需要将问题划分为子问题后求解，最后合并解决，划分问题的方式直接取决于将解决方案合并在一起的方式。上面所描述的这些优势本质上都是模块化带来的，函数式编程在模块化方面的优势在于能够**高效合并子问题**，论文剩余部分详细描述了函数式编程语言的两种用来合并子问题的胶水，即**函数组合**和**程序组合**。

## 函数组合

### 列表抽象

函数式编程允许用一些简单的函数组合抽象成一个更复杂的函数，文中用列表处理问题举例。

$$
listof\ \ast\ ::=\ Nil\ |\ cons\ \ast\ (listof \ \ast)
$$

现在，我们规定一个 `list` 可以表示为一个空列表 `Nil` 或者一个 `Cons`，这里的 `Cons` 意为由一个元素和一个 `list` 构造而成的一个结构。当然其中的元素可以是任何类型，后文常用 `Integer` 类型，我在代码中用的是 `number` 类型。根据上面的描述，我们可以设计出代码如下：

```typescript
abstract class ListBase<A> {
    // 可视化打印
    toString(this:List<A>): string {
        var msg = ""   
        switch (this.tag) {
            case "none": {
                msg = "List[]"
                break
            }
            case "cons": {
                msg = "List[" + this.h
                var cur = this.t
                while (cur.tag === 'cons') {
                    msg += ", " + cur.h
                    cur = cur.t
                }
                msg += "]"
                break
            }
            case "tree": {
                msg += "Node " + this.h
                if (this.t.tag === 'cons') {
                    msg += "(" + this.t.toString() + ")"
                } 
                break
            }
        }
        return msg
    }
}

type List<T> = Nil<T> | Cons<T> | Node<T>
class Nil<T> extends ListBase<T> {
    // 方便类型推断的特殊技巧
    readonly tag: "none" = "none"
  
    // 单例模式
    static readonly Nil: List<never> = new Nil()
    private constructor() {
        super()
    }
}

class Cons<T> extends ListBase<T>{
    readonly tag: "cons" = "cons"
  
    // h 表示第一个元素，t 表示剩余部分构成的 list
    constructor(readonly h: T, readonly t: List<T>) {
        super()
    }
}
```

为了方便创建和打印使用这个列表对象，我编写了如下代码：

```typescript
const list = <T>(...s: T[]): List<T> => {
    if (s.length === 0)
      return none()
    else
      return cons(s[0])(list(...s.slice(1)))
}
const cons = <T>(h: T) => (t: List<T>) => new Cons(h, t)
const none = <T>(): List<T> => Nil.Nil

const Xs = list(1, 1, 2, 3, 5, 8, 13, 21)
log("Xs is " + Xs.toString()) 
// Output:Xs is List[1, 1, 2, 3, 5, 8, 13, 21]
```

### sum 函数的诞生

接下来，文中定义了一个用来求总的递归函数 `sum`，它允许接受一个 `list`（注意，`Nil` 和 `Cons` 都是 `list`）。当对 `Nil` 求和时显然结果为 0，这样边界条件就确定了，而对 `Cons` 求和又可被分解为让 `Cons` 内部的数字加上另一个 `list` 的求和结果。文字描述有点绕口，但其实就是一个简单的递归，不再赘述。

$$
sum\ Nil\ = \ \boxed0\\
sum(Cons\ n\ list)\ = n\ \boxed+\ sum\ list
$$

### Reduce 和 柯里化

观察上面的式子，真正实际参与计算的部分是 0 和 `+`，如此我们可以借助 `foldr` 函数将 `sum` 抽象，在这个过程中，**foldr 表现出了一种通用的递归模式**。实际上文中提到的 `foldr` 概念，就是我们常用的 **reduce**，这里我还用到了**柯里化技巧**方便复用。

```typescript
// 常用函数
const add = (a:number)=>(b:number):number => a + b
const double = (n: number) => 2*n

// foldr
const foldr = <T>(f: (a: any)=>(b: T)=>T, x: T) => (l: List<any>):T => {
    if (l.tag === "none")
        return x
    else 
        return f(l.h)(foldr(f, x)(l.t))
}
const sum = foldr(add, 0)
log("return: ", sum(Xs))
// Output:return:  54
```

后面的**连乘**、**异或**、**求与**都大同小异，然后是 `append` 函数的实现：

```typescript
// append
const append = <T>(a: List<T>)=>(b: List<T>): List<T> => foldr(cons, b)(a)
log("append", Z1.toString(), "and", Z2.toString(), "is", append(Z1, Z2).toString())
// Output:append List[1, 2] and List[3, 4] is List[1, 2, 3, 4]
```

类似地，我们还可以借助 `foldr` 写出求列表长度的函数 `length`，以及对倍增列表中每个元素值的 `doubleall` 函数：

```typescript
const count = <T>(a: T, n: number) => n + 1
const length = foldr(count, 0)
log("count", Xs.toString(), "is", length(Xs))
// Output:count List[1, 1, 2, 3, 5, 8, 13, 21] is 8

const doubleandcons = (n: number, list: List<number>) => {
    return cons(2 * n, list)
}
const doubleall = foldr(doubleandcons, none())
log("doubleall", Xs.toString(), "is", doubleall(Xs).toString())
// Output:doubleallcons List[1, 1, 2, 3, 5, 8, 13, 21] is List[2, 2, 4, 6, 10, 16, 26, 42]
```

观察后发现，`doubleandcons` 实际上是两个函数 `double` 和 `cons` 的组合，因此我们可以将其模块化为 `fandcons` 函数，代码如下：

```typescript
const double = (n: number) => 2*n
const fandcons = <T>(f: (x: T) => T) => (el: T, list: List<T>) => {
    return cons(f(el), list)
}
const doubleandcons2 = fandcons(double)
const doubleall2 = foldr(doubleandcons2, none())
log("doubleall(base on fandcons)", Xs.toString(), "is", doubleall2(Xs).toString())
// Output:doubleall(base on fandcons) List[1, 1, 2, 3, 5, 8, 13, 21] is List[2, 2, 4, 6, 10, 16, 26, 42]
```

### compose

而为了构造 `compose` 函数，我发现之前的 `cons` 需要**柯里化**，重构后得到代码如下：

```typescript
const compose = (f1: any, f2: any) => (x: any) => f1(f2(x))
const fandcons = (f: any) => compose(cons)(f)
const doubleandcons = fandcons(double)
const doubleall = foldr(doubleandcons, none())
log("doubleall", Xs.toString(), "is", doubleall(Xs).toString())
// Output:doubleall List[1, 1, 2, 3, 5, 8, 13, 21] is List[2, 2, 4, 6, 10, 16, 26, 42]
```

### map

当我们设计好 `doubleall` 后又发现另一个模块函数 `map`，**map 接收一个函数，并对列表中每个元素执行，最终得到一个新的列表**，我们可以这样设计：

```typescript
const compose = (f1: any)=>(f2: any) => (x: any) => f1(f2(x))
const map = (f: any) => foldr(compose(cons)(f), none())
const doubleall = map(double)
log("doubleall2", Xs.toString(), "is", doubleall2(Xs).toString())
// Output:doubleall2 List[1, 1, 2, 3, 5, 8, 13, 21] is List[2, 2, 4, 6, 10, 16, 26, 42]
```

然后我们还可以计算二维矩阵的总和函数 summatrix：

```typescript
// summatrix
const summatrix = compose(sum)(map(sum))
log("summatrix", mat.toString(), "is", summatrix(mat))
// Output:summatrix List[List[1, 2], List[3, 4], List[1, 1, 2, 3, 5, 8, 13, 21]] is 64
```

上面这些例子都是从一个最简单的 sum 函数模块化而来。

### 标签树

现在考虑另一个例子，**有序标签树**，一颗树由一个根结点 `label` 和**子树列表**构成，每个子树都是有序标签树。

思路和上面差不多，慢慢地耐心设计：

```typescript
class Node<T> extends ListBase<T>{
    readonly tag: "tree" = "tree"

    constructor(readonly h: T, readonly t: List<Node<T>>) {
        super()
    }
}

const node = <T>(h: T) => (l: List<Node<T>>): Node<T> 
    => new Node(h, l)

// tree
// 重写了下 node 方便造数据来测试
const tree = <T>(h: T) => (...subtrees: Node<T>[]) 
    => new Node(h, list(...subtrees))

const tree4 = tree(4)()
const tree3 = tree(3)(tree4)
const tree2 = tree(2)()
const tree1 = tree(1)(tree2, tree3)
log(tree1.toString())
log(tree2.toString())
log(tree3.toString())
log(tree4.toString())
// Node 1(List[Node 2, Node 3(List[Node 4])])
// Node 2
// Node 3(List[Node 4])
// Node 4
```

然后继续，由于前面我们梳理的比较顺畅，所以 sumtree 和 labels 函数都顺利实现：

```typescript
// folctree
const foldtree = <T>(f: any, g:any, a:T)=>(l: List<any>):T=>{
    if (l.tag === "none")
        return a
    else if (l.tag === "tree")
        return f(l.h)(foldtree(f, g, a)(l.t))
    else
        return g(foldtree(f, g, a)(l.h))(foldtree(f, g, a)(l.t))
}

// sumtree
const sumtree = foldtree(add, add, 0)
log(sumtree(tree1))
// Output: 10

// labels
const labels = foldtree(cons, append, none())
log(labels(tree1).toString())
// Output:List[1, 2, 3, 4]
```

最后是 doubletree：

```typescript
// maptree
const maptree = (f: any)=> foldtree(compose(node)(f), cons, none())
// doubletree
const doubletree = maptree(double)
log(doubletree(tree1).toString())
// Output:Node 2(List[Node 4, Node 6(List[Node 8])])
```

简单总结一下，**在编码过程中我一直在反复重构代码**。由于 Typescript 本身特性，如何设计良好的类型定义是真的很费心力，尤其在下一章引入**惰性求值**后之前所设计的函数基本要全部推倒重写。

## 程序组合

和函数组合不同，形如 $(g\ .\ f)=g (f\ input)$ 的程序组合可能会有如下场景，即 f 仅在 g 尝试读取输入时才运行并提供输出，而当 g 停止取数据时则 f 停止计算，这实际上就是**惰性求值**。

### 惰性求值

为了方便和论文后面描述的内容统一，我们先将之前的代码重构，为了便于区分，将 `list` 重命名为 `stream`，并将之前的函数用新的方式复现：

```typescript
//  简化打印语句：方便测试
export const log = (msg?: any, ...optionalParams: any[]) => console.log(msg, ...optionalParams)

//  lazy
const memorize = <T>(f: () => T): () => T => {
    let memo: T | null = null
    return () => {
        if (memo === null)
            memo = f()
        return memo
    }
}

type Stream<T> = Nil<T> | Cons<T>
type Pair = [number, number]

abstract class StreamBase<A> {
    toString(this: Stream<A>): string {
        var msg = "Stream("
        switch (this.tag) {
            case "none": break
            case "cons": {
                msg += this.h()
                var cur = this.t()
                while (cur.tag === 'cons') {
                    msg += ", " + cur.h()
                    cur = cur.t()
                }
            }
        }
        msg += ")"
        return msg
    }
}

class Nil<T> extends StreamBase<T> {
    static readonly NIL: Stream<never> = new Nil()

    readonly tag: "none" = "none"

    private constructor() {
        super()
    }
}

export class Cons<T> extends StreamBase<T>{
    readonly tag: "cons" = "cons"

    constructor(readonly h: () => T, readonly t: () => Stream<T>) {
        super()
    }
}

const cons = <T>(h: () => T, t: () => Stream<T>): Stream<T> => new Cons(memorize(h), memorize(t))
const none = <T>(): Stream<T> => Nil.NIL

const stream = <T>(...s: T[]): Stream<T> => {
    if (s.length === 0)
        return none()
    else
        return cons(() => s[0], () => stream(...s.slice(1)))
}

const foldr = <A, B>(f: (a: A, b: () => B) => B, x: () => B, s: Stream<A>): B => {
    if (s.tag === "none")
        return x()
    return f(s.h(), () => foldr(f, x, s.t()))
}
const Xs = stream(1, 1, 2, 3, 5, 8, 13, 21)
const Z1 = stream(1, 2)
const Z2 = stream(3, 4)
const M1 = stream(Z1, Z2, Xs)
const add = (a: number, b: () => number): number => a + b()
const double = (n: number) => 2 * n
const sum = (s: Stream<number>) => foldr(add, () => 0, s)
// Output:return:  54
// append
const append = <T>(s1: Stream<T>, s2: () => Stream<T>): Stream<T> => {
    return foldr((a, b) => cons(() => a, b), s2, s1)
}
// length
const count = <T>(a: T, n: () => number) => n() + 1
const length = <T>(s: Stream<T>): number => foldr(count, () => 0, s)
const flatMap = <A, B>(s: Stream<A>, f: (a: A) => Stream<B>): Stream<B> => {
    return foldr((a, b) => append(f(a), b), () => none(), s)
}
// map Stream 函数映射
const map = <A, B>(s: Stream<A>, f: (a: A) => B): Stream<B> => flatMap(s, a => stream(f(a)))
const doubleall = (s: Stream<number>) => map(s, double)

// summatrix
const summatrix = (mat: Stream<Stream<number>>): number =>
    sum(map(mat, sum))
log("Xs is " + Xs.toString())
// Output:Xs is List[1, 1, 2, 3, 5, 8, 13, 21]
log("return: ", sum(Xs))
// Output:return:  54
log("append", Z1.toString(), "and", Z2.toString(), "is", append(Z1, ()=>Z2).toString())
// Output:append List[1, 2] and List[3, 4] is List[1, 2, 3, 4]
log("count", Xs.toString(), "is", length(Xs))
// Output:count List[1, 1, 2, 3, 5, 8, 13, 21] is 8
log("doubleall", Xs.toString(), "is", doubleall(Xs).toString())
// Output:doubleall List[1, 1, 2, 3, 5, 8, 13, 21] is List[2, 2, 4, 6, 10, 16, 26, 42]
log("summatrix", M1.toString(), "is", summatrix(M1))
// Output:summatrix List[List[1, 2], List[3, 4], List[1, 1, 2, 3, 5, 8, 13, 21]] is 64
```

### 平方根的无限序列

我们可以用 **Newton-Raphson 平方根算法**去计算一个数的平方根，算法本质上是在计算一系列近似值，每个值都由前一个值计算得到并且更接近真实结果（即目标值的平方根）。这里我们用 `next` 函数抽象优化过程，继续观察 `next` 函数，我们可以再抽象出 `repeat` 函数。

```typescript
//  [1]squareroot(N)  牛顿-拉弗森公式
const next = (n: number) => (x: number): number =>
    (x + n / x) / 2

const repeat = <T>(f: (x: T) => T, a: T): Stream<T> =>
    cons(() => a, () => repeat(f, f(a)))
```

而为了方便取元素，我还在 `StreamBase` 内部设计了 take 函数：

```typescript
abstract class StreamBase<A> {
    //  取前 n 个元素
    take(this: Stream<A>, n: number): Stream<A> {
        if (this.tag === "none" || n <= 0)
            return none()

        const self = this
        return cons(this.h, () => self.t().take(n - 1))
    }
    
    // omit others...
}
```

### 误差处理

借助 `take` 函数，我们可以定义一个误差处理函数 `within`，用来**截停一个无限的近似序列**，当相邻两个元素误差小于等于 `eps` 时就截止：

```typescript
const within = (eps: number, s: Stream<number>): Stream<number> => {
    if (s.tag === "none")
        return none()

    const [a, b] = s.take(2).toList()
    if (Math.abs(a - b) <= eps) {
        return cons(s.h, () => cons(() => b, () => none()))
    }
    return cons(s.h, () => within(eps, s.t()))
}
```

我们还可以定义另外一种误差处理方式 `withre`，用比率接近 1 去替换绝对误差接近 0:

```typescript
const withre = (eps: number, s: Stream<number>): Stream<number> => {
    if (s.tag === "none")
        return none()

    const [a, b] = s.take(2).toList()
    if (Math.abs(a - b) <= eps * Math.abs(b)) {
        return cons(s.h, () => cons(() => b, () => none()))
    }
    return cons(s.h, () => withre(eps, s.t()))
}
```

最后，得到我们想要的 `sqrt` 函数：

```typescript

const sqrt = (a0: number, eps: number, n: number): Stream<number> =>
    within(eps, repeat(next(n), a0))
const relativesqrt = (a0:number, eps:number, n:number): Stream<number> =>
    withre(eps, repeat(next(n), a0))
```

### 微分

我们知道对于某个点微分的结果等于斜率，当 h 比较小时，可以用 `easydiff` 计算函数 f 在 x 附近的斜率。

```typescript
//  [2]numerical differentiation
const easydiff = (f: (x: number) => number, x: number) => (h: number) =>
    (f(x + h) - f(x)) / h
```

可是虽然 h 越小斜率计算越正确，但当到达某个临界点后随着 h 进一步变小，斜率又会因舍入误差逐渐失真。因此为了找合适的斜率，我们采取的策略可以用合适的 `eps` 去选择足够准确的第一个近似值：

```typescript
const halve = (x: number): number => x / 2
const differentiate = (h0: number, f: (a: number) => number, x: number): Stream<number> =>
    repeat(halve, h0).map(easydiff(f, x))
```

### improve

论文进一步探讨了如何在 `differentiate` 的基础上对其进行优化，`order` 函数是为了确定函数的阶数，从而利用 `elimerror` 根据阶数去消除误差，最终组合得到了 `improve` 函数：

```typescript
const elimerror = (n: number, s: Stream<number>): Stream<number> => {
    if (s.tag === 'none')
        return none()
    if (s.take(2).length() < 2) return s

    const [a, b] = s.take(2).toList()
    return cons(() => (b * Math.pow(2, n) - a) / (Math.pow(2, n) - 1),
        () => elimerror(n, s.t()))
}
const order = (s: Stream<number>): number => {
    const head3elem = s.take(3)
    if (head3elem.length() < 3) {
        throw new Error("[Data Shape Error]: the element number of input \'Stream\' cannot be smaller than 3.")
    }

    const [a, b, c] = head3elem.toList()
    return a === b ? 0 : Math.round(Math.log2((a - c) / (b - c) - 1))
}
const improve = (s: Stream<number>): Stream<number> => {
    if (s.tag === 'none')
        return s
    return elimerror(order(s), s)
}
```

### super

由于在数学上，对于一个近似序列的 `improve` 本身也可以被 `improve`，由此我们可以设计一个 `super` 函数（这里由于 `super` 本身是语言关键字，所以该用 `superman` 替代）**反复优化**：

```typescript
const second = <T>(s: Stream<T>): T => {
    const head2elem = s.take(2)
    if (head2elem.length() < 2) {
        throw new Error("[Data Shape Error]: the element number of input \'Stream\' cannot be smaller than 2.")
    }

    const [a, b] = head2elem.toList()
    return b
}
const superman = (s: Stream<number>): Stream<number> =>
    map(repeat(improve, s), second)
```

但是在实际测试过程中发现，当 `order = 0` 时，`elimerror` 不再适用，因此修改原来的代码如下：

```typescript
const elimerror = (n: number, s: Stream<number>): Stream<number> => {
    if (s.tag === 'none')
        return none()
    if (n === 0) 
        return s.t()
    if (s.take(2).length() < 2) return s

    const [a, b] = s.take(2).toList()
    return cons(() => (b * Math.pow(2, n) - a) / (Math.pow(2, n) - 1),
        () => elimerror(n, s.t()))
}

log("\n===Numerical Differentiation Test===\n")
const fn1 = (x: number): number =>
    Math.pow(x, 3) + x + 13
const dfn1 = (x: number): number =>
    3 * Math.pow(x, 2) + 1
log("fn1 = X3 + x + 13")
log("d.fn1 = 3 * X2 + 1")
log("\n[example 1]: d.fn1(3)")
log("expect:", dfn1(3))
const fn2 = differentiate(1, fn1, 3)
log("init:", fn2.take(10).toString())
log("improve:", improve(fn2).take(10).toString())
log("super:", within(0.0001, superman(fn2)).take(10).toString())
```

### 积分

论文最后一个例子讨论了积分，`easyintegrate` 函数只能给出一个不太靠谱的值，尤其当两个端点靠的过于远的时候。为了得到更准确的积分，可以取 a 到 b 的中点 mid，然后分别计算 a 到 mid 和 mid 到 b 的积分，然后将二者加起来。文中设计了一个生成近似序列的函数 `integrate`：

```typescript
//  [3]numerical integration
const easyintegrate = (f: (x: number) => number, a: number, b: number): number =>
    (f(a) + f(b)) * (b - a) / 2

const addpair = (x: Pair): number => {
    const [a, b] = x
    return a + b
}

//  打包：将两个 Stream 的元素按序两两执行二元运算
const zip2 = <T>(s1: Stream<T>, s2: () => Stream<T>): Stream<Pair<T>> => {
    const self = s1
    const another = s2()
    if (self.tag === "none" || another.tag === "none")
        return none()
    else
        return cons(() => [self.h(), another.h()], () => zip2(self.t(), another.t))
}

const integrate_origin = (f: (x: number) => number, a: number, b: number): Stream<number> => {
    const mid = (a + b) / 2
    return cons(
        () => easyintegrate(f, a, b),
        () => map(zip2(integrate_origin(f, a, mid), () => integrate_origin(f, mid, b)), addpair)
    )
}
```

但其实观察可以发现，integrate 重复计算了 a、b、mid 处的 f 值，为了消除这种重复计算，有了：

```typescript
const integ = (f: (x: number) => number, a: number, b: number, fa: number, fb: number): Stream<number> => {
    const m = (a + b) / 2
    const fm = f(m)
    return cons(
        () => (fa + fb) * (b - a) / 2,
        () => map(zip2(integ(f, a, m, fa, fm), () => integ(f, m, b, fm, fb)), addpair)
    )
}
const integrate = (f: (x: number) => number, a: number, b: number): Stream<number> =>
    integ(f, a, b, f(a), f(b))
```

最后，结合之前可以测试文中给的两个例子：

```typescript
log("\n===Numerical Integration Test===\n")
log("[ example 1 ]: improve (integrate f 0 1)")
log("                where f x=1/(1+x * x)")
log("integrate f = arctan(x), so (integrate f 0 1) = arctan(1) - arctan(0)")
log("expect:", Math.atan(1) - Math.atan(0))
const ft = (x: number): number => 1 / (1 + x * x)
log("Return:", withre(0.00000001, improve(integrate(ft, 0, 1))).toString())

log("\n[ example 2 ]: super (integrate sin 0 4)")
log("integrate sin = -cos(x), so (integrate sin 0 4) = -cos(4) + cos(0)")
log("expect:", -Math.cos(4) + Math.cos(0))
log("Return:", withre(0.00000001, superman(integrate(Math.sin, 0, 4))).toString())
```

```shell
===Numerical Integration Test===

[ example 1 ]: improve (integrate f 0 1)
                where f x=1/(1+x * x)
integrate f = arctan(x), so (integrate f 0 1) = arctan(1) - arctan(0)
expect: 0.7853981633974483
Return: Stream(0.7833333333333333, 0.7853921568627452, 0.7853981256146768, 0.7853981628062056, 0.7853981633882091)

[ example 2 ]: super (integrate sin 0 4)
integrate sin = -cos(x), so (integrate sin 0 4) = -cos(4) + cos(0)
expect: 1.6536436208636118
Return: Stream(1.0617923583434352, 1.5780150025674877, 1.690241935320747, 1.6633363775825898, 1.6536816398263603, 1.6536435134376641, 1.6536436209110699, 1.6536436208636065)
```
