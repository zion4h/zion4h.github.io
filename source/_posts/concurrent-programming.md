---
title: 并发编程
date: 2023-05-21 20:23:03
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img013.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img013.jpg
categories: [编程, 并发]
tags:
    - 并发
toc: true
---
> “[The Free Lunch Is Over](http://www.gotw.ca/publications/concurrency-ddj.htm)” -- Herb Sutter

对于痛恨大厂“挤牙膏”的我而言，Intel 创始人提出的**摩尔定律**听起来再"美味"不过，一个延续数十年名副其实的“神话”。但随着半导体工业发展到极致后，为了继续提高芯片性能，只能转向**多核化**发展。而多核化又进一步促生**并发编程**的需求。
<!--more-->

![Intel CPU Introductions](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/CPU.png)

## 并发还是并行？

一个简单的比方，人不能同时吃饭喝酒，不然你就会呛着。并发的意思就是，你可以抿一口酒再吃一口菜，这样酒也喝了，菜也尝了。而并行的意思就是，你是一个长着两张嘴的人，真的能够同时畅饮纯生并大口嚼着牛排。

1. 并发理论上能够提升处理任务的速度
2. 但实际运行过程中，线程遇到竞争资源会等待，而这个等待时间是漫长的，最终导致整体效果较差
3. 改进的方法比较直接，将竞争资源的颗粒度细化，从而减少线程等待时间

通常，我们会将并行和多核画上等号，但实际上计算机可以在多个层次用到并行技术。比如位级并行，“为什么 32 位计算机比 8 位计算机速度快？”，因为两个 32 位数的加法它只需要执行 1 次，而后者需要 4 次。但这显然是有瓶颈的，因此 128 位的计算机迟迟得不到推出。指令并行，或者说 CPU 加速，当然指令的并行也带来了“内存可见性”问题。数据并行，或者说 GPU 加速，可以对大量数据施加同一个操作。任务并行，即多处理器，这是我们最频繁打交道的部分，也是大家讨论“并行”时的一般性范畴。

## 互斥和内存模型

> 不应该在产品代码上直接使用 Thread 类等底层服务。

很多语言都提供了并发接口，但是仅仅只是将计算机系统底层提供的指令简单抽象包装了一下。如果你在生产代码中实际去使用（比如 `new Thread`），那绝对是有问题的。Java 在并发包给我们提供了很多工具类，需要熟悉掌握。

### 线程

当我们讨论并发时，其实大多数时候都在围绕线程和锁，下面是一个创建 Java 线程的例子：

```java
public static void main(String[] args) throws InterruptedException {
    Thread t = new Thread(()-> System.out.println("Hello from thread."));
    t.start();
    Thread.yield();
    System.out.println("Hello from main.");
    t.join();
}
```

`start` 方法启动线程，`yield` 方法让出处理器占用（重新竞争），`join` 等待线程死亡。

### 锁

**线程**之间通过**共享内存**进行通信，本质上是控制流。但当**竞态条件**发生时，每次运行结果都可能不一样。为了避免这种情况的发生，我们将对竞争资源同步访问，也就是加**锁**。

比如经典的“[i++ 线程安全](https://stackoverflow.com/questions/680097/ive-heard-i-isnt-thread-safe-is-i-thread-safe)”问题。

### 乱序执行的代码

在 Java 中，为了提高运行效率需要乱序执行代码，导致副作用产生，而 **Java 内存模型**则提供了一个标准告诉我们发生了什么副作用。

造成乱序的可能原因：

- **编译器**的静态优化
- **JVM** 的动态优化
- **硬件指令**的乱序执行

可参考：

- *[Java 内存模型](http://www.cs.umd.edu/~pugh/java/memoryModel/) by William Pugh*
- *[JSR-133 FAQ](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html) by William Pugh*

### 死锁

有人说为了让代码安全运行，干脆让全部代码都同步执行好了，但这样线程会频繁阻塞，失去了并发的意义。并且，当引入多个锁之后可能会造成**死锁**。一个经典死锁模型是——[哲学家进餐问题](https://en.wikipedia.org/wiki/Dining_philosophers_problem)。

```java
public class Philosopher extends Thread {
    private Chopstick left, right;
    private Random random;

    public Philosopher(Chopstick left, Chopstick right) {
        this.left = left;
        this.right = right;
        random = new Random();
    }

    @Override
    public void run() {
        try {
            while(true) {
                Thread.sleep(random.nextInt(1000)); // think
                synchronized (left) {
                    synchronized (right) {
                        Thread.sleep(random.nextInt(1000)); // eat
                    }
                }
            }
        } catch (InterruptedException e) {}
    }
}
```

一个最简单的解决思路是：给筷子编号，按全局顺序取筷子避免头尾冲突。

*TODO：Java 内存模型是如何保证对象初始化是线程安全的？是否必须通过加锁才能在线程之间安全地公开对象？*

### 双重检查锁模式

**双检锁 DCL** 是一个反面模式，反面模式来自 *GoF* 的《**设计模式**》一书，用来指代那些经常出现，却又低效且有待优化的模式。

```java
public class DCL {
    private volatile static DCL instance;

    private DCL() {}

    public static DCL getInstance() {
        if (instance == null) {
            synchronized (DCL.class) {
                if (instance == null) {
                    instance = new DCL();
                }
            }
        }
        return instance;
    }
}
```

上面代码是优化过的，原始版本没有 `volatile`。

## 并发工具包

### 可中断的锁

### 交替锁

### 条件变量

### 原子变量

## 并发数据结构

### 写入时复制

之后的函数式语言就更牛逼了，因为它的竞争资源本身不会变化，因此在并发时并不需要保护竞争资源，从而根本没有并发问题。
