---
layout:     post
title:      "Coral DSHT"
subtitle:   " \"Coral DSHT(内容分发网络系统)\""
date:       2018-10-29 19:34
header-img: "img/post-bg-blockchain.jpg" 
author:     "pepperliu"
catalog:      true
tags:
    - ethereum
    - blockchain
    - dht
    - dsht
    - coral
---

### 1. Coral DSHT（Distributed Sloppy hash table）

Coral协议是一套内容分发网络系统（CDN）。CDN的设计是为了避开互联网传输瓶颈，并且降低内容供应服务器的网络压力，使得内容能更快速，更稳定地传递给客户端。

CDN的基本思想是在网络部署一些节点服务器，并且建立一套虚拟网络。网络中节点服务器之间实时更新连接信息，延时信息，用户距离参数等，然后对用户的请求，重定向到最合适的节点服务器上。其好处如下：

- 通过节点服务器中转，用户访问网页的速度大大提高了
- 节点服务器会存储内容服务器的查询信息，降低了内容服务器的网络负载
- 内容服务器也可能出现暂时的离线，那么用户同样能通过节点服务器中的缓存读取

Coral DSHT则是Coral CDN最核心的一个部件。

[Kademlia](http://blog.lpc-win32.com/2018/10/12/kademlia-dht/)，使用的是XOR距离，即信息永远是存储在XOR距离最近的节点中。而这样并没有考虑实际网络的情况，例如节点之间的延时，数据的位置。这样会浪费大量网络带宽和存储空间。Coral解决了这个问题，不同于经典的DHT方式，coral首先对所有的节点评估连接情况，然后根据循环时间（Round\-Trip Time）划分为几个等级

Coral DSHT适用于软状态的键值对检索，也就是同一个Key可能会保存多个Value。这种机制能把给定的Key映射到网络中的Coral服务器地址。

- 使用DSHT来查询距离用户较近的剧名服务器
- 查询拥有特定网站缓存信息的服务器
- 定位周围的节点来最小化请求延时

### 2. 索引机制和分层

每一个Coral节点根据他们的延时特性放在不同的DSHT中。

在Coral中，将DSHT分成了三层，Level2对应两两延迟小于20ms，Level1对应两两延时小于60ms，Level0对应其他的全部节点。Coral在询问是，会优先请求等级更高，响应时间更短的节点。如果失败了，才会在下一级节点中请求。这样的设计不但降低了查询的延时，也更可能优先获得临近节点的返回数据

### 3. 基于键值对的路由层

Coral的键值对与Kademlia一样，都是SHA\-1哈希计算出来的，有160bit。每个节点的ID是由其IP地址，通过SHA\-1运算得到。节点的远近计算方法和路由方法与[Kademlia](http://blog.lpc-win32.com/2018/10/12/kademlia-dht/)是一致的

### 4. Sloppy存储

在Kademlia协议中，数据会直接保存到XOR更近的节点。但实际情况是，如果某些数据非常流程，有其他节点也会大量查询，会因此造成拥塞，我们称为Hot\-Spot。同时，对于一个缓存键值对存储了过多的值，我们称为Tree\-saturation。Sloppy存储就是希望规避这两种情况发生

每一个Coral节点定义有两种异常状态:

- Full，在当前节点R，已经有L个（Key\/Value）对使得Key\=k，并且这L个键值对的生存周期都大于新值的1\/2
- Loaded，对于给定的Key\=k，在过去的一分钟里已经收到超过beta次请求

Coral执行存储分为两步进行：

第一步是前向查询，Coral会持续迭代查找距离Key更近的节点ID，每一个节点返回两个信息，1\.该节点是否loaded，2\.对于该Key，这一节点存储有几个Value，每一个Value的实效时间是多长。客户端根据信息决定这些节点是否可以存储新的值

第二步为反向查询，客户端从第一步得到了可存放的节点列表，按照距离Key从近到远的顺序，依次尝试添加Key\/Value到这些节点