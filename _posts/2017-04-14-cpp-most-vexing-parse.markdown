---
layout:     post
title:      "C++'s most vexing parse"
subtitle:   " \"C++中最令人头痛的语法解析\""
date:       2017-04-14 15:59
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - c++
    - c++11
    - most_vexing_parse
    - gcc
---

### C\+\+中最令人头痛的语法解析：most vexing parse

我们先来看一个例子：std::thread类。  
如同大多数C++标准库一样，std::thread可以用可调用（callable）类型构造，将带有函数调用符类型的实例传入std::thread类中，替换默认的构造函数。  
因此：我们可以写出如下的代码：

```cpp
class background_task
{
public:
  void operator()() const
  {
    do_something();
    do_something_else();
  }
};

// 注意下面两段代码：可以正常运行
background_task f;
std::thread my_thread(f);
```

> 注意：如果你传递了一个临时变量，而不是一个命名的变量；C++编译器会将其解析为函数声明，而不是类型对象的定义。

```cpp
/*
 * background_task f;
 * std::thread my_thread(f);
 * 当这两条语句简化成下面这一条语句的时候，会产生非预期的效果
 */

std::thread my_thread(background_task());
```

这里相当与声明了一个名为my\_thread的函数，这个函数带有一个参数(函数指针指向没有参数并返回background\_task对象的函数)，返回一个std::thread对象的函数，而非启动了一个线程。

使用下面的方式可以解决most vexing parse问题：

1. std::thread my\_thread((background\_task()));
2. std::thread my\_thread{background\_task()};
3. 使用lambda表达式避免该问题，示例如下：

```cpp
std::thread my_thread([]{
  do_something();
  do_something_else();
});
```
