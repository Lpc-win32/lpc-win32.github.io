---
layout:     post
title:      "what is __builtin_expect?"
subtitle:   " \"likely()与unlikely()....\""
date:       2017-04-08 00:25
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - c++
    - c++11
    - likely&unlikely
    - gcc
---

### What is \_\_builtin\_expect?

想一个问题。比方说一个函数的返回值为0的概率低于百万分之1，但是出于程序的稳健性考虑，我们必须对此返回值进行逻辑判断。否则程序可能会造成崩溃甚至非预期效果。  
ok，我们加上了if判断语句后，无疑加大程序的开销。

GCC (version >= 2.96)提供了\_\_builtin\_expect给程序员使用，目的是为了将“分支转移”的信息提供给编译器，这样编译器可以对代码进行优化，以减少指令跳转带来的性能下降

如果不明白上述内容，那么你可能见过likely()与unlikely()，这两个函数在Linux内核源码中出现的比重极高。

```cpp
// likely()条件为true的可能性非常大
#define  likely(x)        __builtin_expect(!!(x), 1)
// unlikely()条件为false的可能性非常大
#define  unlikely(x)      __builtin_expect(!!(x), 0)
```

现在处理器都是流水线的，有些里面有多个逻辑运算单元，系统可以提前取多条指令进行并行处理，但遇到跳转时，则需要重新取指令，这相对于不用重新去指令就降低了速度。  
所以就引入了likely和unlikely，目的是增加条件分支预测的准确性，cpu会提前装载后面的指令，遇到条件转移指令时会提前预测并装载某个分支的指令。unlikely表示你可以确认该条件是极少发生的，相反likely 表示该条件多数情况下会发生。编译器会产生相应的代码来优化cpu 执行效率。 
