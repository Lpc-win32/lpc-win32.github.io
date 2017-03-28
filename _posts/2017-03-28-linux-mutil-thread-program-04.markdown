---
layout:     post
title:      "Linux多线程服务端编程：第四章 C++多线程系统编程精要"
subtitle:   " \"码农生来只知道前进！\""
date:       2017-03-28 22:55
header-img: "img/post-bg-unix-linux.jpg" 
tags:
    - c++
    - c++11
    - linux
    - thread
---

学习多线程编程面临的最大的思维方式的转变有两点：

- 当前线程可能随时会被切换出去，或者说被强占
- 多线程程序中事件的发生顺序不再有全局统一的先后关系。

```cpp
bool running = false;
void threadFunc() {
    // note 1
    while (running) {
        // get task from queue
    }
}
void start() {
    muduo::Thread t(threadFunc);
    t.start();
    running = true; // 这部分会出现问题
}
```

> 有人认为在note 1的位置前加一小段（sleep）就能解决问题，但这是错的，无论加多大的延时，系统都有可能先执行while的条件判断，然后再执行running的赋值。

### 4.1 基本线程原语的选用

不推荐使用读写锁的原因是它往往造成提高性能的错觉（允许多个线程并发读），实际上很多情况下，与使用简单的mutex相比，它实际上降低了性能。

多线程系统编程的难点不在于学习线程原语，而在于理解多线程与现有C/C\+\+库函数和系统调用的交互关系。

### 4.2 C/C++系统库的线程安全性

C\+\+的iostream不是线程安全的，因为流式输出：
```cpp
std::cout << "Now is " << time(NULL);
// 等价于
std::cout.operator<<("Now is ")
         .operator<<(time(NULL));
```

### 4.3 Linux上的线程标识

pthread\_equal函数用于比较两个线程标识符是否相等。这带来了一系列问题：

- 无法打印输出pthread\_t，因为其类型是不确定的
- 无法比较pthread\_t的大小或计算hash值，因此无法用作关联容器的key
- 无法定义一个非法的phtread\_t值，用来表示绝对不可能存在的线程id，因此mutex没办法判断当前线程是否已经持有本锁
- pthread\_t值只在进程内有意义，与操作系统的任务调度之间无法建立有效联系
- phtread只保证统一进程之内，同一时刻的各个线程id不同；不能保证同一进程先后多个线程具有不同的id，更不用说一台机器上多个进城之间的线程id是唯一的了

```cpp
int main()
{
    pthread_t t1, t2;
    pthread_create(&t1, NULL, threadFunc, NULL);
    printf("%lx\n", t1);
    pthread_join(t1, NULL);
    
    pthread_create(&t1, NULL, threadFunc, NULL);
    printf("%lx\n", t2);
    pthread_join(t2, NULL);
}
```

> 上面的运行结果，两个tid是相同的，因此。pthread\_t不能确保全局唯一性。

不过，Linux上为我们提供了gettid()系统调用的返回值作为线程id。

- 它的类型是pid_t，通常是一个小整数
- 在现代Linux中，它直接表示内核的任务调度id，在/proc文件系统中可以轻易找到对应项：/proc/tid或/proc/pid/task/tid
- 使用top命令可以定位到线程
- 任何时刻都是全局唯一的，“Linux”分配的新pid采用递增轮回方法。短时间内启动的多个线程不可能会有相同的线程id
- 0是非法值，因为操作系统的第一个进程init的pid是1

### 4.4 线程的创建于销毁的守则

线程的创建和销毁是编写多线程程序的基本要素，线程的创建比销毁要容易的多。我们需要遵守下面的线程创建原则：

- 程序库不应该在未提前告知的情况下创建自己的“背景线程”
- 尽量用相同的方式创建线程
- 进入main()函数之前不应该启动线程（全局构造、各个编译单元之间的对象构造顺序是完全不同的）
- 程序中线程的创建最好能在初始化阶段全部完成

线程的销毁有几种方式：

- 自然死亡
- 非自然死亡
- 自杀（pthread_exit()）
- 他杀（其他线程调用pthread_cancel()）

> 线程正常退出的方式只有一种：即自然死亡。任何从外部强行终止线程的想法和做法都是错的。因为强行终止线程的话，线程没有机会清理资源。也没有机会释放已经持有的锁，其他线程如果再想对同一个mutex加锁，那么势必产生死锁

#### 4.4.1 pthread_cancel与C++

#### 4.4.2 exit(3)在C++中不是线程安全的

exit函数在C\+\+中的作用除了终止进程，还会析构全局对象和已经构造完的函数静态对象。这有潜在死锁的可能

```cpp
void someFunctionMayCallExit() {
    exit(1);
}
class GlobalObject {
    public:
        void doit() {
            MutexLockGuard lock(mutex_);
            someFunctionMayCallExit();
        }
        ~GlobalObject() {
            printf("GlobalObject: ~GlobalObject\n");
            MutexLockGuard lock(mutex_);    // 死锁
            printf("GlobalObject: ~GlobalObject cleanning\n");
        }
    private:
        MutexLock mutex_;
};
GlobalObject g_obj;

int main() {
    g_obj.doit();
}
```

### 4.5 善用\_\_thread关键字

\_\_thread是GCC内置的线程局部存储设置（TLS）。“非常高效”

\_\_thread使用规则：只能用于修饰POD类型，不能修饰class类型，==因为无法自动调用构造函数和析构函数。== 可以用于修饰全局变量、函数内的静态变量，但是不能用于修饰函数的局部变量或者class的不同成员变量。另外，\_\_thread变量的初始化只能用编译器常量。

### 4.6 多线程与IO

多个线程同时操作同一个socket文件描述符确实很麻烦，是得不偿失的。需要考虑的情况如下：

- 如果一个线程正在阻塞地read某个socket，而另一个线程close了此socket
- 如果一个线程正在阻塞accept某个listening socket，而另一个线程close了此socket
- 一个线程正准备read某个socket，而另一个线程close了此socket；第三个线程又恰好open了另一个文件描述符，其fd号码正好与前面的socket相同。这样程序的逻辑就混乱了

不考虑关闭文件描述符，只考虑读写：

- 如果两个线程同时read同一个TCP socket，两个线程几乎同时各自收到一部分数据，如何把数据拼成完整的消息？谁先到达？
- 如果两个线程同时write同一个TCP socket，每个线程都只发出半条消息，接收方收到数据如何处理？
- 如果给每个TCP socket配一把锁，让同时只能有一个线程读或写此socket，似乎可以解决问题，但这样不如始终同一个线程来操作此socket来的简单
- 非阻塞IO，收发消息的原子性不可能用锁来保证，因为这样会阻塞其他IO线程

> 因此。多线程程序应该遵循的原则是：每个文件描述符只由一个线程操作，从而解决消息收发的顺序性问题，也避免了文件描述符的各种race condition

### 4.7 用RAII包装文件描述符

### 4.8 RAII与fork()

fork()之后，子进程继承了父进程的几乎全部状态。不会继承：

- 父进程的内存锁，mlock、mlockall
- 父进程的文件锁，fcntl
- 父进程的某些定时器，setitimer、alarm、timer\_create等
- Others, See man 2 fork

### 4.9 多线程与fork()

多线程与fork()的协作性很差。这是POSIX系列操作系统的历史包袱。

fork()一般不能在多线程程序中调用，因为linux的fork()只克隆当前线程的thread of control，不克隆其他线程。由此看来，唯一安全的做法是在fork()后立即exec()执行另一个程序，彻底隔断紫禁城与父进程的联系

### 4.10 多线程与signal

Linux/Unix的信号与多线程可谓是水火不容。在单线程时代，边写信号处理函数就是一件棘手的事情，由于signal打断了正在运行的thread of control，在singnal handler中只能调用“可重入函数”

在多线程程序中，使用signal的第一原则是“不要使用signal”。包括：

- 不要用signal作为IPC的手段，包括不要用SIGUSR1等信号来触发服务端的行为。
- 不要使用基于signal实现的定时函数
- 不出动处理各种异常信号，只用默认语义：结束进程。有一个例外：SIGPIPI，服务器程序通常的做法是忽略此信号，否则如果对方断开连接，而本机继续write的话，会导致程序意外终止
- 在没有别的替代方法的情况下，把异步信号转换为同步的文件描述符事件。
