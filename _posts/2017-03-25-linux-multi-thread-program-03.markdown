---
layout:     post
title:      "Linux多线程服务端编程：第三章 多线程服务器的适用场合与常用编程模型"
subtitle:   " \"码农生来只知道前进！\""
date:       2017-03-25 22:09
header-img: "img/post-bg-unix-linux.jpg" 
tags:
    - c++
    - c++11
    - linux
    - thread
---

### 3.1 进程与线程

线程的特点是共享地址空间，从而可以高效地共享数据，一台机器上的多个进程能高效地共享代码段，但不能共享数据。

### 3.2 单线程服务器的常用编程模型

高性能的网络程序中，使用的作为广泛的“non-blocking IO \+ IO multiplexing”这种模型，即Reactor模式。

在
高性能的网络程序中，使用的作为广泛的“non-blocking IO \+ IO multiplexing”这种模型中，程序的基本结构是一个事件循环，以事件驱动和事件回调的方式实现业务逻辑

```cpp
while (!done) {
    int timeout_ms = max(1000, getNextTimedCallback());
    int retval = ::poll(fds, nfds, timeout_ms);
    if (retval < 0) {
        // 处理错误
    } else {
        if (retval > 0) {
            // 处理IO事件
        }
    }
}
```

### 3.3 多线程服务器的常用编程模型

1. 每个请求创建一个县城，使用阻塞式IO操作。在Java 1.4引用NIO之前，这是Java网络编程的推荐做法。可惜伸缩性不佳。
2. 使用线程池，同样适用阻塞式IO操作。
3. 使用non-blocking IO \+ IO mutiplexing。即Java NIO的方式。
4. Leader/Follower等高级模式

#### 3.3.1 one loop per thread

此种模型下，程序里的每个IO线程有一个event loop（或者叫Reactor），用于处理读写和定时时间

这种方式的好处是：

- 线程数目基本固定，可以在程序启动的时候设置，不会频繁创建与销毁。
- 可以很方便地在线程间调配负载
- IO事件发生的线程是固定的，同一个TCP连接不必考虑事件并发

#### 3.3.2 线程池

### 3.4 进程间通信只用TCP

进程间通信首选Sockets，其最大的好处在于：可以跨主机，具有伸缩性。一台主机不够用的情况下。很自然用多机器来处理。

TCP本身是个数据流协议，除了直接使用它来通信外，还可在此之上构建RPC/HTTP/SOAP之类的上层通信协议。另外，除了点对点的通信之外，应用级的广播谢一也是非常有用的，可以方便点构建客观可控的分布式系统。

使用TCP长连接的好处有两点：

1. 容易定位分布式系统中的服务之间的依赖关系。只要在机器上运行 netstat -natp \| grep port 就能立刻列出用到某服务器客户端地址。TCP短连接与UDO不具备这一特性
2. 通过接收和发送队列的长度也较容易定位网络或程序故障
