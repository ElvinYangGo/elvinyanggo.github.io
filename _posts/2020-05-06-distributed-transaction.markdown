---
layout: post
title: "从单机事务到分布式事务"
author: "Yang"
header-style: text
tags:
  - 事务
  - 单机事务
  - redo log
  - undo log
  - MVCC
  - 分布式事务
  - 两阶段提交
  - 三阶段提交
  - XA
  - TCC
  - Saga
  - MQ
---

[toc]

最近在研究分布式数据库相关的技术，对于数据库来说，不管是单机数据库还是分布式数据库，事务都是一个绕不去的坎。不光是数据库，对于微服务架构，不同服务之间也会涉及到分布式事务的处理。本文先介绍事务的基本概念和原理，然后介绍单机事务的实现方案，最后介绍分布式事务的实现方案。

学习事务的过程中参考了不少文章，这些文章比本文更有价值，结尾有这些文章的地址。



## 什么是事务

描述原理和实现之前，我们首先需要了解究竟什么是事务。就像开会，先划定议题，介绍背景知识，才能更高效的讨论。维基百科中对[事务](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BA%8B%E5%8A%A1)的描述如下：

>数据库事务通常包含了一个序列的对数据库的读/写操作。包含有以下两个目的：
>
>1. 为数据库操作序列提供了一个从失败中恢复到正常状态的方法，同时提供了数据库即使在异常状态下仍能保持一致性的方法。
>
>2. 当多个应用程序在并发访问数据库时，可以在这些应用程序之间提供一个隔离方法，以防止彼此的操作互相干扰。
>
>当事务被提交给了数据库管理系统（DBMS），则DBMS需要确保该事务中的所有操作都成功完成且其结果被永久保存在数据库中，如果事务中有的操作没有成功完成，则事务中的所有操作都需要回滚，回到事务执行前的状态；同时，该事务对数据库或者其他事务的执行无影响，所有的事务都好像在独立的运行。

从上面的描述中可以看出，事务有几个重要的特性，一是原子性，要么全部成功，要么全部回滚，二是一致性，即使在异常状态也能保持数据的一致，三是隔离性，一个事务的执行不会影响其他事务。要实现这几个特性，是数据库事务的主要难点。其实准确地说，是有四个特性，也就是常说的ACID。维基百科对[ACID](https://zh.wikipedia.org/wiki/ACID)的描述如下：

>Atomicity（原子性）：一个事务（Transaction）中的所有操作，或者全部完成，或者全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。即，事务不可分割、不可约简。
>Consistency（一致性）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设约束、触发器、级联回滚等。
>Isolation（隔离性）：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括未提交读（Read Uncommitted）、提交读（Read Committed）、可重复读（Repeatable Read）和串行化（Serializable）。
>Durability（持久性）：事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

注意传统数据库的一致性是指数据库本身的完整性，主要是指数据库的约束条件没有被破坏，比如设置约束某个字段值不能小于0，事务会保证始终符合这个约束条件。数据库中的一致性和CAP中的一致性不是一个概念。CAP中的一致性是指分布式系统中不同副本上的数据是一致的。到了分布式事务中，和传统数据库相比，对于一致性的定义可以稍作修改。对于分布式事务，各个事务参与者对于事务是否成功提交达成一致，这个可以认为是分布式事务的一致性。



## 单机事务

传统单机情况下，数据库是怎么实现原子性和隔离性的呢？以MySQL数据库为例，原子性通过undo日志实现，持久性通过redo日志实现，隔离性通过锁、MVCC(Multi Version Concurrency Control)和undo日志实现。下面分别做介绍。

### redo日志

介绍redo日志之前，需要先简单说一下MySQL的数据管理方式。MySQL中的数据并不是每次都是从磁盘中读取的，而是在内存中有一个缓存Buffer Pool，用于存放磁盘中的热点数据页。读取数据时先从Buffer Pool中读取，如果没有的话再从磁盘中读取，然后保存在Buffer Pool中。写入数据时也是先写到Buffer Pool，然后再把Buffer Pool中的数据定期写入磁盘。

虽然Buffer Pool提高了MySQL效率，但是会导致一个问题，如果在写入磁盘前，MySQL宕机了，Buffer Pool中的还没写盘的数据就丢失了，所以MySQL设计了redo日志来解决这个问题。

redo日志由内存中的redo log buffer和redo日志文件组成。修改数据时，先写redo日志添加到内存中的redo log buffer，然后修改Buffer Pool中的数据。提交这次事务时，可以选择是否将redo log buffer中的日志刷新到磁盘。用户可以通过innodb_flush_log_at_trx_commit参数来控制写盘时机，有三种取值：

- 0，事务提交时不写盘，由线程每秒写盘一次。
- 1，事务提交时调用fsync强制写盘。
- 2，事务提交时写入文件系统缓存，由操作系统决定何时将缓存写入磁盘。

redo日志并不一定是提交才会写盘，如果innodb_flush_log_at_trx_commit设置为0，即使还没提交，也可能写盘。

如果每次修改数据都需要写redo日志到磁盘，那为什么不把Buffer Pool中的数据直接写磁盘呢？原因主要有两个：

- 直接刷新数据是一个随机IO，每次修改的数据在不同的数据页中，而redo日志是连续的，写盘是顺序IO
- 直接刷新数据是以数据页为单位，MySQL默认是16KB，即使修改的数据只有一个字节也需要写16KB。而redo日志只包含修改的数据，数据量要少很多。

MySQL中redo日志以块（block）为单位存储，每块的大小为512B，格式如下：

![](/img/in-post/distributed-transaction/post-redo-block.png)

每个block由12字节头部log block header，492字节日志内容log block和8字节尾部log block tailer组成。

log block日志内容中保存是具体redo日志，格式如下：

![](/img/in-post/distributed-transaction/post-redo-log-body.png)

- redo_log_type: redo日志类型
- space: 表空间ID
- page_no: 页偏移量
- redo log body: 根据日志类型的不同，存储的内容格式也不一样

在描述数据恢复过程之前，还需要介绍一下MySQL中有个Log Sequence Number（LSN），8个字节，是一个递增的值，表示当前redo日志总共有多少个字节。LSN保存在redo日志文件和每个数据页的头部。写redo文件时，MySQL会把当前的LSN一起写入文件，然后修改内存当前数据页的LSN。等到数据页写盘时，LSN也会一起保存在该数据页对应的磁盘中，而且当前LSN值会写入数据文件ibdata的第一个page中作为整个数据文件的checkpoint。

MySQL启动时，首先检查当前redo日志文件中的LSN和数据文件中的checkpoint对应的LSN，如果两个一样，说明没有数据丢失。如果checkpoint LSN小于redo LSN，说明有数据丢失，从checkpoint LSN开始，遍历每个redo日志，找到对应的数据页，如果数据页的LSN小于redo日志中的LSN，需要对这一页进行数据恢复。理论上redo日志中所有在checkpoint之后的事务都需要恢复，为什么这里还要比较每一页的LSN？这是因为MySQL刷脏页时，是先把所有脏页写入磁盘，最后再写入checkpoint LSN。有可能脏页已经写入磁盘，但是在写入checkpoint LSN前宕机，这就需要在恢复事务时判断数据页中的LSN，避免重复恢复。这里只介绍了redo日志在数据恢复时的使用，实际上还要结合binlog一起使用，这就更复杂了，不详细展开。

![](/img/in-post/distributed-transaction/post-redo-recovery.png)

### undo日志

undo日志用于事务的回滚和MVCC，分别对应原子性和隔离性。MySQL中修改数据时，并不是简单地在当前数据上修改，而是先把修改前的数据保存在undo log中，然后修改当前数据并且在当前数据中增加一个指针指向修改前的数据。如下图所示，undo日志组成了一个链表：

![](/img/in-post/distributed-transaction/post-undo-history.png)

图中undo列表由三个SQL操作组成，左上角为当前记录的内容，第二个方块是最后一条SQL语句对应的undo日志，日志中保存了事务ID(TRX_ID)和修改前字段的内容("B")，最后一个方块是insert语句对应的undo日志。

根据不同的操作类型，undo日志的格式不一样，下面以update操作为例介绍对应的undo日志格式。

![](/img/in-post/distributed-transaction/post-undo-log-format.png)

- next: 2B，表示下一条undo日志位置
- type_cmpl: 1B，表示undo日志类型
- undo_no: 序号，用来区分一个事务中多个undo日志的顺序
- table_id: 表ID
- info_bits: 一些标记位
- DATA_TRX_ID: 这次修改对应的事务ID
- DATA_ROLL_PTR: 回滚指针，记录当前数据上一个版本在回滚段中的位置
- update vector: 表示修改的数据
- start: 表示上一条undo日志位置

除了上面介绍的字段，undo日志还有一个undo header头部信息，其中一个字段是TRX_UNDO_STATE，表示undo日志的状态。取值有下面几个：

- TRX_UNDO_ACTIVE: 初使状态
- TRX_UNDO_CACHED: 
- TRX_UNDO_TO_FREE: 可以释放
- TRX_UNDO_TO_PURGE: 可以清理
- TRX_UNDO_PREPARED: 准备状态，还未提交

undo日志在事务未提交前是TRX_UNDO_PREPARED状态，事务提交后，根据不同的操作类型转换成TRX_UNDO_CACHED，TRX_UNDO_TO_FREE或者TRX_UNDO_TO_PURGE状态，表示满足一定条件后可以释放，事务如果需要回滚的话，必须是TRX_UNDO_ACTIVE或者TRX_UNDO_PREPARED状态。此时从undo日志中取出上一次的数据作为当前数据的值。需要说明的是写undo日志本身也会产生相应的redo日志。

### MVCC

MVCC的全称是Multi Version Concurrency Control多版本并发控制。它的作用是解决不同事务之间并发执行的时候，数据修改的隔离性问题。数据库有四种隔离级别，由低到高如下：

1. 未提交读(READ UNCOMMITED)：允许读取其他事务未提交的修改
2. 已提交读(READ COMMITED)：只能读取其他事务已提交的修改
3. 可重复读(REPEATABLE READ)：同一个事务内，多次读取操作得到的每个数据行的内容是一样的
4. 可串行化(SERIALIZABLE)：事务执行不受其他事务的影响，就像各个事务之间是按顺序执行的

不同的隔离级别可以解决不同级别的读问题，如下：

| 隔离级别 |   脏读   | 不可重复读 |   幻读   |
| :------: | :------: | :--------: | :------: |
| 未提交读 | 可能发生 |  可能发生  | 可能发生 |
|  提交读  |    -     |  可能发生  | 可能发生 |
| 可重复读 |    -     |     -      | 可能发生 |
| 可序列化 |    -     |     -      |    -     |

脏读、不可重复读、幻读的解释如下：

#### 脏读

当一个事务允许读取另外一个事务修改但未提交的数据时，就可能发生脏读。

举个例子：

| 事务 1                                                       | 事务 2                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| /* Query 1 \*/<br/> SELECT age FROM users WHERE id = 1;<br/> /* will read 20 */ |                                                              |
|                                                              | /* Query 2 \*/<br/> UPDATE users SET age = 21 WHERE id = 1;<br/> /* No commit here */ |
| /* Query 1 \*/<br/> SELECT age FROM users WHERE id = 1;<br/> /* will read 21 */ |                                                              |
|                                                              | ROLLBACK;<br/> /* lock-based DIRTY READ */                   |

#### 不可重复读

在一次事务中，当一行数据获取两遍得到不同的结果表示发生了“不可重复读”.

| 事务 1                                                       | 事务 2                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| /* Query 1 \*/<br/> SELECT * FROM users WHERE id = 1;        |                                                              |
|                                                              | /* Query 2 \*/<br/> UPDATE users SET age = 21 WHERE id = 1;<br/> COMMIT;<br/> /* in multiversion concurrency control, <br/>or lock-based READ COMMITTED */ |
| /* Query 1 \*/<br/> SELECT * FROM users WHERE id = 1;<br/> COMMIT;<br/> /* lock-based REPEATABLE READ */ |                                                              |

#### 幻读

在事务执行过程中，当两个完全相同的查询语句执行得到不同的结果集。这种现象称为“幻读（phantom read）”

| 事务 1                                                       | 事务 2                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| /* Query 1 \*/<br/> SELECT * FROM users WHERE age BETWEEN 10 AND 30; |                                                              |
|                                                              | /* Query 2 \*/<br/> INSERT INTO users VALUES ( 3, 'Bob', 27 );<br/> COMMIT; |
| /* Query 1 \*/<br/> SELECT * FROM users WHERE age BETWEEN 10 AND 30; |                                                              |

对于隔离性，一种方法是基于锁。这种方案的问题是性能可能比较低，尤其是读操作也要加锁的时候。另一种方案是MVCC。MVCC用于替代读锁，写依旧需要加锁。MVCC和undo日志是如何支持不同的隔离级别，解决读问题的呢？

对于未提交读，只要读取记录当前版本的值就行了。

对于已提交读，在执行事务中每个查询语句的时候，MySQL会生成一个叫做视图read view的数据结构，包含以下内容：

- m_ids: 所有正在执行的事务ID，这些事务还未提交
- min_trx_id: 生成read view时正在执行的最小事务ID
- max_trx_id: 生成read view时系统应该分配的下一个事务ID
- creator_trx_id: 生成read view时事务本身的ID

访问数据时，根据以下规则判断某个版本的数据是否可见：

1. 如果当前版本数据的事务ID和creator_trx_id相同，说明是当前事务修改的记录，此时该版本数据可见。
2. 如果当前版本数据的事务ID小于min_trx_id，说明是已经提交的事务作的修改，该版本数据可见。
3. 如果当前版本数据的事务ID大于等于max_trx_id，说明是该版本的事务是在read view创建之后生成的，该版本数据不可见
4. 如果当前版本数据的事务ID在min_trx_id和max_trx_id之间，并且在m_ids内，该版本数据不可见，如果不在m_ids内，该版本数据可见。

如果当前版本数据不可见，使用前面介绍的undo日志，根据ROLL PTR回滚指针找到上一个版本的数据，判断上一个版本的数据是否可见，如果不可见，沿着undo日志链表找到符合条件的数据版本。

对于可重复读，和已提交读的差别在于已提交读是在事务中每条查询语句执行的时候生成read view，而可重复读是在事务一开始的时候就生成read view。

MVCC解决了脏读、不可重复读问题，以及部分幻读问题。为什么说是部分？对于前面幻读举的例子，事务1的两条select都是读的同一版本的数据，因为事务2插入的数据版本号不符合事务1的读取范围，所以不会读到，这种情况的幻读MVCC可以处理。但是另一种幻读则处理不了，这涉及到快照读和当前读的概念。快照读就是前面介绍的使用undo日志来选择一个合适的版本来读取，select操作使用这种方式。而insert、update和delete则使用当前读，对于这几个修改操作，必须使用最新的数据进行修改。举个例子：

| 事务 1                                                       | 事务 2                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| /* Query 1 \*/<br/> SELECT * FROM users WHERE age BETWEEN 10 AND 30; |                                                              |
|                                                              | /* Query 2 \*/<br/> INSERT INTO users VALUES ( 3, 'Bob', 27 );<br/> COMMIT; |
| /* Query 1 \*/<br/> UPDATE users SET name = 'Tom' where id = 3; |                                                              |
| /* Query 1 \*/<br/> SELECT * FROM users WHERE age BETWEEN 10 AND 30; |                                                              |

事务1查询读到两条数据，事务2插入id为3的记录后，事务1执行更新操作，使用当前读获取的最新数据，将id为3的记录名字改成了Tom，然后事务1执行第二次查询，这时是可以读到id为3的数据的，两次select读到的数据不一样。因为事务1进行update操作，数据的版本号是事务1自己的事务ID，所以第二次select能读到id为3的记录。是不是很复杂？

#### 事务流程

介绍完了redo日志，undo日志后，我们完整描述一下事务流程。

准备阶段：

1. 分配事务ID
2. 如果隔离级别是REPEATABLE READ，创建read view
3. 分配undo日志，把修改之前的数据写入undo日志
4. 为undo日志的修改创建redo日志，写入redo log buffer
5. 在buffer pool中修改数据页，回滚指针指向undo日志
6. 为数据页的修改创建redo日志，写入redo log buffer（是否可以写入磁盘？？？？？？？？？？？？？）

提交阶段：

1. 写binlog到磁盘
2. 修改undo日志状态为"purge"
3. 根据innodb_flush_log_at_trx_commit判断是否需要立即把redo log buffer写入磁盘
4. buffer pool中的数据页根据规则等待合适的时机写入磁盘

恢复或者回滚阶段：

1. 根据redo日志中的LSN和事务ID，数据文件中的LSN和binlog中的事务ID决定需要做恢复还是回滚
2. 如果事务已提交，但数据页还未写入磁盘，需要恢复，根据redo日志中的字段对数据页进行恢复
3. 如果需要回滚，根据undo日志中的上一个版本数据进行回滚



## 分布式事务

分布式事务有两种形式，一种是分布式数据库事务，另一种分布式微服务事务。 

第一种分布式数据库事务由传统单机事务进化而来。当单机数据库无法支撑所有负载时，必然要将数据拆分到多台数据库。如果一次操作设计到多台数据库，那就需要一种方案来维护整个数据库集群的事务特性，也就是ACID特性。

第二种分布式微服务事务由分布在不同机器上的服务组成。最近微服务架构兴起，不同的服务被拆分到不同的服务器进程中。比如一个网络购物服务，由订单服务器，支付服务器，仓储服务器和物流服务器组成。每个服务器使用的数据存储方案可能不同，有的用MySQL，有的用Redis。如何让网络购物服务完整执行，而不会导致支付了但仓库中没有剩余库存，这是分布式服务事务需要处理的问题之一。

分布式事务和单机事务相比，处理的问题更为困难。

第一个问题是因为有多个事务参与者。传统单机事务，如果宕机，整个事务的执行都会失败，原子性比较好处理。但是分布式事务，参与者分布在不同的机器上，可能第一个参与者成功锁住资源，而第二个参与者对资源加锁失败，也可能第一个参与者提交事务成功，但是第二个参与者提交时机器宕机，处理起来困难。

第二个问题是网络超时导致的问题。服务器A告诉服务器B提交事务，然后超时了，没有收到服务器B的响应。这时有可能服务器B没收到请求，也可能服务器B收到请求，成功处理，发送给A的响应丢失。如何区分这两种情况，然后进行处理也是困难的地方。

第三个问题是网络导致的延迟问题。传统单机事务不存在网络延迟，所有操作都在一台机器上处理。但是分布式事务由于网络交互带来了延时，这可能会导致实现方案的差异。比如前面介绍中我们提到过事务ID的概念，在分布式系统中，如何保证事务ID的唯一性，以及区分并发事务之间的先后顺序？一种方案是提供一个全局的服务来分配事务ID，单调递增。这种方案在同城部署的时候还好，因为同城的网络往返RTT大概在1ms内，但是如果是全球部署的系统，比如Google Spanner，一个服务器在北美洲，另一个服务器在欧洲，跨洲往返RTT可能是200ms，这个延迟显然是无法接受的，所以这种全局事务ID服务方案不行。

那么分布式事务应该如何实现，接下来我们讨论原理和常见的几种方案。

### 两阶段提交

两阶段提交中有两个角色，一个是参与者，用于管理本地资源，实现本地事务。另一个是协调者，用于管理分布式事务，协调事务各个参与者之间的操作。

流程如下：

```
Coordinator                                          Participant

prepare*
                             QUERY TO COMMIT
                 -------------------------------->
                             VOTE YES/NO             prepare*/abort*
                 <-------------------------------
                 
commit*/abort*
                             COMMIT/ROLLBACK
                 -------------------------------->
                             ACKNOWLEDGMENT          commit*/abort*
                 <--------------------------------  
end

An * next to the record type means that the record is forced to stable storage.
```

#### 第一阶段Prepare

1. 协调者分配事务ID，写到磁盘，然后询问所有参与者是否可以执行事务

2. 参与者执行事务，对资源加锁，写redo/undo日志到磁盘

3. 如果参与者执行事务成功，回复Yes，如果执行失败，回复No

#### 第二阶段Commit/Abort

分两种情况

**所有参与者回复Yes，执行Commit**

1. 协调者写Commit日志到磁盘，然后向所有参与者发送Commit请求

2. 参与者执行Commit操作，写Commit日志到磁盘，释放资源

3. 参与者回复协调者ACK完成消息

4. 协调者收到所有参与者完成消息后，完成事务

**至少有一个参与者回复No，执行Abort**

1. 协调者写Abort日志到磁盘，然后向所有参与者发送Abort请求

2. 参与者使用Undo日志回滚事务，写Abort日志到磁盘，释放资源

3. 参与者回复协调者回滚完成消息

4. 协调者收到所有参与者回滚完成消息后，取消事务

整个两阶段流程，看着挺简单的，但是麻烦的地方在于如何处理服务器宕机和网络超时问题，我们来分析一下整个过程如何处理这两个问题。首先有个原则是如果协调者已经写Commit日志到磁盘，各个参与者就应该提交事务，不允许回滚。

#### 网络超时问题

1. 协调者发送完Prepare请求后，在规定时间内没有收到所有参与者的回复。
   - 此时简单的做法是协调者发送Abort请求给所有参与者，参与者执行回滚操作，但是这个做法在某些条件下可能会导致事务不一致，后文会描述。
   - 复杂的做法是协调者向超时的参与者查询事务状态。
     - 如果所有参与者回复Yes，协调者发送Commit请求。
     - 如果有参与者回复No，协调者发送Abort请求。
     - 如果有参与者表示还没收到Prepare请求，说明请求丢失，或者该参与者重启过，并且重启前没有写redo/undo日志，此时协调者发送Abort请求给所有参与者。
2. 参与者等待协调者的Commit请求超时。

   - 如果该参与者Prepare阶段回复的是No，此时可以直接Abort事务。因为协调者收到No回复后，给所有参与者发送的也是Abort请求。

   - 如果该参与者Prepare阶段回复的是Yes就比较麻烦了。因为不知道协调者发送的是Commit还是Abort，该参与者不能直接执行Commit或者Abort。此时该参与者需要向协调者查询事务状态。

     - 如果协调者状态是Commit，则该参与者可以执行Commit操作。
     - 如果协调者状态是Abort，则该参与者执行Rollback操作。
     - 如果发现协调者宕机怎么办？这时需要各个参与者之间互相查询事务状态。
       - 如果有参与者回复的是No，所有参与者执行Abort操作。

       - 如果所有参与者回复的是Yes，此时参与者可以执行Commit操作吗？不能，原因有两个。

         - 如果之后协调者的Commit请求又被参与者收到了，此时参与者需要能识别出这个事务已经Commit了，不能重复Commit，当然这个问题还比较好处理。

         - 即使所有参与者都回复的是Yes，协调者如果在接收回复阶段超时了，然后写Abort日志，之后宕机了。此时参与者执行Commit操作会导致**事务不一致**。这就是前文说的协调者没有收到Prepare回复时，不能简单Abort的原因。那怎么办呢？一种办法是协调者在没有收到Prepare回复时向参与者查询，另一种办法是保证协调者的可用性(Availablity)，主协调者宕机后有其他的协调者能继续服务。
3. 协调者等待参与者Commit/Abort之后的ACK超时。根据上面的讨论，参与者一定会执行Commit/Abort操作，此时协调者可以认为事务已经完成了，返回结果给客户端。事实上，协调者写完Commit/Abort日志，发送Commit/Abort请求给参与者后，就可以直接返回结果给客户端，不必等待最后的ACK。

#### 宕机问题

宕机问题都有可能转化为超时问题，宕机前如果写了redo/undo日志，重启后需要额外处理。

1. 协调者发出Prepare请求前宕机。此时事务还未开始，不会有影响。
2. 参与者在Prepare阶段宕机。此时协调者超时，处理方式和上文网络超时第一条相同。
3. 在协调者写Commit/Abort日志前，协调者宕机。此时如果协调者在参与者超时前重启，协调者需要向参与者查询事务状态。如果协调者没有及时重启，此时参与者超时，处理方式和上文网络超时第二条相同。
4. 在协调者写Commit/Abort日志后，协调者宕机。此时如果协调者在参与者超时前重启，没收到Commit/Abort请求的参与者在超时后会向协调者查询事务状态，执行Commit/Abort操作。如果协调者在参与者超时后才重启，此时参与者需要在超时后发起相互之间的查询操作，但是可能会遇到上文介绍的**事务不一致**。
5. 在协调者写Commit/Abort日志后，参与者宕机。此时参与者在重启后需要向协调者查询事务状态，执行对应的操作。

通过上面的讨论，我们可以看到由于事务状态分布在协调者和各个参与者之间，要保证两阶段提交的一致性是非常困难的。如果能够保证协调者的可用性(Availablity)，比如采用主备或者Paxos来实现，同时还需要保证各个参与者宕机后能够重启恢复，那么正确实现两阶段提交会简单不少，读者可以再分析一遍上述情况。

上面只讨论了原子性、一致性和持久性，没讨论隔离性的实现，隔离性可以考虑采用锁方案，实现简单，但性能可能比较差，另一种就是前文介绍的MVCC方案，性能会好不少，但实现复杂。

#### 两阶段提交协议本身存在的问题

即使解决了前面说的一些问题，两阶段提交协议还是会有问题，这两个问题是协议本身存在的，如下：

1. 参与者资源阻塞：阻塞问题分成两个方面，一个方面是整个事务过程中，参与者上的相应资源会锁住。设想一下在Prepare阶段，如果有9个参与者，其中有一个没有条件完成事务，其他8个参与者还是需要锁住资源，写redo/undo日志，然后在Commit阶段，这8个参与者又需要回滚。另一个方面是如果协调者或者参与者宕机，必须等待超时或者服务器重启后发起Commit或者Rollback后才能释放资源。
2. 网络分区问题：某个参与者在Prepare阶段发送完Yes后，跟协调者和其他的参与者发生网络分区。此时其他参与者收到协调者的Commit请求后执行提交操作，但是该参与者收不到Commit请求，超时后该参与者也无法向其他参与者查询事务状态，可能导致**事务不一致**。



### 三阶段提交

针对两阶段提交的问题，有人提出了三阶段提交。三阶段提交比两阶段提交多了一个PreCommit阶段，流程如下：

```
Coordinator                                          Participant

can_commit
                             QUERY
                 -------------------------------->
                             VOTE YES/NO             check
                 <-------------------------------

pre_commit*
                             PREPARE TO COMMIT
                 -------------------------------->
                             VOTE YES/NO             prepare*/abort*
                 <-------------------------------
                 
do_commit*/abort*
                             COMMIT/ROLLBACK
                 -------------------------------->
                             ACKNOWLEDGMENT          commit*/abort*
                 <--------------------------------  
end

An * next to the record type means that the record is forced to stable storage.
```

#### 第一阶段CanCommit	

1. 协调者询问所有参与者是否可以执行事务
2. 参与者检查是否可以执行事务，如果可以执行事务成功，回复Yes，如果不能，回复No

#### 第二阶段PreCommit

分两种情况

**CanCommit阶段所有参与者回复**

1. 协调者向所有参与者发送PreCommit请求
2. 参与者执行事务，对资源加锁，写redo/undo日志到磁盘
3. 如果参与者执行事务成功，回复Yes，如果执行失败，回复No

**CanCommit至少有一个参与者回复No**

1. 协调者向所有参与者发送Abort请求
2. 参与者取消事务

#### 第三阶段DoCommit

分两种情况

**PreCommit阶段所有参与者回复Yes**

1. 协调者向所有参与者发送Commit请求
2. 参与者执行Commit操作，释放资源
3. 参与者回复协调者ACK完成消息
4. 协调者收到所有参与者完成消息后，完成事务

**PreCommit至少有一个参与者回复No**

1. 协调者向所有参与者发送Rollback请求
2. 参与者使用Undo日志回滚事务，释放资源
3. 参与者回复协调者回滚完成消息
4. 协调者收到所有参与者回滚完成消息后，取消事务

三阶段提交和两阶段提交相比，优点在于增加了CanCommit阶段，这个阶段资源不会加锁，如果有某个参与者不能执行事务，不会阻塞其他参与者。但是三阶段依然存在事务不一致和网络分区问题。而且三阶段增加了一个请求，整个事务的延迟会增加。



### 分布式事务的实现

分布式事务实现上主要有四种模型：XA模型、TCC模型、Saga模型和MQ模型。

#### XA模型

XA模型是由X/Open组织制定的分布式事务规范和接口。

![](/img/in-post/distributed-transaction/post-xa.png)

XA模型中有三种角色：

- AP(Application Program): 客户端程序，定义事务的内容
- TM(Transaction Manager): 事务的管理者，也即两阶段提交的协调者 
- RM(Resource Manager): 资源管理者，也即两阶段提交的参与者

XA接口规范如下：

![](/img/in-post/distributed-transaction/post-xa-api.png)

XA模型本质是一个两阶段提交的接口规范。

XA模型严格保障事务ACID特性。事务执行过程中需要将资源锁定，这样可能会导致性能低下。因此eBay架构师Dan Pritchett提出了BASE理论。BASE是三个短语的缩写:

- Basically Available: 基本可用，允许损失部分可用性
- Soft state: 软状态，允许数据存在中间状态
- Eventually consistent: 最终一致性，数据最终会达到一个一致的状态

BASE通过牺牲强一致性来获得可用性和系统性能的提升。下面介绍的TCC、Saga、MQ都属于BASE理论模型，满足最终一致性。

#### TCC模型

TCC模型最早由Pat Helland于2007年发表的一篇名为《Life beyond Distributed Transactions:an Apostate’s Opinion》的论文提出。模型中，事务参与者需要实现Try, Confirm, Cancel三个接口。

- Try: 参与者检查资源是否有效，预留资源。
- Confirm: 参与者提交资源。
- Cancel: 参与者执行回滚操作，恢复预留资源。

举个例子，A向B转账100块钱。事务由两个参与者PA(Participator A)和PB(Participator B)组成，PA负责给A减100块钱，PB负责给B加100块钱。TCC模型流程如下：

Try阶段

- PA检查A账户是否有足够的余额，冻结A账户的100块，写日志到磁盘。这个阶段不需要锁住A账户，其他事务可以对A账户操作，可以看到这100块，但是不能对这100块钱操作。
- PB检查B账户的合法性。

如果PA和PB在Try阶段都返回成功，进入Confirm阶段

- PA减A账户的100块，写日志到磁盘。
- PB加100块到B账户，写日志到磁盘。

如果PA或者PB在Try阶段返回失败，进入Cancel阶段

- PA恢复A账户的100块，写日志到磁盘，结束事务。
- PB结束事务。

TCC模型因为没有在Try阶段加锁，所以性能高于两阶段提交。不同事务可以并发执行，只要参与者管理的剩余资源足够。具体实现时，需要注意以下几个问题：

- 每个参与者需要考虑如何将自身的业务拆分成Try, Confirm, Cancel这三个接口。
- 如果Try成功，需要保证Confirm一定能成功。
- 需要保证三个接口的幂等性。由于网络超时，请求可能会重发，这时参与者需要保证操作的幂等性，不能重复执行同一个请求。
- 需要处理空Cancel操作。如果参与者没有收到Try请求，协调者可能触发Cancel请求的发送，这时协调者需要处理这种没有收到Try请求，反而收到Cancel请求的情况。
- 需要处理先收到Cancel请求，后收到Try请求。如果由于网络超时，参与者没有收到Try请求，协调者可能触发Cancel请求的发送，参与者先收到Cancel请求，然后之前超时的Try请求又发送到参与者。
- 和两阶段提交一样，TCC模型也会遇到网络分区，服务器宕机等问题。

TCC模型可以满足原子性(A)，一致性(C)和持久性(D)，但无法满足隔离性(I)。不同事务并发执行的时候，隔离性只能满足读未提交级别(Read Uncommited)。

#### Saga模型

Saga模型由Hector & Kenneth于1987年提出。这个模型中，每个事务参与者需要提供一个正向执行模块和逆向回滚模块，执行模块用于执行事务的正常操作，回滚模块用于在失败时执行回滚操作。

执行流程如下，其中Ti表示每个参与者的正向执行模块，Ci表示每个参与者的逆向回滚模块：

- 成功流程：T1 -> T2 -> T3 -> ... -> Tn
- 失败流程：T1 -> T2 -> ... Ti (failed) -> Ci -> ... -> C2 -> C1

具体实现时，根据是否有协调者，有两种实现方式：

- 基于事件的分布式方案(Events/Choreography)：没有集中式协调者，每个参与者订阅其他参与者的事件，执行自己的业务，生成新的事件供其他参与者使用。
- 基于命令的协调者方案(Command/Orchestrator)：由集中式协调者控制事务流程，协调者发送命令给各个参与者，参与者执行事务后发送结果给协调者。

举个例子来说明这两者的不同。一次完整的网络购物由四个服务组成：订单服务(Order Service)、支付服务(Payment Service)、库存服务(Stock Service)和送货服务(Delivery Service)组成。

基于事件的分布式方案的成功流程如下：

![](/img/in-post/distributed-transaction/post-saga-choreography-commit.png)

1. 订单服务创建订单，发出订单创建事件ORDER_CREATED_EVENT。
2. 支付服务订阅ORDER_CREATED_EVENT，执行支付操作，发出支付完成事件BILLED_ORDER_EVENT。
3. 库存服务订阅BILLED_ORDER_EVENT，扣除库存，发出货物准备完成事件ORDER_PREPARED_EVENT。
4. 送货服务订阅ORDER_PREPARED_EVENT，送货，发出货物交付事件ORDER_DELIVERED_EVENT。
5. 订单服务订阅ORDER_DELIVERED_EVENT，标记事务完成。

基于事件的分布式方案的失败流程如下：

![](/img/in-post/distributed-transaction/post-saga-choreography-rollback.png)

1. 库存服务发现库存不足，发出库存不足事件PRODUCT_OUT_OF_STOCK_EVENT。

2. 支付服务订阅PRODUCT_OUT_OF_STOCK_EVENT，给用户返回钱。
3. 订单服务订阅PRODUCT_OUT_OF_STOCK_EVENT，标记订单失败。

基于命令的协调者方案的成功流程如下：

![](/img/in-post/distributed-transaction/post-saga-orchestrator-commit.png)

1. 订单服务创建订单，发送订单事务给协调者。
2. 协调者发送支付命令给支付服务，支付服务执行支付，返回结果给协调者。
3. 协调者发送准备订单命令给库存服务，库存服务扣除库存，返回结果给协调者。
4. 协调者发送送货命令给送货服务，送货服务送货，返回结果给协调者。
5. 协调者发送结果给订单服务，标记事务完成。

基于命令的协调者方案的失败流程如下：

![](/img/in-post/distributed-transaction/post-saga-orchestrator-rollback.png)

1. 库存服务发现库存不足，返回库存不足结果给协调者。
2. 协调者发送返钱命令给支付服务，支付服务返钱给用户。
3. 协调者发送失败结果给订单服务，标记事务失败。

因为Saga模型是一阶段，而TCC模型是两阶段，和TCC模型相比，Saga模型的性能要高一些，而且实现的时候要简单一些。但因为Saga模型没有Try阶段预留操作，在回滚的时候就会麻烦不少。比如发邮件服务，正向阶段已经发邮件给用户，回滚的时候会对用户不友好。

和TCC类似，Saga模型在实现时也需要注意幂等性，空操作，请求乱序等问题。另外Saga模型也可以满足原子性(A)，一致性(C)和持久性(D)，但无法满足隔离性(I)。不同事务并发执行的时候，隔离性只能满足读未提交级别(Read Uncommited)。

#### MQ模型

MQ模型使用消息队列(Message Queue)来通知事务的各个参与者执行操作。

MQ模型流程如下：

```
Sponsor                                    MQ                                   Participant

                         PREPARE
                 --------------------->
                         ACK             prepare*
                 <---------------------

commit*
                         COMMIT
                 --------------------->
                       VOTE YES/NO       commit*/abort*
                                                                COMMIT
                 																			  --------------------->
                                                                ACK              commit*
                                         end            <---------------------                                                                                           


An * next to the record type means that the record is forced to stable storage.
```

1. 事务发起者发送Prepare消息到MQ。
2. MQ收到Prepare消息后，保存到磁盘，不发送给事务参与者，返回ACK给事务发起者。
3. 事务发起者如果没收到ACK，取消事务的执行，给MQ发送Abort消息。如果收到ACK，执行本地事务，给MQ发送Commit消息。
4. MQ收到消息后，如果是Abort，删除事务消息。如果是Commit，MQ修改消息状态为可发送，并发送该事务消息给事务参与者。
5. 事务参与者收到消息后，执行事务，然后发送ACK给MQ。
6. MQ删除事务消息，标记事务完成。

流程中几个需要注意的地方：

- 第4步中，如果MQ没有收到事务发起者发送的Commit/Abort消息，MQ会向发起者查询事务状态，根据状态执行后续操作。
- 第5步中，如果MQ长时间没有收到事务参与者的ACK消息，MQ会按照间隔（比如1分钟，5分钟，10分钟，1小时，1天等）不断重复发送Commit消息给事务参与者，直至收到ACK。

从以上流程可以发现MQ事务模型依赖于MQ支持事务消息，目前只有RocketMQ支持事务消息。如果不用RocketMQ，需要自己实现一个消息可靠性模块，完成类似的功能。

MQ模型中，需要确保参与者一定能成功执行事务，参与者不能说自己没有条件执行事务，比如支付服务作为参与者检查发现用户余额不足。所以MQ模型有很大的使用范围限制。一般在逻辑上有可能失败的操作（比如支付）需要由事务发起者完成，而事务参与者只执行一定会成功的操作（比如充话费、发送游戏道具等）。

和TCC，Saga模型一样，MQ模型也需要注意事务参与者的幂等性。



#### 各模型开源实现

[ByteTCC](https://github.com/liuyangming/ByteTCC)

支持Java项目，支持TCC和Saga模型，支持Spring Cloud和Dubbo

[tcc-transaction](https://github.com/changmingxie/tcc-transaction)

支持Java项目，支持TCC模型

[EasyTransaction](https://github.com/QNJR-GROUP/EasyTransaction)

支持Java项目，支持TCC、Saga和MQ模型

[Seata](seata.io)

阿里开源的分布式事务方案，支持Java，支持AT(针对SQL数据库)、TCC、Saga模型

[Apache ServiceComb](https://servicecomb.apache.org/)

华为开源的分布式事务方案，支持Java，支持Saga模型



参考：

[数据库事务](https://zh.wikipedia.org/wiki/数据库事务)

[ACID](https://zh.wikipedia.org/wiki/ACID)

[事务隔离]([https://zh.wikipedia.org/wiki/%E4%BA%8B%E5%8B%99%E9%9A%94%E9%9B%A2](https://zh.wikipedia.org/wiki/事務隔離))

[Innodb中的事务隔离级别和锁的关系](https://tech.meituan.com/2014/08/20/innodb-lock.html)

[MySQL技术内幕：InnoDB存储引擎](https://book.douban.com/subject/24708143/)

[MySQL InnoDB Update和Crash Recovery流程](https://cloud.tencent.com/developer/article/1072722)

[InnoDB undo log 漫游](http://mysql.taobao.org/monthly/2015/04/01/)

[5.7 Innodb事务系统](http://mysql.taobao.org/monthly/2014/12/01/)

[InnoDB Repeatable Read隔离级别之大不同](http://mysql.taobao.org/monthly/2017/06/07/)

[从0到1理解数据库事务（下）：隔离级别实现——MVCC与锁](https://www.jianshu.com/p/387579a98a4f)

[The basics of the InnoDB undo logging and history system](https://blog.jcole.us/2014/04/16/the-basics-of-the-innodb-undo-logging-and-history-system/)


[两阶段提交](https://zh.wikipedia.org/wiki/%E4%BA%8C%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4)

[分布式事务 - 两阶段提交与三阶段提交](https://www.iteye.com/blog/m635674608-2322853)

[Saga Pattern | How to Implement Business Transactions Using Microservices](https://dzone.com/articles/saga-pattern-how-to-implement-business-transaction)

[如何选择分布式事务解决方案](https://mp.weixin.qq.com/s/2AL3uJ5BG2X3Y2Vxg0XqnQ)

