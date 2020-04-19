---
layout: post
title: "每个人都应该知道的性能参数"
author: "Yang"
header-style: text
tags:
  - Performance Metric
  - 性能参数
  - 性能估算
---

假设你是系统开发人员，如何估算出你设计开发的系统的性能？可能很多人都没有事先估算过自己系统的性能，而是等到开发完成后进行性能压力测试。但是如果在架构设计上就有问题，等到开发完再去修改架构代价就太大了。那么有没有办法在设计完系统架构后就估算出这个架构的性能呢？或者说我们设计了几套系统架构，如何判断这几个架构的性能优劣？

这个问题的关键点是需要知道常见硬件的性能参数。

![](/img/in-post/2020-04-18-performance_estimate/post-hardware-metric.png)

如何使用这个表呢？我们举几个例子。



#### 例1 生成包含30个图片的页面

假设我们有个网站需要开发一个页面给用户返回30个图片，每个图片256KB。我们根据不同的设计方案估算一下每种方案的性能数据。

设计方案一：顺序从SATA磁盘读取图片

(磁盘寻道时间10ms + 磁盘读取时间 256KB / (30MB/s)) * 30次 = 560 ms

设计方案二：并发从SATA磁盘阵列读取图片

磁盘寻道时间10ms + 磁盘读取时间 256KB / (30MB/s) = 18 ms

设计方案三：顺序从内存读取图片

(内存访问时间100ns + 内存顺序读取时间 256 KB / (4GB/s)) * 30次 = 1.92 ms

可以看到，不同的方案，时间上的差别是巨大的，而这些数据是可以事先估算出来的。当然，程序实际运行时，可能受磁盘缓存，CPU缓存的影响，实际时间可能跟估算出来的有差别，但数量级上应该不会有大的出入。这样可以帮助我们在设计阶段选择出一个最优的方案，减少以后优化甚至推倒重来的时间。



#### 例2 Basic Paxos时间消耗

假设我们有个分布式系统需要使用Paxos作为一致性协议，同城双机房三副本部署，如何估算一致性协议模块的性能？

Basic Paxos流程如下：
![](/img/in-post/2020-04-18-performance_estimate/post-basic-paxos.png)


整个流程中，CPU，内存，网卡的开销和同城网络来回开销，磁盘开销相比可以忽略不计，所以主要考虑网络来回和磁盘开销。Proposer发起提案到value被选择，一共两个网络来回。每个网络来回中，有一个副本是同机房，时间开销0.1 ms，另一个同城跨机房1 ms。网络开销总共2 ms。Proposer和Acceptor至少各有一次磁盘写入操作，如果使用SATA磁盘，每次写入数据量比较少，主要开销是寻道时间，两次共20 ms，整个流程网络开销 2 ms + 磁盘开销 20 ms = 22 ms。如果使用SSD磁盘，每次0.1 ms，两次0.2 ms，整个流程网络开销2 ms + 磁盘开销0.2 ms = 2.2 ms。

从上面的分析可看出，整个流程的开销要么在网络上，要么在磁盘上，所以Paxos所有的工程实现都是在想方设法减少这两个操作的次数。



总之，熟悉硬件性能表，可以帮助我们选择合适的设计方案，并且快速发现设计中的性能瓶颈。



参考：

[大规模分布式存储系统：原理解析与架构实战](https://book.douban.com/subject/25723658/)

[Building Software Systems at Google and Lessons Learned](https://research.google.com/people/jeff/Stanford-DL-Nov-2010.pdf)

[Implementing Replicated Logs with Paxos](https://ongardie.net/static/raft/userstudy/paxos.pdf)

