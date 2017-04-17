---
layout:     post
title:      "c++ std::mutex"
subtitle:   " \"c++11 mutex\""
date:       2017-04-17 17:29
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - c++
    - c++11
    - thread
    - mutex
    - lock_guard
    - gcc
---

> 今天，在这里分享一下std::mutex

### 对比POSIX标准pthread\_mutex

回忆一下：繁琐的posix mutex初始化操作：

1. pthread\_mutex\_t \= PTHREAD\_MUTEX\_INITIALIZER;
2. pthread\_mutex\_init()初始化方式

尤其是第二种初始化方式，用起来比较麻烦

C\+\+中mutex初始化方式：std::mutex my\_mutex;

C\+\+中通过实例化std::mutex创建互斥量，通过调用成员函数lock()进行上锁，unlock()进行解锁。看到这里，可能觉得std::mutex就是对posix mutex中的一层简单的封装。其实不是的，因为lock()与unlock()是我们不推荐使用的。因为我们必须要在每个函数出口都去调用unlock()，也包括异常的情况。

C\+\+标准库为互斥量提供了一个RAII语法的模块类std::lock\_guard，会在构造的时候提供已锁的互斥量，并在析构的时候进行解锁，从而保证了一个已锁的互斥量总是会被正确的解锁

```cpp
#include <list>
#include <mutex>
#include <algorithm>

std::list<int> some_lost;
std::mutex some_mutex;

void add_to_list(int new_value)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    some_list.push_back(new_value);
}

bool list_contains(int value_to_find)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    return (std::find(some_list.begin(), some_list.end(), balue_to_find)
                     != some_list.end());
}
```

使用互斥量来保护数据，并不是仅仅在每一个成员函数中都加入一个std::lock\_guard对象那么简单‘一个迷失的指针或引用，将会让这种保护形同虚设。不过，检查迷失指针或引用是很容易的，只要没有成员函数通过返回值或者输出参数的形式向其调用者返回指向受保护数据的指针或引用，数据就是安全的。更为危险的是下面这种情况：将保护数据作为一个运行时参数

```cpp
class some_data
{
    private:
        int a;
        std::string b;
    public:
        void do_something();
};
class data_wrapper
{
    private:
        some_data data;
        std::mutex m;
    public:
        template<typename Function>
        void process_data(Function func)
        {
            std::lock_guard<std::mutex> l(m);
            func(data);     // 1. 传递“保护”数据给用户函数
        }
};

some_data * unprotected;

void malicious_function(some_data &proteted_data)
{
    unprotected = &protected_data;
}

data_wrapper x;

void foo()
{
    x.process_data(malicious_function);     // 2. 传递一个恶意函数
    unprotected->do_something();            // 3. 在无保护的情况下访问保护数据
}
```
