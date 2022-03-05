---
layout: post
title: "复制"
author: "Yang"
header-style: text
tags:
  - DDIA
  - replication
---

- any list
{:toc}
最近在看[《Designing Data-Intensive Applications》](https://book.douban.com/subject/26197294/)，本文主要记录书中第五章《复制》的内容，加深自己的理解。

复制(Replication)是通过网络把一份数据保存在多个机器上。为什么需要复制数据？原因主要有三个：

1. 把数据放在离用户近的位置，减少网络延迟。
2. 某台机器崩溃后，其他具有相同数据的机器可以继续服务，增加可用性。
3. 把读请求分散在不停机器上，增加读吞吐量。

通常有三种复制方案：

- 主从复制(Leaders and Followers)
- 多主复制(Mult-Leader Replication)
- 无主复制(Leaderless Replication)



# 主从复制

主从复制的流程如下：

1. 指定某个副本作为主副本(Leader)。写数据时，请求必须先发到主副本。
2. 其他副本作为从副本(Follower)。主副本写了本地数据后，把数据的变化发给从副本，从副本更新本地数据。
3. 客户端读数据可以查询主副本或者从副本，但是写只能发给主副本。

示例图如下：

![图一](/img/in-post/2022-02-26-ddia-replication/leader-follower1.png)

## 同步复制和异步复制

主从复制中，主副本会把数据的变化发给从副本复制。这种主从复制有两种方式，分为同步(Synchronous)复制和异步(Asynchronous)复制。

![图二](/img/in-post/2022-02-26-ddia-replication/leader-follower2.png)

上图中，Leader有两个Follower 1和2。Follower1使用**同步复制**，Follower2使用**异步复制**。Leader收到用户的更新请求后，把数据的变化发给Follower1和Follower2。因为Follower1使用同步复制，Leader等待Follower1返回成功后才会给用户返回成功，而Follower2使用异步复制，Leader不会等待Follower2返回结果。

同步复制的优点在于follower上的数据是最新的。如果leader崩溃，follower可以继续服务，不会有数据丢失。缺点在于如果某个follower出故障没有响应，leader就会一直阻塞，导致整个系统无法工作。

异步复制不存在某个follower故障导致阻塞整个系统的问题。但是异步复制会存在数据丢失的问题，如果follower没有成功复制数据，leader先给用户返回成功，然后leader崩溃了，这时follower上并没有最新的数据。

权衡这两种情况，得出了半同步(semi-synchronous)的解决方案。半同步的方案是某一个follower使用同步复制，其他follower使用异步复制。如果同步的follower故障了，把剩余的一个follower从异步复制设置为同步方式。这样可以保证至少有一个副本上的数据是最新的。

## 添加新副本

主从复制方式中，如何添加新的副本？不能直接把已有副本的数据拷贝到新副本，因为客户端还在持续不断地写入新的请求。常用的方案如下：

1. 在leader上保存[全局一致性快照(snapshot)](https://yang.observer/2021/11/27/distributed-snapshots/)和快照时间点。
2. 把快照发到新副本上。
3. 新副本连到leader，让leader发送从快照时间点之后的所有数据变化历史。
4. 新副本处理完所有变化历史后数据就和leader保持一致了。

## 处理副本崩溃

**从副本故障**

follower可以在本地记录已经复制的历史数据位置，比如时间戳或者下标，崩溃后重启，因为有历史数据位置信息，可以向leader请求同步这个位置开始的数据变化。

**主副本故障**

主副本故障的处理相对麻烦一些，流程如下：

1. 检测leader故障。如果leader超过一定时间没有心跳包，认为leader出现故障。
2. 选出新leader。可以通过多数派选举或者指派的方式选出新leader。新leader应该是具有最新数据的副本。
3. 配置整个系统使用新leader。客户的需要把请求发给新leader，其他follower需要从新leader复制数据。如果旧leader恢复上线，整个系统应该拒绝承认旧leader，否则系统里面有两个leader，存在脑裂问题(split brain)。

其中包含了分布式系统的本质问题：节点失效，不可靠的网络以及一致性、持久性、可用性和延迟之间的权衡。

## 复制的实现

前面说了复制的时候，leader会把数据的变化发给follower，这个具体是怎么实现的？有四种方案。

**基于语句的复制**

以DB为例，这种方案中，leader记录客户端的sql语句，发给follower，follower执行每条语句。MySQL 5.1之前的版本使用了这种方式。

这种方案有几个问题：

- sql语句的结果可能不是固定的。比如now(), rand()等函数每次执行的结果可能不一样。
- sql语句使用自增长字段的值不同副本可能不一样。
- sql语句如果有副作用，比如触发器、存储过程等，这些副作用的结果在不同副本上可能不一样。

**Write-ahead log(WAL)**

在[《存储引擎漫话》](https://yang.observer/2021/12/04/ddia-storage-engine/)中介绍过WAL。WAL中按顺序保存了数据的修改。leader可以在写入WAL 后，把它发给follower，follower解析WAL，在本地处理数据的变化。PostgreSQL和Oracle使用了这种方式。

这种方案的缺点是WAL比较底层，里面可能包含哪个磁盘块的哪个字节被改变了，这样就让复制的过程和存储引擎耦合了，如果存储格式变了，leader和follower的程序版本不一样了，可能就无法处理。这可能会影响程序的升级，比如我们要升级整个系统的程序，可以先升级follower，然后把follower升级为leader，这样可以无缝升级整个系统。但是如果WAL格式无法兼容新旧版本就无法做到无缝升级。

**基于逻辑的复制**

为了解决WAL太过底层可能无法兼容的问题，可以使用逻辑(logical)日志，也就是不使用存储引擎用的WAL或者叫物理(physical)日志。这种逻辑日志也叫基于行的日志(row-based)。MySQL的binlog使用了这种方式。

- 插入时，逻辑日志包含插入的所有列数据
- 删除时，逻辑日志包含被删除行的唯一标识。
- 更新时，逻辑日志包含行的唯一标识和变化的数据。

这种方式解偶了存储引擎和复制，而且逻辑日志可以发给外部系统用于离线处理。

## 复制的延迟问题

主从复制中，写请求由leader处理，读请求由任何一个副本处理，这样可以把读请求分散在不同副本上，降低leader的负载，但是这会导致一致性的问题。在异步复制或者半同步复制的方案下，客户端发起写请求，leader给客户端返回成功后，客户端向follower发起读请求，这时leader的数据还没有复制到这个follower，这时就产生了一执行问题。常见的一致性问题有三类。

**读自己的写操作**

![图三](/img/in-post/2022-02-26-ddia-replication/lag1.png)

如图所示，异步复制下，用户先发送写请求，leader给用户返回成功，同时给两个follower发送同步请求，之后用户给Follower 2发送读请求，但这时leader的同步请求还没到Follower 2，这就是写后读不一致(read-after-write inconsistency)。常用的解决方案有：

- 从leader上读用户自己的数据，从follower上读其他人的数据。比如社交程序，一般自己的用户数据只能自己修改，别人的数据只能查看。这种情况下，如果是读自己的数据，可以发到leader，如果是读其他人的数据，可以发到follower。
- 网关程序记录每个用户数据的更新时间，如果当前时间在更新时间的1分钟内，把读请求发给leader，否则发给follower，也就是认为延迟最长是1分钟。也可以记录每个follower上的复制延迟，这样更精确。
- 客户端记录写的时间戳，发请求给follower的时候带上这个时间戳，如果follower发现本地数据的时间戳不够新，把请求转发给leader。这种方案需要注意用户在多个终端登录的情况，不同终端上需要同步相同的本地写请求的时间戳，而且要注意不同终端的网络连接可能不一样。

**单调读**

![图四](/img/in-post/2022-02-26-ddia-replication/lag2.png)

如图所示，User1234插入了一条数据，leader把数据同步给两个Follower，Follower1先收到，Follower2过了一段时间才收到。User2345先在Follower1上查询到了这条数据，然后又查询了一次，这次的请求落在了Follower2上，但是Follower2还没同步到这条数据，这就是单调读问题(Monotonic Read)，也就是读数据的时候，出现了数据回退的情况。

解决方案是同一个用户的读请求应该落在同一个副本上，可以通过对用户ID做哈希的方式来实现。

**一致性前缀读**

![图五](/img/in-post/2022-02-26-ddia-replication/lag3.png)

一致性前缀读(Consistent Prefix Read)也叫因果一致性，如图所示，Mr. Poons和Mrs. Cake有如下对话。

```
Mr. Poons
    How far into the future can you see, Mrs. Cake?
Mrs. Cake
    About ten seconds usually, Mr. Poons.
```

Mr. Poons先问了一个问题，Mrs. Cake回答了问题。Mr. Poons和Mrs. Cake在不同的分区(partition)，每个分区都有自己的leader。此时有第三个观察者在看他们的对话，由于网络延迟，Partition2先把回答发给了观察者，之后Partition1才把问题发给了观察者，这违反了因果关系，应该先看到问题再看到回答。这个问题是因为不同用户在不同的分区，如果大家都在同一个分区不会有这个问题。在[计算机的时钟（三）：向量时钟](https://yang.observer/2020/09/12/vector-clock/#%E5%90%91%E9%87%8F%E6%97%B6%E9%92%9F%E7%9A%84%E5%BA%94%E7%94%A8)中也讨论过相似的例子。一种解决方案是文中提到的向量时钟，这里不再重复描述。

# 多主复制

上面介绍的主从复制只有一个leader，另外一种方案是多主复制(multi-leader replication)，也就是系统中有多个leader。常用的多主复制场景如下。

**多中心操作**

为了提升系统的可用性（比如灾备），或者降低用户访问延迟（比如全球部署），系统可能会部署在多个数据中心。

![图六](/img/in-post/2022-02-26-ddia-replication/multi-leader1.png)

**客户端离线操作**

比如一个日历程序，可能会在手机，笔记本和台式机上访问。如果在离线的时候修改了日历内容，下次联网的时候需要把数据同步到服务器，这种情况下每个设备都是一个leader或者说是一个数据中心。

**协作编辑**

在线实时协作编辑软件，比如Google Docs，每个用户本地浏览器中编辑，然后数据同步到服务器上，这个和客户端离线操作比较类似。

## 写冲突处理

多主复制最大的一个问题是写冲突，多个leader同时接受了同一个id的写入请求就会造成写冲突。

![图七](/img/in-post/2022-02-26-ddia-replication/multi-leader2.png)

如图所示，两个用户连接了两个不同的leader，同时修改了同一个id的数据，造成了冲突。

处理冲突的一种方式是避免冲突(Conflict avoidance)。比如不同ID的写请求路由到不同的leader上。但是这样可能失去了多主复制的优势，使用多主的目的是为了提升可用性或者降低延迟，如果把数据分配在不同leader上，如果某个leader崩溃，多主复制就没有用了。

另外一种方式是一致性状态收敛(converging toward a consistent state)。在主从复制中，写操作是顺序发生的，不会产生冲突。多主复制中，由于写操作可能同时发生，需要解决冲突，让数据收敛到一致的状态。常用的方案有：

- last write wins(LWW)，给每个写请求分配唯一ID，发生冲突时，保留ID最大的写操作，丢弃其他并发写操作。
- 给每个副本分配唯一ID，发生冲突时，保留ID最大的副本上的写操作，丢弃其他副本的写操作。
- 同时写入冲突的数据，并且记录冲突状态，之后由用户来解决冲突(比如弹窗让用户选择数据)。

## 多主复制拓扑

多主复制中leader之间需要复制数据，常用的复制拓扑结构有三种：环形，星型和全连接。

![图八](/img/in-post/2022-02-26-ddia-replication/multi-leader3.png)

对于环形和星型，可能出现某个副本故障导致整个复制失败的情况。

对于全连接，可能会出现上面介绍过的因果一致性问题。如下图所示：

![图九](/img/in-post/2022-02-26-ddia-replication/multi-leader4.png)

# 无主复制

主从复制和多主复制这两种方案中，客户端的写请求都是发给某一个leader，只由这个leader处理这个请求，然后再复制到其他副本。其实第三种方案叫做无主复制(leaderless replication)，这种方案中客户端把写请求同时发给所有副本，副本之间没有主从之分。

![图十](/img/in-post/2022-02-26-ddia-replication/leaderless1.png)

如图所示，客户端写入时，把请求同时发给3个副本，收到2个副本的成功响应，第3个副本故障了不影响这次的请求，只要有多数派返回成功就认为这次写入成功。那第3个副本重启后，缺少了这次写请求的数据，怎么同步这次数据的改动？有两种办法。

- 读修复：客户端读数据的时候，向所有副本发请求，这时副本1和副本2返回了最新的数据，版本号是7，而副本3返回的是旧数据，版本号是6，这时客户端向副本3发一次写请求，更新副本3上的数据。
- 后台进程修复：有个后台进程定时扫描副本之间数据的差异，同步最新的数据。这个方案的好处时可以同步所有数据，读修复方案只能修复热数据，冷数据可能要很久才有机会修复。

选择写或者读的成功数量时，有个条件: ```w + r > n```，其中n为总副本个数，w为写请求至少收到成功响应的个数，r为读请求至少收到成功响应的个数。只要满足```w + r > n```就可保证一定能读到最新的数据。一般n会设置成奇数，```w = r = (n + 1) / 2```，这样可以容忍 n / 2 个副本故障。不过根据业务的情况，可以修改w和r。比如对于写很少，读很多的业务，可以设置```w = n, r = 1```，不过这样不能容忍写入时有副本故障。

无主复制适合对可用性和低延迟要求比较高，对一致性或者读到旧数据要求不高的场景。

## 处理并发写

无主复制最大的一个问题是并发写冲突。举个例子：

![图十一](/img/in-post/2022-02-26-ddia-replication/leaderless2.png)

客户端A和B分别向系统发送写请求，A 设置X = A, B 设置 X = B。

- Node 1 只收到了A请求。
- Node 2 先收到A，后收到B。
- Node 3 先收到B，后收到A。

这时系统数据不一致，读取X的时候，有两个节点返回A，一个节点返回B。我们需要实现最终一致性，上面介绍了多主复制方案下写冲突的处理方法，这里也可以使用last write win(LWW)方法来处理。

另外一种方法是构造"happend befefore"关系。举个例子，两个用户同时往同一个购物车添加商品。如下图所示：

![图十二](/img/in-post/2022-02-26-ddia-replication/leaderless3.png)

1. Client 1第一次把milk加到购物车中，没有带版本号，服务器成功处理，返回更新后的数据和当前版本号1。
2. Client 2第一次把eggs加到购物车中，没有带版本号，服务器发现数据版本号已经是1了，而且客户端没有带版本号过来，服务器把[eggs]和[milk]分别作为两个版本保存，升级版本号为2，把[eggs]和[milk]和版本号返回Client 2。
3. Client 1想把flour加到购物车，在第1步的返回结果中已经知道版本号是1，所以发送购物车内容为[milk, flour]，带上版本号1。服务器发现Client 1带过来的版本号是1，说明Client 1是在版本号1基础上做的写入，可以安全把版本号1的内容替换为[milk, flour]，而[eggs]属于版本2，大于Client 1带来的版本号1，所以是并发操作，需要分开存储，[milk, flour]版本号升级为3，[egg]的版本号还是为2。把[milk, flour]和[eggs]，以及最新版本号3返回Client 1。
4. Client 2想把ham加到购物车，在第2步的返回结果中已经知道版本号是2，购物车中已经有[milk]和[eggs]，它需要做合并，同时把ham加入，发送内容为[eggs, milk, ham]，带上版本号2。服务发现Client 2带过来的版本号是2，说明Client 2岁在版本号2基础上做的写入，可以安全把版本号2点内容替换为[eggs, milk, ham]，而[milk, flour]的版本号是3，大于Client 2带来的版本号2，所以是并发操作，需要分开存储，[eggs, milk, ham]版本号升级为4，[milk, flour]的版本还还是3。把[eggs, milk, ham]和[milk, flour]，以及最新版本号4返回Client 2。
5. Client 1想把bacon加到购物车，在第3步返回结果中已经知道版本号是3，购物车中已经有[milk, flour]和[egg]，它需要做合并，同时把bacon加入，发送内容为[milk, flour, eggs, bacon]，带上版本号3。服务替换版本号3的内容为[milk, flour, eggs, bacon]，升级版本号为5，版本号4的内容还是[milk, eggs, ham]，把[milk, flour, eggs, bacon]和[eggs, milk, ham]以及最新版本号5返回给Client 1。

整个过程中服务器收到了客户端带来的版本号，可以安全的替换小于等于这个版本号的数据，这属于"happened before"，需要保留比这个版本号的的数据，这属于concurrent。下图描述了整个过程中的"happened before"和concurrent关系。

![图十三](/img/in-post/2022-02-26-ddia-replication/leaderless4.png)

对于多副本的情况，每个副本需要保存自己的版本号，同时需要记录其他副本的版本号，这个算法和[向量时钟](https://yang.observer/2020/09/12/vector-clock/#%E5%90%91%E9%87%8F%E6%97%B6%E9%92%9F%E7%9A%84%E5%BA%94%E7%94%A8)有点像，叫做版本向量(version vector)，这里不做过多介绍。



# 参考

[Designing Data-Intensive Applications](https://book.douban.com/subject/26197294/)

