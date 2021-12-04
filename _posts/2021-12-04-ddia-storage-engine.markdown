---
layout: post
title: "存储引擎漫话"
author: "Yang"
header-style: text
tags:
  - DDIA
  - hash
  - LSM-Tree
  - B-Tree
  - SSTable
  - index
  - database
---

- any list
{:toc}

最近在看[《Designing Data-Intensive Applications》](https://book.douban.com/subject/26197294/)，本文主要记录书中第三章存储和检索的内容，加深自己的理解。

# 最简单的数据库

什么是世界上最简单的数据库？只需要两个bash函数：

```
#!/bin/bash
db_set () {
    echo "$1,$2" >> database
}
db_get () {
    grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

使用例子：

```
$ db_set 123456 '{"name":"London","attractions":["Big Ben","London Eye"]}' 
$ db_set 42 '{"name":"San Francisco","attractions":["Golden Gate Bridge"]}'
$ db_get 42
{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
$ db_set 42 '{"name":"San Francisco","attractions":["Exploratorium"]}' 
$ db_get 42
{"name":"San Francisco","attractions":["Exploratorium"]}
$ cat database
123456,{"name":"London","attractions":["Big Ben","London Eye"]} 
42,{"name":"San Francisco","attractions":["Golden Gate Bridge"]} 
42,{"name":"San Francisco","attractions":["Exploratorium"]}
```

这个最简单的数据库底层存储文件是一个文本文件"database"，每行是用逗号分隔的key和value。每次调用db_set，把key和value追加到dababase文件中。每次调用db_get，查询的是对应key的最后一次写入记录。

这个数据库具备了存储和查询这两种最基本的数据库功能。存储使用了append操作，而不是随机写。对于写操作，append是最快的操作。对于读操作，使用的是线性查找，复杂度O(n)。随着数据量的增加，查找所花的时间也会线性增长。

为了提升查询性能，可以给数据增加索引。索引可以显著地提升查询性能，但同时也会降低写入性能，因为写入时除了要写入数据本身，还要更新索引，所以需要权衡如何高效地建立索引。

# 哈希索引

一种直观地建立索引的方法是使用哈希表索引(Hash Indexes)。我们可以在内存中建立一个哈希表，哈希表的key是数据的key，哈希表的value是这个key对应的数据所在的文件的偏移。举个例子，上文123456和42这两个key的索引如下：

![图一](/img/in-post/2021-12-04-ddia-storage-engine/hash-index1.png)

[Bitcask存储引擎](https://riak.com/assets/bitcask-intro.pdf)使用的就是这种方式。Bitcask存储引擎写入时对文件执行的是追加(append)操作，查询的时候通过建立哈希索引加快查询性能。哈希表保存在内存中。这种存储引擎比较适合key的数量有限，并且写操作比较频繁的场景。Bitcask存储引擎遇到的问题和解决方案如下。

**数据文件只有append操作，磁盘早晚会有耗尽的一天，怎么解决这个问题？**

解决方案是写入数据的时候，把数据文件分成固定的大小，一个数据文件达到大小后，写入新的文件中。然后对已经写入的文件进行压缩(compaction)操作。因为同一个key的操作可能有很多次，只有最后一次操作的值才有意义，之前操作的记录没有存在的必要，所以我们可以遍历已经保存过的文件，只保留每个key的最后一次操作值，然后把这些最新的值写入新的数据文件中，老的数据文件就可以删除了。压缩完了还需要更新哈希表中key对应的文件偏移。

![图二](/img/in-post/2021-12-04-ddia-storage-engine/hash-index2.png)

上图中yawn, scratch, memw, purr这四个key在老的数据文件中出现了多次，经过压缩后，数据减少了很多。

**哈希表只保存在内存中，进程重启或者崩溃的时候怎么办？**

一种解决方法是进程重启的时候，扫描所有的数据文件，在内存中重建哈希表。这个过程可能很耗时间，所以Bitcask使用的优化方式是把哈希表也保存在磁盘中，重启的时候可以加载磁盘的哈希表，快速建立索引。当然这样会降低写入的性能，因为写入数据的时候需要把哈希索引表也写入磁盘中，需要权衡。

哈希索引也有缺点，主要有两个：

1. 如果key的数量很多，内存不够大，无法把所有的key都保存在内存中。
2. 范围查询效率不高。比如需要查询kitty00000到kitty99999这个范围的数据，必须先到哈希表中查询每个key的文件偏移位置，再去读磁盘。

# SSTable和LSM-Tree

既然哈希索引范围查询效率不高，有没有优化方法呢？

有一种解决方案是使用Sorted String Table(SSTable)。上文介绍Bitcask存储引擎的数据分成了一个个固定大小的文件，每个文件中key的顺序是不固定的。SSTable的区别在于每个数据文件是根据key排过序的。这是如何做到的呢？

首先，数据写入的时候先把数据保存在内存中的平衡树结构中（比如红黑树或者AVL树）。这种内存树结构也被叫做memtable。

当memtable的大小超过一定值时（比如几MB），把当前memtable写入磁盘的SSTable文件中。因为平衡树本身是有序的，所以把它们写入磁盘的时候也可以保持顺序写入。新的写请求会写入到内存中新的memtable。

和Bitcask存储引擎一样，SSTables方案里有一个后台压缩进程持续地对已经生成的SSTable文件进行合并。合并的方式可以采用归并排序(mergesort)算法，这样合并后的SSTable文件也是有序的。排序过程中，如果一个key在多个文件中出现，只需要使用最新的文件中的数据。如下图所示：

![图三](/img/in-post/2021-12-04-ddia-storage-engine/sstable1.png)

使用SSTables结构，查询数据时，首先到内存memtable中查询数据是否存在，如果不存在，再到磁盘的SSTable文件中查找。对于SSTable文件，内存中也有key的索引表，区别在于我们不需要在内存中保存所有的key。因为SSTable中数据是有序的，我们只需要保存少数几个key的文件位置偏移就行了。如下图所示：

![图四](/img/in-post/2021-12-04-ddia-storage-engine/sstable2.png)

对于key的范围为handbag和handprinted，内存中只需要保存handbag的位置偏移，查找key时先查找和这个key最相近的key所对应的位置偏移，然后读取整个偏移范围的内容，对这个范围内的数据再查找我们想要查的key。以handiwork为例，查找这个key时，先在内存索引表中找到最近的key为handbag，然后从SSTable文件中对应偏移位置开始读取对应长度的数据，然后扫描这段范围的数据，找到handiwork的值。一般来说内存索引表中一个key对应几KB的数据，所以扫描很快。另外保存这段范围的数据时，可以压缩之后再写入磁盘，这样增加磁盘IO的效率，不过这也带来了CPU压缩和解压缩的开销。比如handbag到handprinted这个范围的数据可以先压缩后再保存在磁盘中。

上文说到查询数据时先到memtable中查找key是否存在，不存在再到最新的SSTable文件中查找，然后再到旧的SSTable文件中查找，一直到最老的SSTable文件。如果数据库中根本不存在这个key的话，查找过程可能会比较耗时间，一种优化方式是使用[布隆过滤器(Bloom filter)](https://en.wikipedia.org/wiki/Bloom_filter)来判断key是否不存在，key存在的话再执行上面的查找流程。

SSTable方案内存中保存了memtable，如果进程重启或者崩溃怎么处理？一种方案是内存写入memtable前，磁盘先写入一个log文件，这个log文件只有append操作。进程重启的时候，使用这个log文件重建memtable。每次内存中的memtable写入SSTable文件后，这个log文件也可以删除了。

整个方案写入数据的时候写入memtable和append-only log文件中，所以写入性能很高，同时SSTable文件是有序的，进行范围查询的时候性能也很高。

方案最早是在[The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf)这篇论文中描述，所以也叫LSM-Tree。很多数据库都使用了LSM-Tree，包括LevelDB、RocksDB、Cassandra、HBase等。

# B-Tree

虽然LSM-Tree近来发展很快，B-Tree仍然是目前使用最广泛的索引结构。

B-Tree也是一种有序的数据结构，可以高效地进行范围查找。示例如下：

![图五](/img/in-post/2021-12-04-ddia-storage-engine/btree1.png)

B-Tree中，每个数据文件的大小是固定，一般为4K或者更大，称作块(block)或者页(page)。B-Tree是分层结构。最上层的页作为根页。根页中按顺序保存了一些key，同时保存了指向子页的指针，也就是子页数据文件的位置。每个子页保存了一段范围的key和对应的位置指针。最底层的称做叶子页(leaf page)，叶子页保存了所有的数据，也有实现方式是在叶子页中只保存数据的指针。每页中保存的指针数量叫做分支系数(branching factor)，一般是几百个。大部分数据库B-Tree的层数是3到4层。对于4层的B-Tree，如果页是4KB，分支系数是500的话，可以储存256TB数据。

B-Tree添加key的时候，需要找到包含这个key的页，如果这个页已经满了，需要分裂成两个新的页。示例如下：

![图六](/img/in-post/2021-12-04-ddia-storage-engine/btree2.png)

B-Tree修改页的时候也面临进程崩溃的问题。为了解决这个问题，写入的时候会先写入一个append-only log文件中，也叫做write-ahead log(WAL)或者[redo log](https://yang.observer/2020/05/06/distributed-transaction/#redo%E6%97%A5%E5%BF%97)。WAL写入成功后再写入数据页，如果进程崩溃了，WAL可以用来恢复数据页。

# B-Tree和LSM-Tree比较

我们来比较一下B-Tree和LSM-Tree。一般来说，LST-Tree写入速度更快，而B-Tree查询速度更快。LSM-Tree查询慢一些的原因是因为需要查询几次才能在对应的SSTable中找到数据。不过究竟哪个更合适还是需要根据自己业务数据的性能测试结果来定。

B-Tree写入时需要至少两次IO操作，第一次写入WAL，第二次写入数据页。LSM-Tree执行写入操作的时候只需要写入一次WAL，另外一次是内存操作。但是因为后台有个压缩进程会反复对数据进行合并和压缩，一个数据会反复写入多次，这对SSD不太友好，因为SSD的写入次数是有限的。另外后台压缩进程运行的时候会占据大量磁盘IO，可能会影响正在执行的写入操作。随着数据文件的增大，压缩占据的磁盘IO也会更多，如果同时写入操作也很多的话，有可能压缩操作的速度赶不上写入操作的速度，导致磁盘空间被用光。

LSM-Tree更容易被压缩，所以数据文件比B-Tree要小一些。而且B-Tree数据页可能不是完全利用的，比如一个页分裂成两个页的时候，这两个页都只有一半空间被使用了，LSM-Tree不存在这个问题。

# 二级索引

我们知道很多数据库都可以建立二级索引，它的实现方式和一级索引一样，也可以使用B-Tree或者LSM-Tree。这里会遇到的问题主要是数据本身存在哪里？

一种方式是数据存在每个索引中，这样会造成数据重复，有几个索引就会有几份数据，更新数据的时候会比较麻烦，需要更新多份数据，可能会导致一致性问题。这种方式也叫聚集索引(clustered index)，MySQL InnoDB的主键使用了这种方式。

另外一种方式是数据只有一份，所有索引中不保存数据，只保存数据的指针。更新数据的时候如果数据长度没有变大的话，只需要更新一次。如果数据变大的话，可能需要把数据保存在新的地方，所有索引中的数据指针都需要更新。MySQL InnoDB中的二级索引使用了类似的方式，没有保存数据，保存的是主键中的数据指针。

还有一种折中方案是在索引文件中只保存部分数据列，而不是整行，这也称作覆盖索引(covering index)。

# 参考

[Designing Data-Intensive Applications](https://book.douban.com/subject/26197294/)
[The Log-Structured Merge-Tree (LSM-Tree)](https://www.cs.umb.edu/~poneil/lsmtree.pdf)
