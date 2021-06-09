---
layout: post
title: "计算机的时钟（三）：向量时钟"
author: "Yang"
header-style: text
tags:
  - Vector Clocks
  - 向量时钟
  - Light Cone
  - 光锥
  - Relativity
  - 相对论
  - 朋友圈
---

本系列文章链接如下：

[计算机的时钟（一）：NTP协议](http://yang.observer/2020/07/11/time-ntp/)

[计算机的时钟（二）：Lamport逻辑时钟](http://yang.observer/2020/07/26/time-lamport-logical-time/)

[计算机的时钟（三）：向量时钟](http://yang.observer/2020/09/12/vector-clock/)

[计算机的时钟（四）：TrueTime](http://yang.observer/2020/11/02/true-time/)

[计算机的时钟（五）：混合逻辑时钟](http://yang.observer/2020/12/16/hlc/)



本系列文章主要介绍计算机系统中时钟的处理。主要内容包含NTP，Lamport逻辑时钟，向量时钟，TrueTime等。本文是第三篇，介绍向量时钟。

在[《计算机的时钟（二）：Lamport逻辑时钟》](http://yang.observer/2020/07/26/time-lamport-logical-time/)中，我们介绍过**对于任意两个事件a和b，如果 a → b，那么 C (a) < C (b)，但是反向并不成立，C(a) < C(b)推不出来a→b。**本文介绍的向量时钟(Vector Clocks)可以保证反向也能成立，也即充分条件变成了充分必要条件，这是如何做到的呢，且听我一一道来。



# 向量时钟算法

向量时钟是1988年由Colin Fidge和Friedemann Mattern提出的，比Lamport老爷子提出逻辑时钟晚了刚好十年。Lamport逻辑时钟存在的问题是**不能描述事件的因果关系**，C(a) < C(b)不能推导出a → b，这样导致即使知道了两个逻辑时钟值，但却不能确定这两个事件的因果关系。

向量时钟可以解决这两个问题，它的思想是进程间通信的时候，不光同步本进程的时钟值，还同步自己知道的其他进程的时钟值。算法如下：

分布式系统中每个进程Pi保存一个本地逻辑时钟向量值VCi，向量的长度是分布式系统中进程的总个数。VCi (j) 表示进程Pi知道的进程Pj的本地逻辑时钟值，VCi的更新算法如下：

1. 初始化VCi的值全为0：VCi = [0, ... , 0]

2. 进程Pi每发生一次事件，VCi[i]加1。

3. 进程Pi给进程Pj发送消息，需要带上自己的向量时钟VCi。

4. 进程Pj接收消息，需要做两步操作。

   4.1 对于VCj向量中的每个值VCj[k]，更新为 max (VCi[k], VCj[k])。

   4.2 将VCj中自己对应的时钟值加1，即VCj[j]加1。

向量时钟举例：

![图二](/img/in-post/2020-09-12-vector-clock/post-vector-clock2.png)

向量时钟里的向量是一种偏序关系：

如果向量VCi中的每个元素VCi[k]都小于等于VCj中的对应元素VCj[k]，则VCi *≤* VCj。

如果VCi中的每个元素VCi[k]都和VCj中的对应元素VCj[k]相等，则VCi = VCj。

如果VCi和VCj不能比较大小，则称两个向量是并发的 VCi \|\| VCj。

从以上算法可以很容易地得出下面两个结论：

1. 同一个进程内的两个事件a和b，如果 a → b，那么 VCi (a) < VCi (b)。
2. a是Pi进程的消息发送事件，b是Pj进程该消息的接收事件，那么 VCi (a) < VCj (b)。

由以上两个结论又可以得出，**对于任意两个事件a和b，如果 a → b，那么 VC (a) < VC (b)**。

怎么证明VC(a) < VC(b) 可以推导a → b呢？（以下证明过程摘自[逻辑时钟 - 如何刻画分布式中的事件顺序](https://writings.sh/post/logical-clocks)）

如果事件a和b在同一个进程内，很显然 a → b。

如果事件a和b在不同进程内，比如Pa和Pb。

设VCa = [m ,n], VCb = [s, t]。

因为VCa < VCb，所以m *≤* s，所以必然在不早于a之前和不晚于b之后的时间内，Pa向Pb发送了消息，否则Pb对Pa的计数器得不到及时刷新，s就不会小于m。

![图三](/img/in-post/2020-09-12-vector-clock/post-vector-clock-proof1.jpg)

实际上，可以分为以下几种情况：

![图四](/img/in-post/2020-09-12-vector-clock/post-vector-clock-proof2.jpg)

1. 当a = c且d = b，易得a → b。
2. 当a = c且d → b，由传递性，得a → b。
3. 同样对于d = b且a → c的情况。
4. 当a → c且d → b，根据进程内的算法逻辑性和传递性，也很容易得出结论。

综上: VCa < VCb 推导出 a → b 得证。

向量时钟将Lamport逻辑时钟的全序时钟值改成了向量时钟的偏序关系，可以准确刻画事件的顺序，



# 因果关系与光锥

向量时钟可以反映事件之间的因果关系，下图中对于中间的B4事件，左边蓝色区域内的事件都是B4事件的“因”，右边红色区域内的事件都是B4事件的“果”。而上下白色区域内的事件则属于和B4事件没有因果关系的平行事件。

![图五](/img/in-post/2020-09-12-vector-clock/post-vector-clock-wikipedia.png)

这让我想起了相对论中的[光锥](https://zh.wikipedia.org/wiki/光锥)。光锥是[闵可夫斯基时空](https://zh.wikipedia.org/wiki/%E9%96%94%E8%80%83%E6%96%AF%E5%9F%BA%E6%99%82%E7%A9%BA)下能够与一个单一事件通过光速存在因果联系的所有点的集合，简单地说就是事件所能影响的时空范围，它的图形如下：

![图六](/img/in-post/2020-09-12-vector-clock/post-vector-clock-light-cone.png)

时空是由三维空间和一维时间构成的，由于人类是三维空间中的生物，没办法感受四维时空，所以光锥图中把三维的空间简化为了二维xy平面，时间是z轴。空间平面中的每一个维度都可以向正负两个方向移动，而时间不可能倒退，只能向正方向移动。

以当前观测者为原点，因为任何物体的运动速度都不可能超过光速，所以它产生的事件造成的影响也不可能超过光速。这个事件每秒钟在二维空间平面中的传播范围也不可能超过光每秒钟走过的地方。所以光锥呈现一个圆锥的形状。光锥分为未来光锥和过去光锥，未来光锥指当前发生的事件对未来造成影响的范围，而过去光锥指过去多大时空范围的事件能对现在造成影响。

![图七](/img/in-post/2020-09-12-vector-clock/post-vector-clock3.png)

上图中，把当前时刻的太阳作为光锥的坐标原点，因为太阳光到地球需要8分钟，所以当前时刻太阳发出的光需要8分钟后才能对地球造成影响。这就是未来光锥，也就是当前时刻的事件对未来造成影响的范围。

![图八](/img/in-post/2020-09-12-vector-clock/post-vector-clock4.png)

如果把当前时刻的地球作为光锥的坐标原点，那么只有8分钟之前的太阳光才会对地球造成影响，8分钟前的太阳处于过去光锥的范围内。而4分钟前的太阳不会对现在的地球造成影响，它处在过去光锥之外。

> “宇宙中的任何事件都只能影响它的将来光锥内的物体，凡事在事件的将来光锥外的物体不会受该事件的任何影响。”



# 向量时钟的应用

向量时钟有什么作用呢？举两个使用例子。

作用之一是检测数据冲突。分布式数据库中数据存在多个副本，每个副本都可能提供写操作，如果两个副本都对同一个数据进行写操作，就可能造成冲突，向量时钟可以检测这种冲突，但不能解决冲突。

![图九](/img/in-post/2020-09-12-vector-clock/post-vector-clock5.png)

上图中 a → b → c → d → e 都有因果关系，不会产生冲突，但是e和f是并发关系 e \|\| f，这两个会有数据冲突，向量时钟可以检测这种冲突，但是需要应用层来解决冲突。

作用之二是确定事件间的因果关系。举个微信朋友圈的例子。微信朋友圈不同地域的用户可能在不同的服务器上，通过网络消息来同步朋友圈好友发的图片以及对应的评论。

![图十](/img/in-post/2020-09-12-vector-clock/post-vector-clock6.png)

上海小王发了一个风景图片到朋友圈，香港Mary看到后评论了问“这是哪”，上海小王回复评论“梅里雪山”。由于网络原因，对评论的回复先到了加拿大的服务器，评论后到，导致加拿大Kate先看到回答“梅里雪山”，后看到提问“这是哪”。

![图十一](/img/in-post/2020-09-12-vector-clock/post-vector-clock7.png)

使用向量时钟，VC(c) < VC(e)，因此 c → e，加拿大的服务器知道c发生在e前面，可以对评论和回复正确排序后显示。



# 参考

[Logical Time](http://www2.imm.dtu.dk/courses/02220/2015/L7/Logical_Time.pdf)

[Vector Clock](https://en.wikipedia.org/wiki/Vector_clock)

[Timestamps in Message-Passing Systems That Preserve the Partial Ordering](https://zoo.cs.yale.edu/classes/cs426/2012/lab/bib/fidge88timestamps.pdf)

[逻辑时钟 - 如何刻画分布式中的事件顺序](https://writings.sh/post/logical-clocks)

