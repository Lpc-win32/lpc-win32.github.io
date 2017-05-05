---
layout:     post
title:      "C++程序中如何避免死锁？"
subtitle:   " \"c++11 dead-lock\""
date:       2017-05-05 15:05
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - c++
    - c++11
    - thread
    - mutex
    - dead-lock
    - gcc
---

### 死锁的产生未必因为“锁”

虽然锁是产生死锁的一般原因，但也不排除死锁出现在其他地方。无锁的情况下，仅需要每个std\:\:thread对象调用join()，两个线程就能产生死锁。这种情况下，没有线程可以继续运行，因为他们正在互相等待。这种情况很常见，一个线程会等待另一个线程，其他线程同时也会等待第一个线程结束，所以三个或更多线程的互相等待也会发生死锁。

### 解决死锁的一些建议

- 避免嵌套锁
- 避免在持有锁时调用用户提供的代码（因为你并不清楚用户想要做什么）
- 使用固定顺序获取锁
- 使用锁的层次结构

### 什么是锁的层次结构？

锁的层次的意义在于提供对运行时约定是否被坚持的检查。这个建议需要对你的应用进行分层，并且识别在给定层上所有可上锁的互斥量。  
当代码试图对一个互斥量上锁，在该层锁已被低层持有时，上锁是不允许的。你可以在运行时对其进行检查，通过分配层数到每个互斥量上，以及记录被每个线程上锁的互斥量。

```cpp
hierarchical_mutex high_level_mutex(10000);
hierarchical_mutex low_level_mutex(5000);

int do_low_level_stuff();

int low_level_func()
{
    std::lock_guard<hierarchical_mutex> lk(low_level_func);
    return do_low_level_stuff();
}

void high_level_stuff(int some_param);

void high_level_func()
{
    std::lock_guard<hierarchical_mutex> lk(high_level_mutex);
    high_level_stuff(low_level_func());
    // 高层级内部嵌套低层级锁是完全ok的，反之是不允许的
}

void thread_a()
{
    high_level_func();
}
```

std\:\:lock\_guard<>模板与用户定义的互斥量类型一起使用。虽然hierarchical\_mutex不是C++标准的一部分，但是它写起来很容易  
下面原封不动贴出《C\+\+并发编程》书中hierarchical\_mutex的简单的实现

```cpp
class hierarchical_mutex
{
    std::mutex internal_mutex;

    unsigned long const hierarchy_value;
    unsigned long previous_hierarchy_value;

    static thread_local unsigned long this_thread_hierarchy_value;

    void check_for_hierarchy_violation()
    {
        if(this_thread_hierarchy_value <= hierarchy_value) {
            throw std::logic_error(“mutex hierarchy violated”);
        }
    }

    void update_hierarchy_value()
    {
        previous_hierarchy_value=this_thread_hierarchy_value;
        this_thread_hierarchy_value=hierarchy_value;
    }

public:
    explicit hierarchical_mutex(unsigned long value):
        hierarchy_value(value),
        previous_hierarchy_value(0)
    {}

    void lock()
    {
        check_for_hierarchy_violation();
        internal_mutex.lock();
        update_hierarchy_value();
    }

    void unlock()
    {
        this_thread_hierarchy_value=previous_hierarchy_value;
        internal_mutex.unlock();
    }

    void try_lock()
    {
        check_for_hierarchy_violation();
        if(!internal_mutex.try_lock())
            return false;
        update_hierarchy_value();
        return true;
    }
};

thread_local unsigned long hierarchical_mutex::this_thread_hierarchy_value(ULONG_MAX);
```

