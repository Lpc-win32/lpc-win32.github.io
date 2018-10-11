---
layout:     post
title:      "以太坊源码分析-以太坊框架解析"
subtitle:   " \"先了解一下以太坊整体框架\""
date:       2018-10-11 14:01
header-img: "img/post-bg-ethereum.jpg"
author:     "pepperliu"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - ethereum
    - blockchain
---

### 以太坊框架总览

> 先来看看整体大纲，细节后续依次分析

**以太坊框架可分为如下几个层面**：

- 应用层：钱包客户端（可发起交易操作），用户能与之直接产生交互。
- 合约层：EVM（以太坊虚拟机），计算消耗gas\_limit
- 激励层|共识层：POW共识算法（挖矿），txpool交易池，数据校验
- 网络层：P2P，TCP传输数据、UDP发现节点（广播+NAT穿透）
- 存储层：levelDB（数据存储引擎）

以太坊转账时消费的是gas，旷工会收获gas与奖励两种出处的价值

EVM虚拟机的作用：解析solidity语言编写的智能合约、智能合约账户

txpool交易池：包含queue与pending两个等待队列。当我们发起一笔交易时，交易首先会存放于queue中，条件满足后再放至pending等待队列中，而pending就等待着挖矿（根据Gas_Limit最大值在交易池中取，因此谁给的gas越多，越容易被取到，然后POW挖矿）

![image](http://blog.lpc-win32.com/img/2018-10-11/ethereum_global.png)