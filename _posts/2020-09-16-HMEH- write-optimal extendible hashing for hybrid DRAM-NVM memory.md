---
layout:     post
title:      HMEH:write-optimal extendible hashing for hybrid DRAM-NVM memory
subtitle:   Hash索引
date:       2020-09-16
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
   - Hash索引
---

这是一篇学姐发的和Hash索引有关的论文，要好好看的啊！主要将关键部分，比如设计和一致性保证这两个问题。

<!--more-->

## *HMEH Design And implementation*

HMEH的架构如下

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/20200915192350.png">

#### *A.Two Structures of Directory*

由于目录查找比较耗时，作者使用DRAM存储目录，由于DRAM中的数据掉电会丢失，为了恢复，采用一个$Radix-tree$来进行恢复。这一部分比较简单，结构大致如下图。

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/20200915193144.png">

#### *B. Cross-KV Mechanism*

首先说一说$key$,$val$的持久化操作，$failure$-$atomic$ $unit$是8字节的， 为了达到奔溃一致性的目的，一般先写$val$,再写$key$。为了保持这个先后顺序，需要在两个写操作之间加一个屏障，也就是mfence指令。这个指令是耗时的。

<font color=blue>一般奔溃之后，检测这个键是否被成功写入，就是用key去查找,如果找的到的话，那就说明成功写入。上述方法刚好符合要求。如果先写key的话，找到了key。但事实上，val并没有被写入到NVM中</font>

作者将8字节的$key$和8字节的$val$分开组合成$(key1,val1),(key2,val2)$.

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/20200915201735.png">

这样子的话就可消除掉这个屏障了，因为如果我只是将第一个$(key1,val1)$写入的话，我们从Cross-KV中取出来key，看看是不是能够索引到同一个段，同一个桶。如果不可以的话，就说明是部分写入的。但是有些偶然情况，一些部分写入的项目会索引成功，因此在插入数据之前，检测所有部分写的情况，如果可以索引成功，那么对于这个项还是需要使用屏障。

<font color=blue>这样子的拆分Key Val会导致搜索性能有一定的下降，因此key和val被拆分了，所以在读取的时候需要将key，val都读出来。而往常的哈希只要比较key就行了</font>

但是去掉屏障所带来的性能提升可以弥补搜索性能的下降。因此此方案是可行的。

#### *C. Low-overhead In-place Segment Split*

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/20200915213414.png" style="zoom:50%">

这里的话主要是段分割操作，和目录分割操作。

1. 对于段的分割操作：<strong>先持久化分裂出来的段，再更新原先段</strong>，如果先更新原来的段的话，一奔溃。数据就没了。如果分裂出来的段更新时崩溃，可以重复操作。（使用屏障）。先把数据复制到新段，再更新LD并且使得目录指针指向它。
2. 对于目录的分割操作：同理，先持久化分裂出来的段，再去更新原先段，道理同上。(分配一个新的目录节点，分配一个新的段，将segment1的数据赋值到2，之后目录先指向2，010这个目录指向新的节点，更新本地深度)

(作者这里有一个减少持久性开销的方法 我没看懂)

#### *D. Improvement of Load Factor*

这里作者采用引入额外的stash桶，来进行处理一些碰撞。每个segment线性探测四个bucket，并且bucket大小是256B(根据测试报告)，然后当有额外的键需要插入这个segment，它会探测接下里的四个桶有没有额外的位置。如果都没有的话，那么就将数据放入stash桶中。这样子在提高负载率的同时也延迟了段和目录的分割操作。但是对于search操作来说，可能会增加时间。（stash也满的情况下，再开始分割操作）

<mark><strong>线性探测距离最好为4，不知道这个距离包不包括第一次查找到的桶本身</strong></mark>

#### *E. Recovery of Two Directories*

1. 正常关闭的情况下：我们将R-Tree正常刷新到了NVM中，这里查看图4。主要的是看全局深度和本地深度如何计算。全局深度就是树的高度，本地深度就是树到这个条目的路径长度。之后再建立FS即可

2. 对于崩溃的情况，只有叶子节点具有不一致性。这里也是根据全局深度和本地深度来进行处理的。当一个segment的本地深度等于全局深度的时候，一个条目对应一个segment，不同的时候，有	有$2^{GD-LD}$的条目指向同一个segment。直接遍历，如果在这个区间的条目不指向这个segment，那么就让他指向这个条目。伪代码如下：

   <img src="https://gitee.com/hustdsy/blog-img/raw/master/img/20200916100810.png">

## *CONCURRENT AND CONSISTENT HMEH*

#### *Mutex and Version Number for Directories.*

在FS-目录加倍的时候，需要同时更新全局深度和指向目录的指针。如果GD被更新了的话，但是目录指针并没有更新，那么根据新的key计算出来的目录可能会超出范围(这个很好理解的吧，<strong>比如你现在需要4位去进行索引，但是目录还是3位</strong>.如果目录指针被更新，但是GD还没更新。所以你现在还是根据GD的位去索引数据，可能导致搜索不到数据，甚至失败。作者在这里采用版本号来进行处理。GD的版本号和对应的目录的版本号一致的话才回去进行索引段的操作。

#### *Fine-grained Lock for Segment split and lock-free read.*

1. 由于读操作不设计更改操作，所以可以无锁化，但需要验证目标段的局部深度。具体来说，在搜索完成后，我们会检查LD是否已经已修改，如果是，则需要重试查询操作。这是因为更新后的LD表明目标段已被分割，因此，目标项可能无效，因为并发写可能修改了它。
2. 由于分割段的目录项保持不变，我们可以在准备一个新段后释放分割段的锁，以提高可伸缩性。但是，在这种情况下，插入时应该验证密钥是否属于旧段，如果不属于，则重复插入，直到成功为止。(通常情况下是等分割完成，再释放锁，作者这里把释放锁的步骤提前了)。

#### *Compare-and-swap (CAS) Instructions for Slots of Buckets*

这里我的理解，当执行写入操作的时候，使用KV作为CAS的版本号，这样子的话，就表明这个槽被我占了，当有其它写入操作的时候，就回去寻找其它的新槽。

## *PERFORMANCE EVALUATION*

#### *A.Experimental Setup*

1. 2-sockets，36-core的装有1.5TB的DCPMM，186GB DRAM  32MB的LLC(Last Level Cache)的 Linux server(内核版本5.0.0)
2. Ext4-DAX文件系统，OPtane DC使用APP Direct mode，使用clwb指令
3. 使用PMDK[38]中的libvmem库来支持构建在内存映射文件上的易失性内存池的传统分配接口。
4. 160 million随机整数最为工作负载。每个方案的hash table初始大小为2048个$key-value$项
5. 比较方案包括CCEH, Level hashing, persistent linear hashing and persistent cuckoo hashing

#### *B. Experimental Results and Analysis*

###### HMEH设计的灵敏度分析

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/20200916153515.png">

Fig 7(a):当$segment$过小的话，导致分裂次数过于频繁，但是$segment$过大的话，分裂时间急速上升，因此$size$大小在$4KB-16KB$之间   			   较好。

FIg 7(b):随着$segment$的上升，平均探测距离也之间变小(作者解释随着$segment$的变大，可以记录桶信息的位数变多，因此探测距离变   			  少(很奇怪，它这篇文章segment的结构画的及其糟糕，就是动态哈希的结构，画大了一点。。不是很清楚为什么变小)

Fig 7(c):横坐标表示$Stash$桶的数量，每个桶的大小为256B，可以看到随着$Stash$数量的增加，负载因子也在增加，但是搜索的吞吐量会 			  下降。在搜索吞吐量和负载因子之间做一个trade off，可以看到取值1-8的时候比较好。（$stash$越多，$segment$的大小也就越	         			   大）

<center><mark><strong>综上所述，HMEH的段大小设置为16KB比较好，其中stash的数量为4</strong></mark></center>

###### *Comparative Performanc*

1.1.6亿的随机写，并且分别记录目录操作和段操作所花的时间

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/image-20200916163027888.png" style="zoom:50%">

FS目录操作执行在DRAM中执行，并且RT目录重用节点。相比于之前FS目录放在NVM来说，节省了不少时间。

2.比较1.6亿数据的写延迟，分为$rehash$时间和写入的时间

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/20200916164130.png" style="zoom:50%">

3.比较负载因子，共享桶数量一样的情况下，HMEH优于CCEH

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/image-20200916165018571.png" alt="image-20200916165018571" style="zoom:50%;" />

4.并发性能，使用YCSB生成三种不同的工作负载，都有1.6亿个条目

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/image-20200916170033721.png" alt="image-20200916170033721" style="zoom:50%;" />

levl和P-LNP吞吐量没有变化，因为他们的全表散列带来巨大的延迟，并且阻止了其它线程的插入操作。

libcuckoo性能最差，因为受到锁操作的限制

由于HMEH的目录访问放在DRAM中，因此搜索性能最好。

尾延迟：对于LEVL和P-LINP，libcuckoo静态哈希来说，rehash的时候需要锁定整个表，平坦区域代表的是rehash所花费的时间

<img src="https://gitee.com/hustdsy/blog-img/raw/master/img/image-20200916170735506.png" alt="image-20200916170735506" style="zoom:50%;" />

5.恢复时间，包括恢复RT-Tree并且根据RT-Tree重建FS-Tree，均为毫秒级，瞬时恢复

![image-20200916171954626](https://gitee.com/hustdsy/blog-img/raw/master/img/image-20200916171954626.png)

