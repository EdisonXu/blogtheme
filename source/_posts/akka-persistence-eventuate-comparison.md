---
title: (译)Akka Persistence和Eventuate的对比
date: 2017-01-22 17:23:53
tags:
- eventsourcing
- eventuate
- akka
---
> 在实现微服务架构中，遇到了分布式事务的问题。Event-sourcing和CQRS是一个比较适合微服务的解决方案。在学习过程中，遇到了这篇文章，觉得很不错，特地翻译给大家。本文翻译自：[A comparison of Akka Persistence with Eventuate](http://krasserm.github.io/2015/05/25/akka-persistence-eventuate-comparison/)

Akka Persistence和Eventuate都是Scala写的，基于Akka的event-sourcing和CQRS工具，以不同的方式实现分布式系统方案。关于这两个工具得详情，请参见他们自己的在线文档。

我是Akka Persistence和Eventuate的原作者，目前主要关注在Eventuate的开发实现。当然，我的意见肯定会带有偏见;) 言归正传，如果我出了什么大错，请一定一定告之我。

## Command side
在Akka Persistence中，command这边(CQRS中的C)是由`PersistentActor`s(PAs)来实现的，而Eventuate是由`EventSourcedActor`s(EAs)来实现的。他们的内部状态代表了应用的写入模型。

PAs和EAs根据写入模型来对新的command进行校验，如果校验成功，则生成并持久化一条/多条后续会被handle来更新内部状态的事件。当crash或正常的应用重启，会根据event log中
