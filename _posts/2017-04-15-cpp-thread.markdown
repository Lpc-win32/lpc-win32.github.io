---
layout:     post
title:      "c++ std::thread"
subtitle:   " \"c++11 thread\""
date:       2017-04-14 15:59
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - c++
    - c++11
    - thread
    - thread_join
    - thread_detach
    - gcc
---

### c++11标准库 - std::thread

让我们先来说说c++标准库为我们提供的std::thread。与绝大多数标准库一样，std::thread在使用上与用法上相较于posix标准的多线程库更加方便易于接受。但是，有一些方面需要注意。

#### 1. C\+\+'s most vexing parse问题

在上一篇章我们对这个问题已做过讨论，再次不在赘述。

#### 2. std::thread用法上面的坑

首先，我们编写了如下的代码：

```cpp
// FileNmae : my_thread.cpp
#include <iostream>
#include <thread>
void do_thread()
{
    std::cout << "This is func do_thread()" << std::endl;
}
int main(int argc, char **argv)
{
    std::thread my_thread(do_thread);
    my_thread.join();
    return 0;
}
```

g\+\+ my\_thread.cpp时，编译器会报出这样的错误：

```
    #error This file requires compiler and library support for the ISO C++ 2011 standard. 
This support is currently experimental, and must be enabled with the 
-std=c++11 or -std=gnu++11 compiler options.
```

g\+\+ \-std=c\+\+11 my\_thread时，生成a.out可执行文件，看起来是对了，但是./a.out执行时，会出现如下的错误

```
    what():  Enable multithreading to use std::thread: Operation not permitted
```

这种情况下，无论采用静态连接还是动态链接，都会出现这样的问题。

出现这个情况的原因，是因为我们少链接了pthread，std::thread在底层依赖着posix标准的pthread。

g\+\+ \-std=c\+\+11 \-lpthread my\_thread.cpp 即可正常运行。

#### 3. detach and join

1. detach还是join？

探寻这个问题，我们先来看看一种不正确的用法：

```cpp
std::thread my_thread(do_thread);
return 0;
```

如上代码所示：启动一个线程就直接结束程序的main函数。由于C程序运行的机制可知，do\_thread一定没有运行。（此处下文做一个说明。在此标注1）

因此运行会报如下的错误：

```
    terminate called without an active exception
```

使用detach写法：

```cpp
std::thread my_thread(do_thread);
my_thread.detach();
return 0;
```

这种情况下，不会有任何结果打印。因为main线程提前结束，do\_thread()相关资源已被销毁，不会再打印任何消息。

使用join写法：

```cpp
std::thread my_thread(do_thread);
my_thread.join();
return 0;
```

这种情况下，main线程会等待子线程结束并回收子线程相关资源后，接着运行。子线程结束之前会阻塞在my\_thread.join()的位置。

#### 问题：何时detach()？何时join()？

由于线程的生命周期是不确定的。我们很难确定main线程的指令执行到哪一条时，会有子线程终止。如果detach()或join()声明的位置过于靠后，会失去其相应的意义，尤其是detach()，因为默认的线程是non-detach的
