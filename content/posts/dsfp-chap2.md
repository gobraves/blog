---
title: Distributed system for fun and profit 第二章笔记
date: 2019-01-14 00:06:53
tags: ["分布式"]
---
## 序
more abstraction:
1. 从组成结构上，A具有的B都有，A并没有添加其他任何新的部分,但是对事物的描述更加准确和易于管理
2. 通过移除一些不重要的部分，使得概念更容易理解

那么，分布式系统的最小集描述是怎样的呢？怎样的系统会被认为是分布式的呢？

## System Model
一般情况下，分布式系统的运行情况是这样的：
- 程序同时地运行在独自的节点上
- 程序间通过网络通信，但可能因此导致不确定性事件和消息丢失
- 没有共享内存和共享锁

上述情景意味着：
1. 每个节点同时执行程序
2. 存储在本地的数据: 节点可以更快的访问它们的状态，但是任何关于全局状态的信息可能不准确（即过期）
3. 节点可以单独挂掉并且单独地从宕机中恢复
4. 消息可能会发生延迟或者丢失
5. 每个节点的时间可能会不一致

系统模型列举了与特定系统设计相关的许多假设。

系统模型就是关于实现分布式系统的环境和设施的一组假设。

这些假设包括：
1. 每个节点具有怎样的能力且它们会怎样挂掉
2. 如何操作节点间的通信链接和通信链接会怎样挂掉
3. 整个系统的特点

一个健壮的系统模型使用最不严格的假设（weakest assumptions）：针对这种系统模型的算法需要考虑各种不同的环境可能发生的错误，并容忍这种错误，因为这中模型要求很少（very few and very weak assumptions）

另一方面，也可以创建一个系统模型with strong assumptions

### nodes in system mode
节点作为一台可以计算和存储的主机：
1. 具有执行程序的能力
2. 可以将数据存储到内存中或者稳定的状态
3. 时钟

- 节点失败
  - 大多数crash-recovery failure model: nodes can only fail by crashing, and can recover after crashing at some later point.
  - 假定节点在任何任一情况下都可能执行错误 Byzantine fault tolerance

### communication links in system model

节点通过通信链路相互连接，并可以相互通信。
区分节点故障和网络分区

### Timing/ordering assumptions
现实中，由于物理因素的影响，信息可能并不是同时到达不同的节点。

假设有三个节点A, B, C Distance<pointB,pointC> < Distance<pointA,pointB> < Distance<pointA,pointC>, 当A 发送消息给B,C后，B发送消息给A,C, 因为距离原因，可能导致B在收到A的消息时，C还未收到；并且可能C先收到B的消息，后收到A的消息  
解决方案：
- synchronous system model
  - 使用锁来同步每个步骤
  - 每个过程有准确的时钟
  - 消息在最大传输延迟内接收

- asynchronicity system model 
  - 没有信息传输的延迟限制
  - 没有十分有效的锁

解决同步模型的问题更容易，但不现实。现实中网络容易出问题，消息延迟没有严格限制，实际中多是部分同步。

### 共识问题

两个系统属性：
1. 网络分区
2. 同步或者异步假设

通过两个不可能的结果对系统设计的最终选择产生影响

#### FLP

#### CAP

- Consistency: 所有节点在同一时刻数据相同
- Availability: 失灵的节点不会影响正常的节点继续后续的操作
- Partition tolerance: 尽管由于网络或者节点出现问题导致消息丢失，但系统仍然可以继续操作

three differ system types:
- CA
- CP
- AP

CA和CP都是使用强一致性模型

##### CA
要保持一致性还要保证可用性，但分区间无法确认分区链接是否能正常或者节点是否正常。因此需要禁止向该系统写入数据避免数据不一致

##### CP
只保留多数分区，并要求少数分区不可用。在降低一定可用性的情况下保证一致性。CP模型将网络分区并入故障模型中。

##### CAP结论
1. 分区容错是现代系统一个重要特性
2. 在具有分区容错下，强一致性和高可用性有冲突
  这个问题如何解决呢？
  1. 加强假设，即让系统不具有分区容错能力
  2. 弱化一致性，即不使用强一致性
  3. 放弃可用性
3. 强一致性与性能存在冲突
4. 在分区容错下，也不能放弃可用性，可以考虑使用非强一致性模型

一致性模型是数据存储为使用它的程序提供的保证。也是程序员和系统之间的契约，其中系统保证如果程序员遵循某些特定规则，数据存储上的操作结果将是可预测的

##### consistency model
- strong consistency model(可以保证数据一直一致（maintaining a single copy）)
  - linearizable consistency（操作生效的顺序与操作的顺序相同）
  - sequential consistency(operating order as order)
- weak consistency model
  - client-centric consistency models
  - causal consistency: strongest model available
  - eventual consistency models

强一致性模型可确保整个系统各个部分的操作顺序及最终展示的结果相同

> ACID consistency != CAP consistency != Oatmeal consistency

###### strong consistency model
- linearizable consistency model 操作顺序和操作结果产生的顺序相同
- sequential consistency model 在保证每个节点顺序一致情况下，操作顺序和操作结果产生的顺序不同

###### client-centric consistency model
1. 最终是什么时候？需要有明确的界限
2. 备份系统如何达到最终的一致性？返回固定值？还是返回最大时间戳时的值


