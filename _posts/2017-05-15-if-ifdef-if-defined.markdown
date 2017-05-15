---
layout:     post
title:      "#if、#ifdef、#if defined()之间的区别"
subtitle:   " \"很简单，一看便知....\""
date:       2017-05-15 10:56
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - c
    - gcc
    - system_macro
---

### \#if的使用说明

\#if的后面接的是表达式

```c
#if (MAX==10)||(MAX==20)
    code...
#endif
```

它的作用是：如果(MAX==10)||(MAX==20)成立，那么编译器就会把其中的#if 与 #endif之间的代码编译进去（注意：是编译进去，不是执行！！）

### \#if defined()的适用

\#if后面接的是一个宏

```c
#if defined (x)
    ...code...
#endif
```

这个\#if defined它不管里面的“x”的逻辑是“真”还是“假”它只管这个程序的前面的宏定义里面有没有定义“x”这个宏，如果定义了x这个宏，那么，编译器会编译中间的…code…否则不直接忽视中间的…code…代码。  
另外 \#if defined(x)也可以取反，也就用 \#if !defined(x)

### \#ifdef的使用

\#ifdef的使用和\#if defined()的用法一致  
\#ifndef又和\#if !defined()的用法一致

最后强调两点：

1. 这几个宏定义只是决定代码块是否被编译
2. 别忘了\#endif
