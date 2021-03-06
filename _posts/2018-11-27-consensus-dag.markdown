---
layout:     post
title:      "一致性算法 - DAG(Directed Acyclic Graph)"
subtitle:   " \"Directed Acyclic Graph\""
date:       2018-11-27 11:41
header-img: "img/post-bg-unix-linux.jpg"
author:     "pepperliu"
catalog:      true
tags:
    - 分布式
    - consensus
    - DAG
---

### 1. 什么是DAG

DAG：Directed Acyclic Graph，中文意为“有向无环图”

DAG原本是计算机领域一种常用数据结构，因为独特的拓扑结构所带来的优异特性，经常被用于处理动态规划、导航中寻求最短路径、数据压缩等多种算法场景

从结构上看，DAG是分布式的体系结构，而不是链式结构，DAG与链式结构的本质区别在于异步与同步通讯。有向无环图是一种存储数据的方式。“有向”指所有数据顺着同一方向存储；“无环”指数据结构间不构成循环。像条毛线织的围巾，可以一直编下去

### 2. DAG of block起源

最早在区块链中引入DAG概念作为共识算法是在2013年提出的GHOST协议，用于作为比特币的交易处理能力扩容解决方案

后来NXT社区有人提出用DAG的拓扑结构来存储区块，解决区块链的效率问题

区块链只有一条单链，打包出块无法并发执行。如果改变区块的链式存储结构，变成DAG的网状拓扑可以并发写入。在区块打包时间不变的情况下，网络中可以并行打包N个区块，网络中的交易就可以容纳N倍

此时DAG跟区块链的结合依旧停留在类似侧链的解决思路，交易打包可以并行在不同的分支链条进行，达到提升性能的目的。此时DAG还是有区块的概念

![DAG of block](http://blog.lpc-win32.com/img/2018-11-27/1.png)

2015年9月，Sergio Demian Lerner发表了《DagCoin: a cryptocurrency without blocks》一文，提出了DAG-Chain的概念，首次把DAG网络从区块打包这样粗粒度提升到了基于交易层面，但DagCoin本身是一篇论文，没有代码实现

DagCoin的思路，让每一笔交易都直接参与维护全网的交易顺序。交易发起后，直接广播全网，跳过打包区块阶段，达到所谓的Blockless。这样省去了打包交易出块的时间。如前文提到的，DAG最初跟区块链的结合就是为了解决效率问题，现在不用打包确认，交易发起后直接广播网络确认，理论上效率得到了质的飞跃。DAG进一步演变成了完全抛弃区块链的一种解决方案

2016年7月，基于Bitcointalk论坛公布的创世贴，IOTA横空出世，随后ByteBall也闪亮登场，IOTA和Byteball是头一次DAG网络真正技术实现，也是此领域最耀眼的领军者；此时，号称无块之链、独树一帜的DAG链家族雏形基本形成

区块链是每个区块记多笔交易，而DAG是每个区块存一笔交易，所以它们的本质相同。在IOTA白皮书里，把结扎在一起的交易称为缠结（Tangle）

DAG共识算法的诞生是为了解决区块链的效率问题，通过DAG拓扑结构存储交易区块，支持网络中并行打包出块，提高交易容纳量。之后DAG不断演化逐渐形成了 blockless 的发展方向。从数据结构来看，DAG模式是一种典型的Gossip算法，即本质上为异步通讯，带来的最大的问题是一致性不可控，并且网络传输数据量会随着节点的增加而大幅增加

DAG算法支持交易快速确认，低廉交易手续费，同时也剔除了矿工角色。但是目前来看安全性低于POW等机制，容易形成中心化，例如IOTA依赖validator，字节雪球则需要见证人节点

### 3. DAG的安全问题

#### 3.1 双花

DAG异步处理数据的特征导致攻击者可能利用节点间的信息差进行双花。具体来说，如果两个顶点间没有明确的父子关系，攻击者可以分别在只看到这两个顶点中的一个的不同节点处，对同一笔存款进行双花。这种双花只有在同时看到两个区块的节点处才能被检测到，并且只有在两个顶点重新汇合到一个新顶点时才能最终判定哪一笔是双花

#### 3.2 影子链攻击

DAG允许多重并行交易的特征，导致攻击者可能暗中生成一条影子链，并且时不时地将影子链跟主链进行对接以逃避检测算法。极端情况下，这条影子链有可能代替主链成为全网的共识

#### 3.3 DAG与区块链的区别

与区块链技术不同，DAG技术最大的特点是没有区块。在该网络中没有矿工的概念，其一致性由交易本身来维护；每笔交易发出时都需参考之前未确认的交易，并立刻广播至全网，以形成互有联系的数据网络。从某种意义上来说，DAG就像是并发式多线程区块链；把传统区块链一维单点的存储模式改变为，一个三维全网并行的复杂工作环境

### 4. DAG总结

在并行存储模式之下，随着交易量的增多，DAG网络的结构会越来越复杂。虽然如此，但是从一个节点出发还是能找到一根主链的，只要把所有主链捋顺，便能够顺着结构链追踪到“谁给谁转账”及“每个地址发生了什么交易”等等信息；从而处理交易过程中可能会出现的双花问题

总的来说，这种数据存储架构具有交易速度快、无须挖矿、手续费较低的特点，常用于解决交易验证、并发及交易处理速度等问题

DAG是面向未来的新一代区块链，从图论拓扑模型宏观看，从单链进化到树状和网状、从区块粒度细化到交易粒度、从单点跃迁到并发写入，这是区块链从容量到速度的一次革新
