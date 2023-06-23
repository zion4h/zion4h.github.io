---
title: 并发编程
date: 2023-05-21 20:23:03
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img013.jpg
thumbnail: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img013.jpg
categories: [编程, 并发]
tags:
    - 并发
    - TODO
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

*问题：Java 内存模型是如何保证对象初始化是线程安全的？是否必须通过加锁才能在线程之间安全地公开对象？*

Java 内存模型保证对象初始化是线程安全的，不需要显式地加锁。具体来说，当一个线程执行到对象初始化代码时，Java 内存模型通过 happens-before 原则保证该线程先前所做的所有操作都对其他线程可见，然后再执行对象初始化代码。这样，其他线程在访问该对象时将访问到正确的、完整的、已经初始化的对象实例。

Happens-Before 原则是 Java 并发编程中一个重要的概念，它描述了一个事件在时间上的先后顺序。具体地说，如果操作 A happens-before 操作 B，那么操作 A 对共享变量的修改将对操作 B 可见，即操作 A 的结果被操作 B 观察到。

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

上面代码是优化过的，原始版本没有 `volatile`，但随着 JVM 的不断被优化，我们应该尽量避免使用这个关键字。

## 并发工具包

用 ReentrantLock 来替换内置锁，来获取死锁中断、超时释放功能。

### 可中断的锁

```java
ReentrantLock l1 = new ReentrantLock();
ReentrantLock l2 = new ReentrantLock();
Thread t1 = new Thread(()->{
    try {
        l1.lockInterruptibly();
        Thread.sleep(1000);
        l2.lockInterruptibly();
    } catch (InterruptedException e) {
        System.out.println("t1 interrupted.");
    }
});
```

### 超时

设置超时时间尽管避免了无尽死锁，但也有可能导致死锁，比如所有锁一起超时。一些技巧可以改善这种情况，比如为不同锁设置不同超时时间，但整体来说通过设置超时来处理死锁不是一个好的选择。

### 交替锁

首先锁住头两个结点，如果判断不在这两个节点点插入，则释放第一个节点，锁住第三个节点，如此遍历。

与此同时，如果我们想测量链表长度，也可倒序遍历，每次锁住一个节点即可。

### 条件变量

一个条件变量通常需要和一把锁相关联，线程在开始等待这个条件之前需要先获取这把锁。

```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
lock.lock();
try {
    while (a < 100) {
        condition.await();
    }
} finally {
    lock.unlock();
}
```

### 原子变量

也是用来替换内置锁的，一种无锁非阻塞的解决方案。

问题 1 什么是公平锁？何时使用？使用非公平的锁会怎样？

公平锁让多个线程遵循 FIFO 原则获取锁，优点是可以避免**线程饥饿**现象的发生，但如果使用非公平的锁会使得锁的获取变快。线程饥饿问题是指由一些线程持续获取锁导致另一些线程长久等待，甚至永远无法获取。

## 并发数据结构

### 写入时复制

之后的函数式语言就更牛逼了，因为它的竞争资源本身不会变化，因此在并发时并不需要保护竞争资源，从而根本没有并发问题。

为什么要有线程池？1. 拒绝服务攻击 2. 手动创建开销大。另外，线程池的大小会考虑到任务本身，一般 CPU 密集的任务会接近可用核树，而 IO 密集的会更大。

java.util.concurrent 包中的 ArrayBlockingQueue 是一个并发队列，非常适合消费者-生产者模式，提供了高效的并发 put、take 方法。take 会在取空时阻塞直到非空，put 会在放满时阻塞直到有足够空间。

TODO 作者是如何一步步改进程序并发性能的：1. 引入线程池 2. 引入 concurrentHashMap3. 将对竞争资源的访问转换成处理离线任务再合并。

问题 1 阅读 ForkJoinPool 文档——fork/join 框架和线程池区别？分别适用什么场景？

Fork/Join 框架是一个基于工作窃取算法的任务调度框架，而线程池则是用于执行一组线程任务的框架。具体来说，ForkJoinPool 线程池中的每一条线程都有一个自己的工作队列（WorkQueue），这个工作队列是双端队列，从自己队列的头部取得任务，从尾部添加任务，如果自己的队列中没有任务了，它又会去“偷”其他线程的任务。这一点就是工作窃取算法。

问题 2 什么是 countDownLatch 和 CyclicBarrier？

CountDownLatch 和 CyclicBarrier 都是 Java 中的多线程工具类。

CountDownLatch 是一个计数器，可以用来控制多个线程的并发执行。CountDownLatch 有一个计数值，调用一次 countDown() 方法计数器的值就会减 1，当计数器的值减为 0 时，表示所有线程都已经执行完毕，等待在 await() 方法上的线程就可以继续执行。

CyclicBarrier 也是一个计数器，但与 CountDownLatch 不同的是，CyclicBarrier 的计数值只能在初始化时设置，所有线程都到达后，它会释放所有等待的线程，并且将计数值重置为初始值。CyclicBarrier 适合需要多个线程协同完成某个任务的场景。

问题 3 Amdahl 定律是什么？

Amdahl 定律是一个公式，它描述了在添加更多处理器或核心时，计算机系统可以达到的理论最大性能加速比。它表明，系统的加速比将受到程序中无法并行化的部分的限制。换句话说，如果一个程序由可并行化和不可并行化部分组成，则即使增加更多处理器，不可并行化部分也会限制系统的整体加速比。

Amdahl 定律的公式为：

速度提升比 = 1 / (1 - P + P/N)

其中，P 是可并行化程序的部分比例，N 是处理器的数量，速度提升比是预期的性能改进。该公式假设问题规模保持不变，处理器之间没有通信或同步成本。

Amdahl 定律强调了识别和优化程序中无法并行化的关键部分的重要性，以实现在添加更多处理器时的最佳性能提升。

线程与锁模型最大的好处就是能被轻松集成到各种语言中，它本质上是将硬件工作形式化，而这也意味着它在语言层面其实并没有提供多大帮助，而且还容易带来不确定性的隐患。因为不确定，所以很难测试和复现问题。作者这里用赛车做了一个类比，如何从一堆废墟中找到问题，答案是完善数据记录。

一些好的并发编程建议：1. 访问共享变量时同步 2. 按照全局固定顺序来取锁 3. 持有锁时避免调用外星方法。
