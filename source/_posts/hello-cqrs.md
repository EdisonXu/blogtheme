---
title: CQRS&Event Souring系列（一）——Hello
date: 2017-03-12 12:57:52
tags:
- axon
- CQRS
- eventsourcing
---
> 在研究微服务的过程中，跨服务的逻辑处理，尤其是带有事务性需要统一commit或rollback的，是比较麻烦的。本系列记录了我在研究这一过程中的心得体会。本文主要就以下几个问题进行介绍：
> - 什么是EventSourcing？
> - 什么是CQRS？
> - 为什么CQRS+EventSourcing可以作为微服务中跨服务逻辑的解决方案？

## 什么是EventSourcing?
记录发生的所有事件，通过回溯的方式，获取事物当前的状态。
可以想象一个时光轴的实现，它记录了一个用户从Day1发生的所有事件，然后进行统计和展示。另一个形象的类比：银行ATM账户。一个银行账户创建时必然有一个余额，然后每一次存取，这个余额都会发生增减变化。用户看到的是余额，银行内部必然有一个流水账，余额与流水账的计算结果一定是一致的，并且可以通过这一方式找到任一时间节点的余额情况。

## 什么是CQRS?
**CQRS**全称是`Command Query Responsibility Segregation`，即命令查询职责分离。
这个
