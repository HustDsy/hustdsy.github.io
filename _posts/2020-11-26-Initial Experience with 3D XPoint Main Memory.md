---
layout:     post
title:      Initial Experience with 3D XPoint Main Memory
subtitle:   测试Optane的性能
date:       2020-09-06
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Optane Performance
---

> 这是一篇对OPtane进行性能测试的文章

## 1.实验配置

![image-20201126145230975](https://gitee.com/hustdsy/blog-img/raw/master/image-20201126145230975.png)

AEP机器如上图所示，一个AEP机器包含两个3D XPoint模块(256GB)分别连接到两个不同的CPU上。对于单个CPU来说，一个是本地模块，一个是远程模块。这也在意味着在AEP机器中存在NUMA效应。作者使用PMDK对3D Xpoint进行访问。主要是想测试以下几个方面的性能。

- <strong>写内容对Optane的性能的影响</strong>

   为了提高NVM的写性能和降低写能耗，文献中提出了各种技术，包括数据比较写。这些技术利用了这样一个事实，即所写的比特的一部分与原始内容相比是不变的。因此，可以将新内容与原始内容进行比较，并仅对已更改的位执行NVM写操作，从而避免不必要的重写。我们想检查这样的效果是否存在于真正的3D XPoint内存硬件

- <strong>访问的数据的总量</strong>

  ssd和hdd经常在设备中使用缓存来提高性能。如果工作数据集适合设备内缓存，应用程序将看到更好的性能。除了缓存之外，ssd通常还保留额外的闪存空间(例如，实际的容量比显示的容量多20%)，以便更好地组织新的写操作，以达到消耗均衡的目的。虽然NVM与ssd和hdd有显著的不同，但可以利用类似的技术。因此，有必要观察3D XPoint在不同数据访问量下的性能。

- <strong>内存访问的随机性</strong>

  数据库的操作一般显示为不同的内存访问模式，比如表扫描执行顺序访问，哈希或者B+树索引访问通常以随机方式访问内存位置。随机访问可以是<strong><mark>独立的，也可以不是独立的</mark></strong>。执行依赖随机访问的典型场景是跟踪链表。链表中下一个节点的内存地址存储在前一个节点中。因此，对下一个节点的访问取决于对前一个节点的访问。相反，在给定记录id的情况下检索一组记录会导致独立的随机访问，这可以由并行内存访问提供。研究不同的内存访问模式是很重要的

- <strong>是否使用clwb+sfence持久化数据</strong>

  一个存储指令(例如mov)在到达CPU缓存时完成。在那之后，何时以及以什么顺序修改的行被写回内存是由CPU缓存硬件控制的。不幸的是，CPU缓存是不稳定的。因此，为了保证崩溃恢复后NVM数据结构的一致性，必须使用特殊的指令(如clwb和sfence)来确保修改后的内容被写回NVM内存。我们想要量化这种特殊指令的开销。、

- <strong>NUMA效应</strong>

  通过控制CPU的亲和性以及内存绑定来进行测试

作者跑了一组内存测试。每一个测试都为DRAM和Optane分配相同大小的*Block Nums×Block Size*大小的内存。其中*Optane*的内存分配通过PMDK完成。然后对于分配的内存进行初始化（by writting every bytes）。在这之后，它测定指定的内存访问模式。并且使用gettimeofday用于测量执行所有内存访问所花费的时间。最后，它通过结合运行时间、块大小和块大小来计算访问延迟和带宽。我们测试以下内存访问模式:

- <strong>顺序读模式(seqread)</strong>:顺序读取*Block Nums×Block Size*大小的内存
- <strong>随机读模式(独立的)(randread)</strong>:内存被分成了块，每一块都有块大小的字节。随机读模式中分配了一个*Block Nums*大小的指针数组，数组中存取的块的内存地址，然后随机打乱数组项。通过数组中的指针读取块，从而执行独立的随机访问。在实验中随着块大小的增加，内存访问模式越来越接近顺序访问模式。
- <strong>随机读模式(依赖的)(depread)</strong>:类似于独立的随机读模式，依赖读模式将打乱之后的数组中的数据按照顺序构造一个链表。后面内存访问的地址依赖于前面的内存访问地址。从而得到依赖的随机读模式。
- <strong>顺序写模式(seqwrite)，随机写模式(独立的)(randwrite,depwrite)</strong>:这三种写模式的实现类似于三种读模式的实现，但是唯一不同的就是写分普通写和持久化写。持久化写代表使用clwb+sfence指令

## 2.实验结果

#### 2.1 写内容对写性能的影响

这次测试作者采用了三个初始值，分别为0xf0f0f0f0f0f0f0f0，0xaaaaaaaaaaaaaaaa，0x0000000012345678(8B)。数据访问的总量分别为8GB和32GB，然后进行的是顺序写和随机写模式的测试。每次修改1B的数据，横坐标代表的是写入的数据与初始数据有多少B的数据不同，纵坐标代表平均刷新一次cacheline的延迟测试结果如下：

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201126171551584.png" alt="image-20201126171551584" style="zoom:50%;" />

可以看出，无论什么写入模式下，写性能和写内容是无关的。

#### 2.2 Varying Total Amount of Data Accessed

 这个实验中，作者测试了访问数据总量在1GB到256GB的情况下顺序读，随机读(依赖，非依赖);顺序写，随机写(依赖，非依赖)的带宽和延迟情况。块大小统一设置为64B，等于cacheline的大小。图3 (a)-(f)显示了DRAM (mem)和3D XPoint内存上的seqread、randread和depread的带宽和延迟。图4 (a)-(f)显示了DRAM (mem)上的seqwrite、randwrite和depwrite、3D XPoint内存和带有clwb+sfence指令的3D XPoint(持久化)的带宽和延迟。请注意，写性能图(图4 (a)-(f))中的y轴是对数刻度。

（带宽和延迟指的是一次IO操作的，每次IO操作的块大大小）

![image-20201127101431710](https://gitee.com/hustdsy/blog-img/raw/master/image-20201127101431710.png)

<strong>DRAM VS 3D XPoint读取性能</strong>

- 在1GB-256GB这个区间中，seqread要慢1.3 - 1.6倍，randread要慢1.2-2.3倍，depread要慢1.3-2.5倍。
- 在32GB-256GB这个区间中，seqread要慢1.5 倍，randread要慢2.1-2.3倍，depread要慢2.3-2.5倍。

depread确保一次只能进行一次读取（因为下一次读取依赖于上一次读取到的内存地址）。另一方面，在randread和seqread中，CPU可以并行执行多个读取操作。randread和seqread的性能也反映了设备支持并发内存访问的能力。因此，如果我们考虑内存读取的延迟，3D XPoint比DRAM慢2.3 - 2.5倍(<strong>因为不考虑并发</strong>)。

![image-20201127102858858](https://gitee.com/hustdsy/blog-img/raw/master/image-20201127102858858.png)

<strong>DRAM VS 3D XPoint写性能</strong>

   PS:这里比较的是3DX point和mem，不涉及clwb+sfence的持久化操作

- 在1GB-256GB这个区间中，seqwrite要慢1.2 - 1.9倍，randwrite要慢1.1 - 2.5倍，depwrite要慢1.3 - 2.3倍
- 在32GB-256GB这个区间中，seqwrite要慢1.4 - 1.7倍，randwrite要慢2.2 - 2.5倍，depwrite要慢2.2 - 2.3倍。

<strong>Write performance: persist vs. normal 3D XPoint write.</strong>

从图4中，我们可以看到持久化比针对seqwrite和randwrite的普通3D XPoint写入要糟糕得多。这是因为普通的3D XPoint写都缓存在CPU缓存中。对3D XPoint的实际写操作可以在后台重新排序并并行执行。相反，persist使用clwb强制写入操作立即进入3D XPoint内存。sfence进一步确保在下一个数据块是在前一个写入操作完成之后写入的。由于这组实验中的块大小为64B，所以在持久化实验中不会有并发的缓存线写入。

- 因此，与普通的3D XPoint写入相比，seqwrite的persist慢了10.7 - 14.2倍，randwrite慢了2.9 - 5.1倍。如果我们只考虑大于或等于32GB的数据大小，那么与普通的3D XPoint写入相比，持久化seqwrite要慢12.1-14.1x, randwrite要慢3.0-3.4x。有趣的是，最大的减速发生在数据很小的时候，这是缓存机制最有效的地方。
- 对于depwrite，持久化曲线和3D XPoint曲线几乎相互重叠。对于depwrite，在写操作中没有时间或空间局部性，并且在内存系统中没有并发的写操作。因此，CPU缓存对depwrite的好处有限。因此，depwrite和persist具有相似的性能。

<strong>Performance varying data size</strong>

随着数据访问总量从1GB增加到256GB，所有图中DRAM曲线都保持相对平坦，而3D XPoint的随机依赖性读写曲线变化明显。我们通过将延迟增量除以相邻点的增量数据大小来计算变化的斜率。我们看到在所有情况下，在32GB之后坡度变得相对较小。因此，我们怀疑在3D XPoint模块中使用了缓存等技术来提高性能。当访问的数据量为32GB或更大时，这种影响就不重要了。

<strong>3D XPoint: Read vs. Write.</strong>

与读取相比，3D XPoint的写入对于顺序访问要慢1.4倍，对于随机访问要慢2.3倍，对于依赖访问要慢1.2倍。总的来说，NVM读写的非对称效果是适度的:3D XPoint写操作比读操作多2.3倍。

#### 2.3 Varying Block Size

作者先clwb块的每一行 再发送sfence指令

![image-20201127113600347](https://gitee.com/hustdsy/blog-img/raw/master/image-20201127113600347.png)

- 首先，在seqread和seqwrite的情况下，DRAM和3D XPoint的性能基本稳定。在测试程序中，内部循环处理一个块中的数据，外部循环处理每个块。测试程序对所有被访问的数据执行顺序的内存访问。内部循环中的块大小没有太大区别。
- 其次，在randread和randwrite情况下，DRAM和3D XPoint的带宽随着块大小的增大而增大。这是因为块的大小越大，访问的随机性就越小，顺序也就越好。因此，randread (randwrite)性能越来越接近seqread (seqwrite)性能。
- 最后，持久化写操作的性能也会随着块大小的增加而增加。然而，有趣的是，在depwrite的情况下，当块大小超过512B时，persist的曲线和normal 3D XPoint写入的曲线会出现分歧。当访问模式变得越来越类似于顺序访问时，就会出现clwb的代价（出现了空间局部性和时间局部性）。

#### 2.4 NUMA effect

![image-20201127114428326](https://gitee.com/hustdsy/blog-img/raw/master/image-20201127114428326.png)

正如预期的那样，DRAM和3D XPoint的远程内存访问性能都低于本地内存访问性能。系统中的两个cpu通过QPI链路连接。远程内存访问必须通过两个CPU之间的QPI链接，然后使用远程CPU上的内存控制器来执行内存访问。这扩展了内存访问延迟并减少了内存带宽。

对于seqread、seqwrite、randread和randwrite, 3D XPoint的延迟差异大于DRAM。然而，延迟比率都是相似的。在这些情况下，CPU可以发出多个并发内存访问。由QPI支持的带宽是决定因素。这是QPI的一个特点。因此，DRAM和3D XPoint的延迟比率是相似的。

对于depread, DRAM和3D XPoint的延迟差异是相似的，但是DRAM的延迟比3D XPoint的延迟比要大。在这种情况下，没有并发内存访问。主要的因素是QPI造成的额外延迟。因此，DRAM和3D XPoint的延迟差异是相似的。延迟率=(本地延迟+差异)/本地延迟= 1 +差异/本地延迟。由于3D XPoint的本地延迟大于DRAM，因此其延迟比较小。

对于3D XPoint持久化写操作，延迟差异和延迟比率都大于DRAM和3D XPoint常规写操作。远程持久存储比本地持久存储开销大得多。clwb+sfence对远程访问的影响大于对本地访问的影响。我们猜测，在远程情况下，特殊指令需要两个cpu协作，执行某些额外的操作，从而导致额外的开销。

