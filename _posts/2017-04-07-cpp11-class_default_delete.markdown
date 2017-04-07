---
layout:     post
title:      "C++11 '=default','=delete'"
subtitle:   " \"一些关键点的个人领悟....\""
date:       2017-04-07 14:25
header-img: "img/post-bg-2015.jpg" 
tags:
    - c++
    - c++11
    - =delete
    - =default
---

### 问题：\=delete与=\default有什么好处？

#### 1 \=default的好处

通常我们在开发中会遇到这种尴尬的情况：当我们自定义了一个构造函数。那么默认的构造函数、拷贝构造、移动拷贝构造函数。都不再自动生成。  
注意一点：自定义了析构函数，不会影响编译器自动生成默认的构造函数。  
当我们在class A中自定义了A(int param)构造函数时，再采用A a方式默认构造一个对象的时候，编译器会提示没有这样的构造方法。因此在C\+\+11之前，我们需要手动构造一系列我们可能会需要的默认构造函数。非常的麻烦

这个为题在C\+\+11中得到了解决：

```cpp
class A 
{
    public:
        A(int a)
        {
            // do some things you want
        }
        A() = default;
};

int main(void)
{
    A a(3);         // ok，此处各个版本的C++都不会存在问题
    A b();          // 由于=default的存在，此处ok
    A c = b;        // 惊奇的发现，拷贝构造的自动实现也被加上了。
                    // 因此一个=default解决了很多不必要的麻烦
    return 0;
}
```

#### 2 \=delete的优点

C\+\+11版本之前，我们想实现一个类不能拷贝构造需要这么写：

```cpp
class A
{
    public:
        A() {}
        ~A() {}
    private:
        A(A &a) {}  // 如果此处{}替换成了;将会无比的麻烦
        A &operator=(A & a) {}
};
```

在C\+\+11中，我们可以通过下面的方式来实现一个noncopyable函数

```cpp
class noncopyable
{
    public:
        noncopyable() = default;
        ~noncopyable() = default;
    private:
        noncopyable(const noncopyable&) = delete;
        void operator=(const noncopyable&) = delete;
};
```

> 注意一点：\=delete是可以针对任何成员函数的，不局限与构造与析构函数

当我们用\=delete修饰了基类的拷贝构造函数时，子类的拷贝构造如果没有明确的自定义的话。子类的拷贝构造也是失效的
