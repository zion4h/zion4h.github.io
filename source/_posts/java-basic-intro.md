---
title: Java 扫盲（一）基础中的基础
date: 2023-04-05 00:14:33
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/wallpaperimg1006.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/wallpaperimg1006.jpg
categories: [编程, Java, 扫盲]
tags:
    - java
    - intro
toc: true
---

根据 **TIOBE**，Java 直到 2023 年依然是主流编程语言。在 Java 8 以前掌握这门语言是一件比较简单的事情，但在 8 之后学习曲线开始变陡，即使经过多次减负学习成本也在慢慢增加，尤其需要耗费在学习各种框架的原理和应用之上。_Java 扫盲系列_不会像官方 Doc 一样事无巨细地对语法絮絮叨叨不停，而是会按我个人喜好将其拆解分析，着重于**方法论**上。

<!-- more -->

![TIOBE Index 2023](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/TIOB-2023.png)

在学习一门编程语言之前，按照惯例，附上三篇大牛漫谈雄文，分别是 **Paul Graham**、**Bruce Eckel** 和 **Peter Norvig**。

[⭐IT 大牛谈编程语言（网文 3 篇）译自 阮晓寰](https://program-think.blogspot.com/2012/05/weekly-share-5.html)

## Object 通用方法

在 Java 中，**Object 类**是所有类的祖先，因此我们应该先去见见这个 “老祖宗”，看看它所定义的常用方法。

* `getClass` 返回该对象运行时类的信息，我们常用这个方法获取对象的类名
* `toString` 默认返回类似 “**对象名 @14114e1c**” 这种形式，后面跟着的是_散列码的 16 进制表示_

### hashCode

`hash` 中文译作 “**散列**”，本身代表一种基本的计算机概念，每个 `hash` 都代表一种哈希生成算法，而 Java 中的 `hashCode` 会返回一个和内存地址相关的 32 位整数，从而利用对象唯一性来保证其哈希码的唯一性。

### equals

原始的 `equals` 方法只是让两个对象用 `==` 直接比较，因此 `String` 类有重写该方法，当我们设计类的时候想要正确比较同样也需要重写这个方法。如

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;

    EqualExample that = (EqualExample) o;

    if (x != that.x) return false;
    if (y != that.y) return false;
    return z == that.z;
}
```

### clone

拷贝又分为浅拷贝和深拷贝，浅拷贝就是返回同一个引用对象，而深拷贝就是返回和原始对象的引用类型但引用的是不同对象，前者基本没啥意义，我们谈拷贝默认指深拷贝。在 Java 中，如果一个对象没有实现 `Cloneable` 接口，默认抛出 `CloneNotSupportedException`，另外 `clone` 方法是复制一个数组最快的方法。

为什么我们需要克隆方法而不是直接用 `new` 创建一个新对象？因为用 `new` 创建对象会耗费大量时间。

如何在实际应用中使用 `clone`？如果是已经设计部署好的类，我们可以定义一个父类并实现 `Cloneable` 接口。

下面，用一个实际例子说明如何正确使用 `clone` 方法：

```java
public class demo {
    public static void main(String[] args) throws CloneNotSupportedException {
        B b = new B("X ming", 12);
        System.out.println(b);
        B c = (B)b.clone();
        System.out.println(c);
    }
}

class A implements Cloneable{

}

class B extends A{
    String name;
    int age;

    B (String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public String toString() {
        return super.toString() + ", " + name + ", " + age;
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

### wait, notify 和 notifyAll

线程类指那些继承自 Thread 的类，唤醒指让线程状态从阻塞态转为就绪态。

![Thread status](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/thread-status.png)

`wait`, `notify` 和 `notifyAll` 方法是 Java 留给线程通信的，本质是修改锁对象状态。比如其中的 `notify` 就是用来唤醒**单个线程**的。

```java
synchronized( lockObject )
{ 
  while( ! condition )
  { 
    lockObject.wait();
  }
   
  //take the action here;
}
```

## String

Java 中的 `String` 底层是 `char` 数组，但是实现了 `CharSequence` 接口的类还有 `StringBuffer` 和 `StringBuilder`。我们会用 `StringBuilder` 创建_可变字符串_，为什么专门提出这个概念，字符串不都是可以修改吗？并不是，Java 中的 `String` 是 `final` 修饰的，当我们尝试修改它时，本质上是用一个新创建的实例去替换它。

`String` 对象有两种创建方式，一种是字面量创建的方式，如

```java
    String s1="Welcome";  
    String s2="Welcome";//It doesn't create a new instance  
```

用这种方式创建的字符串会直接再**字符串常量池**里面创建，大大节省内存效率。而另外一种创建方式就是 `new` 关键字了，略过不表。

## 反射

通过**反射**，我们可以知道一个对象的类是什么，它能够执行哪些方法。换句话说，我们可以在运行时通过反射调用 `private` 方法甚至构造一个对象。

![Java lang pkg](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/javalang.png)

我们的调试和测试工具就有用到反射，常用的框架比如 **JDBC**、**Spring**、**SpringBoot** 等都用到了反射，将数据库参数这种都写在配置文件中而不是代码里，让程序运行的时候自动去取。

总的来说，反射是帮助 Java 实现自动化必不可少的一个帮手。

## 异常

Java 用异常捕获来处理运行时发生的错误，比如 `ClassNotFoundException`, `IOException`, `SQLException`, `RemoteException` 等等。一般来说，异常的名称就告知程序员问题所在，而详细信息则描述了上下文信息。

异常又可以细分为 **Checked Exceptions** 和 **Unchecked Exceptions**，前者是可以恢复的。

![Exceptions](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/exception-java.jpg)

我们可以利用 Java 的异常机制来检测失败和成功而不是笼统返回 `true` 或者 `false`。

## 参考

[【 1 】Java 新手的通病系列](https://program-think.blogspot.com/2009/01/defect-of-java-beginner-0-overview.html)

[【 2 】On Java 8](https://m.ituring.com.cn/book/2935)
