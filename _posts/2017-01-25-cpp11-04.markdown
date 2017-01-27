---
layout:     post
title:      "深入理解C++11（笔记）：第四章 新兵易学，老兵易用"
subtitle:   " \"Use c++11 for fun.\""
date:       2017-01-25 13:39
header-img: "img/post-bg-2015.jpg"
tags:
    - c++
---

### 4.1 右尖括号\>的改进

> 在C\+\+98中，有一条需要程序员规避的规则：如果在实例化模板的时候出现了连续的两个右尖括号，那么它们之间需要一个空格来进行分隔，以避免发生编译时的错误

```cpp
template <int i> class X {};
template <class T> class Y {};
Y<X<1> > x1;    // 编译成功
Y<X<2>> x2;    // 编译失败
```

> 事实上，除去潜逃的模板标识，在使用形如static_cast、dynamic_cast、reinterpret_cast，活着const_cast表达式进行缓缓的时候，我们也时常会遇到相同的情况。

```cpp
const vector<int> v = static_cast<vector<int>>(v);    //无法通过编译
```

> C\+\+98同样会将\>\>优先解析为右移。在C\+\+11中，这种限制被取消了。会智能的进行判断是否为右移符号。

### 4.2 auto类型推导

#### 4.2.1 静态类型、动态类型与类型推导

> 对于所谓的静态类型，类型检查主要发生在编译阶段；而对于动态类型，类型检查主要发生在运行阶段

> C\+\+11中类型推导的实现的方式之一就是重定义了auto关键字。另外一个实现是decltype

> auto声明的变量必须被初始化，以使编译器能够葱其初始化表达式中推导出其类型

#### 4.2.2 auto的优势

#### 4.2.2 auto的使用细则

> C\+\+11中，auto可以与指针和引用结合起来使用，使用的效果基本上会符合C/C\+\+程序员的想象

```cpp
int x;
int * y = & x;
double foo();
int & bar();
auto * a = &x;      // int *
auto & b = x;       // int &
auto c = y;         // int *
auto * d = y;       // int *
auto * e = &foo();  // 编译失败，指针不能指向一个临时变量
auot & f = foo();   // 编译失败，nonconst的左值不能和一个临时变量绑定
auto g = bar() ;    // int
auto & h = bar();   // int &
```

### 4.3 decltype

> RTTI机制是为每个类型产生一个type_info类型的数据，程序员可以在程序中使用typeid随时查询一个比纳凉的类型，typeid就会返回变量相应的type_info数据。

> 与auto类似，decltype也能进行类型推导，不过两者的使用方式有一定的区别。

```cpp
#include <typeinfo>
#inckude <iostream>
using namespace std;
int main()
{
    int i;
    decltype(i) j = 0;
    cout << typeid(j).name() << endl;   // i
    return 0;
}
```

> 如果对象的定义中有const或volatile限制符，使用decltype进行推导时，其成员不会继承const或volatile限制符

### 4.4 追踪返回类型

> 住总返回类型函数的声明方式如下：

```cpp
auto func(char *a, int b) -> int;
```

```cpp
class OuterType {
    struct InnerType {
        int i;
    };
    InnerType GetInner();
    InnerType it;
};
auto OuterType::GetInner() -> InnerType {
    return it;
}
```

> 最终返回类型的时候，InnerType不必写明其作用域

### 4.5 基于范围的for循环

> 很简单，代码中随处可见，例子如下：

```cpp
#include <verctor>
#include <iostream>
using namespace std;
int main() {
    vector<int> v = {1, 2, 3, 4, 5};
    for (auto i = v.begin(); i != v.end(); ++i) {
        cout << *i << endl;
    }
    for (auto e : v) {
        cout << e << end;;
    }
}
```
