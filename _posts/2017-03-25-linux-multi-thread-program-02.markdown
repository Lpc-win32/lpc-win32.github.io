---
layout:     post
title:      "Linux多线程服务端编程：第二章 线程同步精要"
subtitle:   " \"码农生来只知道前进！\""
date:       2017-03-25 11:27
header-img: "img/post-bg-unix-linux.jpg" 
tags:
    - c++
    - c++11
    - linux
    - thread
---

线程同步的四项原则：

1. 最低限度共享对象，减少需要同步的场合。一个对象能不暴露给别的线程就不要暴露
2. 使用高级的并发编程构件：TaskQueue、C、Producer-Consumer Queue、CountDownLatch等
3. 使用底层同步原语时，只用非递归的互斥器，慎用读写锁，不用信号量
4. 除了使用“automic”整数之外，不自己编写lock-free代码，也不用“内核级”同步原语

### 2.1 互斥器

使用互斥器的原则：

- 用RAII手法封装mutex的创建、销毁、加锁、解锁这四个操作。
- 只用非递归的mutex
- 不手动调用lock()和unlock()函数，一切交给Guard对象构造和析构函数负责
- 防止因加锁顺序不同而导致死锁
- 不使用跨进程的mutex，进程间通信只用TCP socket
- 必要的时候使用PHTREAD_MUTEX_ERRORCHECK排错

#### 2.1.1 只使用非递归的mutex

mutex分为递归和非递归两种（可重入与非可重入）。它们唯一的区别在于：同一个线程可以重复对recursive mutex加锁，但是不能重复对non-recursive mutex加锁。  
使用non-revursive并非是为了性能，性能的差别并不大。在同一个线程中多次对non-recursive mutex加锁会立刻导致死锁，这是它的优点。而recursive mutex可能会隐藏代码里的一些问题。

```cpp
MutexLock mutex;
std::vector<Foo> foos;
void post(const Foo& f)
{
    MutexLockGuard lock(mutex);
    foos.push_back(f);
}
void traverse()
{
    MutexLockGuard lock(mutex);
    for (std::vector<Foo>::const_iterator it = foo.begin(); it != foos.end(); ++it) {
        it->doit();
    }
}

// 如果有一天，Foo::doit()间接调用了post(),那么会议欧戏剧性的结果：
// mutex非递归：死锁
// mutex递归，push_back()可能导致vector迭代器失效，程序偶尔会crash
```

#### 2.12 死锁

### 2.2 条件变量（condition variable）

互斥器（mutex）是加锁原语，用来排他性地方粉共享数据，它不是等待原语。在使用mutex的时候，我们总是期望尽可能快的拿到锁，用完尽快解锁，以不至于影响并发性与性能。

条件变量只有一种正确使用方式，几乎不可能用错。对于wait端：

1. 必须与mutex一起使用，该布尔表达式的读写需受此mutex保护。
2. 在mutex已上锁的时候才能调用wait()
3. 把判断布尔条件和wait()放到while循环中

```cpp
muduo::MutexLock mutex;
muduo::Condition cond(mutex);
std::deque<int>  queue;

int dequeue()
{
    MutexLockGuard lock(mutex);
    while (queue.empty()) {
        cond.wait();
        // 这一步会原子地unlock mutex并进入等待，不会与enqueue死锁
    }
    assert(!quqeue.empty());
    int top = queue.front();
    queue.pop_front();
    return top;
}
```

对于signal/broadcast端：

1. 不一定要在mutex已上锁的情况下调用signal
2. 在signal之前一般要修改布尔表达式
3. 修改布尔表达式通常要用mutex保护

```cpp
void enqueue(int x)
{
    MutexLockGuard lock(mutex);
    queue.push_back(x);
    cond.notify();
}
```

倒计时（CountDownLatch）是一种常用且易用的同步手段。它主要有两种用途：

- 主线程发起多个子线程，等这些子线程各自都完成一定的任务之后，主线程才继续执行。通常用于线程等待多个子线程完成初始化。
- 主线程发起多个子线程，子线程都等待主线程，主线程完成其他一些任务之后通知所有子线程开始执行。通常用于多个子线程等待主线程发出“起跑”命令。

### 2.3 不要用读写所和信号量

读写锁在实际的工作中，效率比起mutex并不见得有显著的性能提升。

条件变量配合互斥器可以完全替代信号量的功能，而且更不易用错。

### 2.4 封装MutexLock、MutexLockGuard

### 2.5 线程安全的Singleton实现

### 2.6 sleep(3)不是同步原语

生产代码中线程的等待可分为两种：一种是等待资源可用，一种是等待进入临界区以便读写共享数据。后一种等待通常极短，否则程序性能和伸缩性就会有问题
