---
layout:     post
title:      "linux command: nc"
subtitle:   " \"netcat..网络界的瑞士军刀\""
date:       2017-04-20 23:28
header-img: "img/post-bg-unix-linux.jpg"
author:     "pepperliu"
catalog:      true
tags:
    - linux
    - cmd
    - command
    - nettools
    - network
---

> 今天工作中遇到了一个问题，业务方在接入我们的容器平台有一个这样的需求，需要提供一组类似于xshell、SecureCRT所支持的rz，sz这样的功能，将生产环境可能产生的Coredump文件copy至本地环境进行问题的分析。由于容器环境所提供的访问终端是web组件，因此rz，sz这样的基于串口传输的命令自然是行不通的。因此我对此有两个计划。

1. 最先能够想到的就是scp这样的命令了，但是scp不能满足线上环境堡垒机缺失密码等问题。
1. 基于Python快捷的实现一套HttpServer，暴露出一个端口，访问此端口即下载Coredump（会有额外的开发成本，虽然Python写一个这样的tool及其方便）
2. 使用nc--netcat命令完成这样的功能

### 问题：什么是nc？

nc是netcat的简写，有着网络界的瑞士军刀美誉。因为它短小精悍、功能实用，被设计为一个简单、可靠的网络工具

何来简易之说？功能强大，代码短小。1.8.4版本仅有25k

### nc的作用

1. 实现任意TCP/UDP端口的侦听，nc可以作为server以TCP或UDP方式侦听指定端口
2. 端口的扫描，nc可以作为client发起TCP或UDP连接
3. 机器之间传输文件
4. 机器之间网络测速

### ns完成Coredump传输操作

```
两台机器：线上环境1.1.1.1与个人工作机器192.168.0.1(个人工作机器可访问到线上环境机器)

线上环境1.1.1.1配置：
nc -l 18888 < core
(阻塞，等待客户端拉取)

个人工作机器：
nc 1.1.1.1 18888 > core
(拉取结束后，线上环境1.1.1.1的nc命令执行完毕)
```

> nc就是这样一个简单暴力的网络工具，默认情况下的传输层协议为TCP。今天暂时分享至此，日后遇到相应问题再对此工具做更深层次讨论
