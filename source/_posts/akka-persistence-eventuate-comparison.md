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

PAs和EAs根据写入模型来对新的command进行校验，如果校验成功，则生成并持久化一条/多条后续会被handle来更新内部状态的事件。当crash或正常的应用重启,内部状态可以通过重演整个event log中已持久化的事件或从某一个snapshot开始重演，来恢复内部状态。PAs和EAs都支持至少送达一次到其他actor的机制, Akka Persistence提供了`AtleastOnceDelivery`来实现，而Eventuate则使用`ConfirmedDelivery`。

从这个角度来看，PAs和EAs非常相似。一个主要的区别是，PAs必须是单例，而EAs则可以是多分可复制的，并且可以并发对其修改。如果Akka Persistence意外地创建和更新了两个具有相同`persistenceId`的PA的实例，那么底层的event log将会被污染，要么是覆盖已有事件，要么是拼接进了彼此冲突的事件。Akka Persitence的event log是被设计为只容许一个并且不允许共享的event writer。

在Eventtuate中，EAs可以共享同一个event log。基于事先自定义的事件路由规则，一个EA发出的的事件可以被另一个EA消费。换而言之，EA之间可以通过交换同一个共享的事件来进行协作，例如不同类型的EA组成一个分布式业务流程，或者多地址下相同类型的EA之间因为重现和更新内部状态而触发的状态复制。这里的多地址甚至可以是全局分布的(globally distributed)。多地址间的状态复制是异步的，具有相当可靠性。

## Event Relations
在Akka Persistence中，每个PA产生的事件是有序的，然而不同PA产生的事件之间是没有任何关联性的。即使一个PA产生的事件是比另一个PA产生的事件早诞生，但是Akka Persistence是不记录这种先后顺序的。比如，PA<sub>1</sub>持久化了一个事件e<sub>1</sub>，然后发送了一个command给PA<sub>2</sub>，使得后者在处理该command时持久化了另一个事件e<sub>2</sub>，那么显然e<sub>1</sub>是先于e<sub>2</sub>的，但是系统本身无法通过对比e<sub>1</sub>和e<sub>2</sub>来决定他们之间的这种先后的关联性。

Eventuate额外跟踪记录了这种happened-before的关联性(潜在的因果关系)。例如，如果EA<sub>1</sub>持久化了事件e<sub>1</sub>，EA<sub>2</sub>因为消费了e<sub>1</sub>而产生了事件e<sub>2</sub>，那么e<sub>1</sub>比e<sub>2</sub>先发生的这种关联性会被记录下来。happen-before关联性是由[vector clocks](http://rbmhtechnology.github.io/eventuate/architecture.html#vector-clocks)来跟踪记录的，系统可以通过对比两个事件的vector timestamps来决定他们之间的关联性是先后发生的还是同时发生的。

跟踪记录事件间的happened-before关联是运行多份EA relica的前提。EA在消费来自于它的replica的事件时，必需要清楚它的内部状态的更新到底是先于该事件的，还是同时发生(可能产生冲突)。

如果最后一次状态更新先于准备消费的事件，那么该事件可被当作一个普通的更新来处理；但如果是同时产生的，那么该事件可能具有冲突性，必须做相应处理，比如，
* 如果EA内部状态是[CRDT](http://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)，则该冲突可以被自动解决(详见Eventuate的[operation-based CRDTs](http://rbmhtechnology.github.io/eventuate/user-guide.html#operation-based-crdts))
* 如果EA内部状态不是CRDT，Eventuate提供了进一步的方法来跟踪处理冲突。根据情况选择自动化或交互式方式。

## Event logs
前文提到过，Akka Persistence中，每一个PA有自己的event log。根据不同的存储后端，可冗余式的存于多个node上(比如为了保证高可用而采用的同步复制)，也可存于本地。不管是哪种方式，Akka Persistence都要求对event log的强一致性。

比如，当一个PA挂掉后，在另外一个node上恢复时，必须要保证能够按正确的顺序读取到所有之前写入的事件，否则这次恢复就是不完整的，会导致这个PA会覆写已存在的事件，或者把一些新事件直接拼到event log后面，而实际上新事件与之前的一些未读事件是冲突的。所以，只有支持强一致性的存储后端才能被Akka Persistence使用。

AKka Persistence的写可用性取决于底层的存储后端的写可用性。根据[CAP理论](http://en.wikipedia.org/wiki/CAP_theorem)，对于强一致性、分布式的存储后端，它的写可用性是有局限性的，所以，Akka Persistence的command side选择CAP中的CP。

这种限制导致Akka Persistence很难做到全局分布下应用的强一致性，并且完全的事件有序性还需要实现全局统一的协调处理。Eventuate在这点上做得要更好：它只要求在一个*location*上保持强一致性和事件的完全有序性。这个*location*可以是一个数据中心、一个(微)服务、分布式中的一个节点、单节点下的一个流程等。

单location的Eventuate应用与Akka Persistence应用具有相同的一致性模型。但是，Eventuate应用通常会有多个location。单个location所产生的事件会异步地、可靠地复制到其他location。Eventuate定义了跨location的事件复制，并维护了因果事件的存储顺序，不同location的存储后端之间并不直接通信，所以，不同location可以使用不同的存储后端。
