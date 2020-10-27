---
layout:     post
title:      Write-Optimized Dynamic Hashing for Persistent Memory
subtitle:   Fast '19 CCEH
date:       2020-10-20
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Hash索引
---

> 本文提出的CCEH，是我印象中第一篇将如何让动态哈希适用于PCM的文章，文章开头DISS了一下Level Hash，可以在被背景这一块提到。Our implementations of CCEH, linear probing (LINP), and cuckoo hashing (CUCK) are available at https://github.com/DICL/CCEH. For path hashing (PATH) and level hashing (LEVL), we downloaded the authors’ im- plementations from https://github.com/Pfzuo/Level-Hashing.

## 1.BackGround

- 静态哈希适用于一些事先知道哈希表大小的场景，然而很多场景下，是不知道哈希表大小的，比如一些常见的文件系统和数据库系统
- PFHT，Path Hash以及Level Hash都是静态哈希，其中PFHT和Path Hash的查找时间并不是O(1)，Level哈希的rehash操作并没有像它文章一样描述的那么有用。
  - 比如level哈希现在有100个记录在top level，50个条目在bottom levle，扩容到600，需要两次，相当于一共哈希了150个条目。假设传统的哈希表假设有150个条目，现在需要扩容的话我们一次性扩容到600，相当于也只哈希了150个条目，这样子一来level哈希只是将全表扩容分到了两次操作里面。
  - 还一个问题，那就是level哈希插入的时候有没有查看数据是否有重复的情况，比如我现在插入key,val1,接下来又插入key，val2。这样的话，如果不检测key是否一样的话，这样子就会有新旧数据的差别了，这也是一个问题。

## 2.Cacheline-Conscious Extendible Hashing

#### 2.1 Three Level Structure of CCEH

在字节寻址的PM上，失败原子写的大小是8B，但是*CPU*和*Memory*之间的数据传输粒度是一个*cacheline*的大小，一般为64B。我们设计的bucket大小一般为一个 *cacheline*大小，假设key和val都是8B的话，那我们这样子的话就存储不超过4个键值对。假设一个目录对应一个bucket那这样的话，目录就会显得很大，作者这里因此设计三层结构，多个桶组成一个段，一个目录指向一个段。这样子的话原先可能只要搜索一个桶，现在要搜索一个段里面的所有桶，搜索效果会很差，这里作者的解决方法是一个key的MSB位置作为目录的索引而key的LSB位作为桶的索引，这样子就保证了每次访问只要访问两个cacheline即可，一个是目录所在的cacheline，一个是桶所在的cacheline。

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201020155636481.png" alt="image-20201020155636481" style="zoom:50%;" />

For Example：这里的话，我们使用前两位作为目录的索引，这个是根据目录的全局深度来决定的，为10，这样子的话我们就可以对应到						  Segment3，之后查看Bucket index，索引到了Bucket N-1，这样子的话我们就可以访问Bucket N-1里面所有的键值对，						  找到对应的即可。

#### 2.2 Failure-Atomic Segment Split

对于动态哈希，当目录分为分割和扩容两个操作，这里的话，我们讨论一下分割操作，分割操作发生的情况是当我们需要往一个目录插入数据的时候，这个目录满了，但是这个目录的本地深度小于全局深度，那么我们就只需要进行分割操作就行而不需要进行扩容操作。

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201020162144365.png" alt="image-20201020162144365" style="zoom:50%;" />

对于分割操作，假设我们现在需要插入数据  $1010...11111110_{(2)}$ 到图(a)中，我们索引10，找到了对应段，再根据$11111110$索引对应的bucket，我们发现bucket已经满了，那我们可以执行分割操作，将$11$开头的数据分割出去

- 首先将*Segment3*中的$11$开头的数据赋值到*Segment4*中
- 让目录$11$的指针指向*Segment4*，并且更新*Segment4*的本地深度
- 更新*Segment3*的本地深度

(<strong><mark>从右到左更新目录</mark></strong>，原因第三章说)

对应于分割操作，那肯定有合并操作，合并操作的步骤，和分割步骤相反，步骤如下

- 首先将*Segment4*中的数据迁移到3中
- 将目录11的指针指向*Segment3*,更新*Segment3*的本地深度
- 更新*Segment4*的本地深度

(<strong><mark>从左到右更新目录</mark></strong>，原因第三章说)

#### 2.3 Lazy Deletion

传统的哈希在赋值操作结束之后会将旧的历史数据，因为一次page write可以将本地深度也改变，同时也可以将数据删除。但是CCEH并不会，我们可以看出在图(b)中，分割成功的话，*Segment3*中$11$开头的数据会被设置为无效，当现在插入一个$10$开头的数据，发现第一个数据$1011...11111110$是有效地,但是第二个数据$1101...11111110$是无效的.

> 这里我猜测作者的实现方式是插入一个桶的时候，除了看这个位置的数据是否为11或者为空，满足任何一个条件的话都可以进行插入.这里也说明了从右到左更新条目的重要性，如果先把3的本地深度变为2的话，这样子的话11数据无效，但这个时候crash的话，那么11开头的数据就丢失了。

下图就是不使用Lazy Deletetion，需要复制两个Segment。

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201020205039708.png" alt="image-20201020205039708" style="zoom:50%;" />

下图使用*Lazy Deletion*，将本地深度修改为2，暗示着迁移出去的数据时无效的

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201020210957727.png" alt="image-20201020210957727" style="zoom:50%;" />

#### 2.4 Segment Split and Directory Doubling

> 这一小节首先阐述了一下为什么要用MSB做段的索引而不是用LSB

我们可以通过下面的图看出来LSB和MSB的区别

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201020211834418.png" alt="image-20201020211834418" style="zoom:50%;" />

我们来看图(b)，这是LSB的结果，对于传统的基于DISK的动态哈希来说，前面的目录在一块连续的区域里面，LSB生成目录在新的连续的块中，我们只需将新生成的目录追加在旧的BLOCK之后就行。之后来看图(c)，这是MSB的结果，我们是在中间插入新的目录，这样子的话原先的没脏的目录中间会插入脏的目录的。这样子的话使得块里面的所有的页都是脏的。

然而对于我们的PM来说，并不能复用前面的目录，因为所有的目录项需要存储在连续的内存空间中，我们需要重新划分一块新的内存，从而将新的目录放过去。<mark><strong>这就可以说明,在PM中利用不了LSB的优势</strong></mark>.也就是说无论使用LSB和MSB，目录的加倍需要的cflush是一样的。

> 我们可以看到使用MSB的时候，两个目录是相邻的，当需要进行段分割的时候，目录相邻的话可以很好的利用局部性，显然段分割的频率要明显优于目录分割，所以还是使用MSB做段索引，当然这还有个好处，那就是使用MSB可以有利于恢复。

## 3.Recovery

#### 3.1Insert And Delete

失败原子写的大小为8B。我们一般写入key，val键值对的顺序是先写val，再写key。如果再val和key之间崩溃的话，这样子的话，只插入了val，根据key我们索引不到这个位置，因为比较操作是根据我们传入的key索引段号和桶号，再一一比对key，但这时候key没写入所以这个位置依然无效。如果先写key再写val的话，这样子的话，key写进去了，我们索引到了这个位置，比对key也一样，但是发现val不在，这就导致了部分更新。如果key是8B的话，就刚刚好通过一个clwb指令写入即可，如果大于8B的话，比如key是32B。因为我们使用的是MSB位索引段位，我们先写入后24B,再原子写入8B。当我们前8B没有成功写入的话，段号就可能不对，这样子的话相当于Lazy Deletion，这里面的数据相当于无效数据。当我们要删除一个key的时候，只需要将key的8B改写就行。

> 这里的话，我觉得使用token可能会更加好，按照作者的说法，当key大于8B的话，我们需要一次clwb(value),clwb(24B key) menfence clwb(8B key)。这里的话使用token的话，clwb(key),clwb( value),menfence clwb(token),大概这样子的顺序，但是我觉得使用token查找起来可以省略一下对比key的操作,在Lazy Delete那里只需要将所有Token变为0就行，然后和本地深度的元数据一起clwb就行，这里建议采用失败原子写，元数据不超过8B比较好。

#### 3.2Recovery

<img src="https://gitee.com/hustdsy/blog-img/raw/master/%E6%88%AA%E5%B1%8F2020-10-21%2015.01.02.png" alt="截屏2020-10-21 15.01.02" style="zoom:50%;" />

这里作者的伪代码讲实话，我没看太懂，讲一下我自己的理解吧。首先将分割的扩容操作，我们是从右到左进行扩容的，扩容之后旧段和新段的L都应该+1，这个意思就是我们应该先更新旧段再更新新段，那么旧段更新成功的话，其L会比新的大，我们从左到右遍历目录，比较最右边的L和它的大小，比如我现在在S2，Directory[8]处，它比Directory[11]的L要小，但是按道理来说应该一样，所以开始遍历，将8-11之间的目录全部指向一个Segment然后更新L为2。

<img src="https://gitee.com/hustdsy/blog-img/raw/master/%E6%88%AA%E5%B1%8F2020-10-21%2015.19.02.png" alt="截屏2020-10-21 15.19.02" style="zoom:50%;" />

下面来看一个缩小的例子。这里的话，我自己画图，从左到右遍历，左边深度为1，应该由两个条目指向这个段，从左到右遍历，看到11的本地深度为2，不对，所以改为1.

## 4.Concurrency and Consistency Model

对于写，桶和段都加写锁，如果使用Lazy Delete的话就不能实现无锁读了，因此采用COW，可以实现无锁读，这里讲一下原因

- COW(*Copy On Write*)：为了实现无锁查询，我们使用cow的方式，在没有将指针指向两个新分割的段之前 线程都是访问旧段。我们采用一个引用计数，查看有多少线程访问了这个旧段，当为0的时候，才释放旧段。
- Lazy Delete:如果使用原地更新，lazy Delete的话，那读需要加锁。因为假设我现在一个读请求段号为11，但这个时候11和10都是一块的，指向10，现在有个分裂段的请求，将1的数据迁移到新段了，此时又有一个写请求过来把原先的11的数据改成了10，这样子的话 原先的读请求就失效了。

## 5.Experiments

4个Intel Xeon Haswell-EX E7-4809 v3处理器(8核、2.0GHz、8×32KB指令缓存、8×32KB数据缓存、8×256KB L2缓存、20MB L3缓存)和64GB DDR3 DRAM的工作台上进行实验。由于字节寻址的持久主存还不能使用，我们使用Quartz来模拟持久内存，这是一种基于DRAM的PM延迟仿真器[9,41]。为了模拟写延迟，我们在每个clflush指令后注入停顿周期。

#### 5.1Quantification of CCEH Design

作者使用传统的动态哈希方案EXTH(LSB)和CCEH(LSB)->使用LSB做段索引，CCEH(MSB)->使用MSB做段索引，做实验进行对比。默认一个桶大小为256B

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201022112448048.png" alt="image-20201022112448048"  />

- 当桶大小为256B是，CCEH(LSB)和EXTH(LSB)的性能差不多

- 随着Segment大小的增加,EXTH的clflush指令次数逐渐变小，这是因为段分割和目录分割次数变少了。但是插入性能和查找性能变差了，因为查找一个数据的时候可能需要查询更多的桶。而CCEH的插入和查找性能却提升了，这是因为它插入的时候会去索引在哪个桶，因此不会访问多个桶。256KB的时候，clflush指令增加我的理解是数据太多，分裂一次的代价太大了。还一个点就是CCEH的探测距离也逐渐变小随着段的增大。

  <mark>This is because the larger the segment size, the more bits CCEH uses to determine which cacheline in the seg- ment needs to be accessed</mark>

- CCEH(LSB)和CCEH(MSB)相比起来，MSB的性能要优于LSB，因为分割段的时候MSB很好的利用了局部性。其访问的目录是连续的，而LSB访问的目录是零零散散的。

#### 5.2 Comparative Performance

接下来的实验，作者采用16KB的桶大小，并且CCEH采用MSB。和线性哈希（LINP）,Cuckoo哈希（CUCK）,path hashing（PATH）和level hashing(LEVL)做比较。CCEH(C)表示分裂的时候采用的是copy on write,CCEH表示的是Lazy Deletion

| LINP | 当负载因子达到95%的时候扩容                      |
|-------------------------------------------------------------------------|
| CUCk | 当进行16次替换之后依然找不到空位置的话，全表哈希 |
| PATH | 使用8层，可以达到最大的92%的负载因子             |

下面来看一下结果：执行插入操作 

![image-20201022150228873](https://gitee.com/hustdsy/blog-img/raw/master/image-20201022150228873.png)

- CCEH性能最好，CCEH(C)随着写延迟的增大，性能逐渐比CUCK和LINP差
- PATH和Level的Rehash时间是最差的，因为Level在Rehash的时候需要删除以前的数据，会带来一系列的clflush指令的消耗。而CUCK和LINP是申请一段新区域，先把数据复制到新区域，再整个释放旧区域。当崩溃的时候，释放新区域，再rehash一遍就行。Level Hash和Path Hash都可以这样子，所以作者写了改写了Path和Level。（M）
- 注意到LEVEL(M)的rehash时间和CCEH(C)差不多，但比LINP和CCEH差，这主要是因为LEVEL两层结构的设计，以及它单个桶的设计，这会造成更多的cacheline的访问。(这里我的理解是CCEH把很多bucket放在一个level中，而level hash的桶不是）

#### 5.3Concurrency and Latency

![image-20201022170635689](https://gitee.com/hustdsy/blog-img/raw/master/image-20201022170635689.png)

- 对于图(b)我们可以看出来，由于静态哈希的全表散列的影响，以及锁争用，可伸缩性是很差的。
- 对于搜索性能，图(c)，这是正向搜索，CCEH(C)是最好的，LEVEL的搜索性能很差，首先它也探测四个桶，这四个桶在四个不同的位置，无法很好的利用内存并行性，并且LEVEL使用小尺寸的桶，相当于桶级别的哈希，一个桶多个槽。这进一步增加了cacheline的访问次数

![image-20201022171908140](https://gitee.com/hustdsy/blog-img/raw/master/image-20201022171908140.png)

- 对于负向搜索，LINP和PATH是最差的，因为他们搜索的项目数不固定。
- 大多数哈希方案的插入和搜索吞吐量对哈希表的大小不敏感。然而，我们观察到CCEH的吞吐量由于分层结构而线性下降。当CCEH索引1600万条记录时，目录大小只有1兆字节。由于目录比段更频繁地被访问，所以它在CPU缓存中的概率更高。但是，当CCEH索引2.56亿条记录时，目录大小变为16兆字节，而所有段的总大小为8兆字节。考虑到我们测试台机器的LLC大小为20 MBytes，目录的LLC缺失率随着目录大小的增加而增加。因此，当索引超过6400万条记录时，CCEH的搜索性能与LEVL和CUCK相似，CCEH和LEVL(M)之间的吞吐量差距缩小

