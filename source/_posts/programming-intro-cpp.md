---
title: C++ 入门/扫盲
date: 2023-06-09 08:18:00
cover: https://i.imgur.com/Ysqn3vw.jpeg
thumbnail: https://i.imgur.com/Ysqn3vw.jpeg
categories: [编程, C++, 扫盲]
tags:
    - c++
    - 入门
    - TODO
toc: true
---

简单列出一些 C++ 语言的特点和知识，目前处于 “知其然” 阶段。

<!--more-->

## 复合类型

```c++
int a, &b, *c;
```

上面这行代码定义了一个变量、引用和一个指针，指针和引用可能容易被混淆，二者都只是一个 “路牌”，用来指向真正的对象，但指针本身是一个真正的对象，而引用不是。比如，我们可以让 `*c = 1`，但不能使用 `&b = 1`。注意，`int *c` 表示 c 是一个 int，它是一个指针，`*` 是用来修饰 c 的而不是用来补充 `int`。

## 智能指针

智能指针可以自动释放不再引用的对象，而不用再纠结 new 和 delete 的使用，底层用到的是 “引用计数”。

仿函数

## 模板类

就是泛型参数相关那套。

```c++
template <typename T> T foo(T* v) {
    T tmp = *v;
    // ...
    return tmp;
}
```

namespace

成员模板

泛化、特化和偏特化
偏特化又分为个数的偏和范围的偏。

数量不定的模板参数
variadic templates，模板参数包，函数参数类型包，函数参数类型包

引用 Reference
引用、指针的区别，Java 都是引用

虚函数
相当于 Java 中的接口，函数中的函数，好像很难解释，确实有一点抽象，又有一点 AOP 的思想。

new 和 delete
如何将两个动作分解。
