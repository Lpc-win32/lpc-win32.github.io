---
layout:     post
title:      "C++11网络库：libttzhu正式启动"
subtitle:   " \"项目启动前吹的一波牛逼....\""
date:       2017-03-18 10:45
header-img: "img/post-bg-2015.jpg" 
author:     "pepperliu"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - c++
    - c++11
    - libttzhu
    - network lib
---

### 说在前面：为什么要写一套自我专属的C\+\+11网络库？

> 从事C\+\+网络服务端研发即将1年了。由于本人团队技术水平较为成熟，致使我掌握的知识较为宽广、但踩得坑较少。认清自己还是个“菜鸟”后，大胆地做了这样的决定--自己写一套网络开发库作为自己日常开发的利器。希望通过这次，能获得更大的启发！

> 在此感谢“陈硕-muduo库”为我提供了很多灵感。ttzhu初期会参考muduo的设计。删繁就简融入一些自己的需求。初步计划如下：

- 先不去考虑网络功能的实现，初步实现base功能（Date、time、Logger）等
- 每实现一个组件，会将组件的原理分享至该Blog
- 待大体功能实现完毕后，会开放此lib的源码

### Finnal

> 希望可以结交更多的同行，为我提供更多的建议
