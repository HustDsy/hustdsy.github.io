---
layout:     post
title:      Lock-free Concurrent Level Hashing for Persistent Memory
subtitle:   ATC'20 Clevel
date:       2020-10-23
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Hash索引
---

> Clevel主要实现无锁的rehash操作，方法我们拭目以待

## 1.Introduction

1.*P-CLHT*和*Level Hash*在进行*rehash*操作的时候需要对全表进行锁定

2.CCEH在分割段的时候会对段进行锁定，当对目录加倍的时候，需要对目录进行锁定

![image-20201026104850837](https://gitee.com/hustdsy/blog-img/raw/master/image-20201026104850837.png)

 其中Duplication和Missing代表的意思是

<strong>Duplication</strong>:一个对某个项使用单槽锁的插入线程不能阻止其他线程将具有相同键的项插入到其他候选位置，因为一个项有16个槽(每个级别有2个候选桶)用于存储

<strong>Missing</strong>:由于条目移动，当我查找这个桶的时候，它把本应该在这个桶的数据移动到其它位置去了，导致查找失败。

![image-20201026105824210](https://gitee.com/hustdsy/blog-img/raw/master/image-20201026105824210.png)

CCEH在写入的时候没有考虑插入的数据是否已经存在因此可能也存在重复项，而且我觉得CCEH的内存利用效率应该算挺好的，因为它桶之间设计了平衡策略，但是如果探测太多的桶的话，务必会增加搜索距离，所以CCEH作者折中了一下。

## 2.The Clevel Hashing Design

Clevel Hashing的设计主要是为了解决三个问题

(1) *How to support concurrent resizing operations without blocking the queries in other threads?*

(如何在不阻塞查询操作线程的情况下进行并发的扩容操作)

 (2) *How to avoid lock contention in concurrent execution?*

(如何避免并发情况下的锁争用)

 (3) *How to guarantee crash consistency with low overheads?* 

（如何低开销的情况下保证崩溃一致性)

#### 2.1.The Clevel Hashing Index Structure

###### 2.1.1 Dynamic Multi-level Structure

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201028100200445.png" alt="image-20201028100200445" style="zoom:50%;" />

Clevel和level很像，也是多重结构，与Level Hash不同的是，每个level会由一个链表来进行管路它的每个桶大小是8个slot，而Level Hash是4个，并且CLevel去掉了一次移动的操作，换句话说，一次移动会增加PM的一次额外写，因此作者去掉了，但是这就会让负载因子降低，为了弥补负载因子的损失，作者将桶大小扩大了一倍，而且每个slot存储的是8B的指针，因此刚好一个slot大小就是一个cacheline的大小。

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201028100920379.png" alt="image-20201028100920379" style="zoom:50%;" />

###### 2.1.2 The Support for Concurrent Resizing

Clevel有一个全局的Context，它包括两个指针以及一个flag。其中last_level指向最底层的level，first_level指向最上层的level，flag用来标识Hash Table是否正在进行rehash操作。每个线程都保留了一个<strong>指向全局Context的指针ptr</strong>。*Resizing*操作主要包括五个步骤：

1. 每个线程Copy一份指向*Context*的指针副本。

2. 解引用指针，读取旧的*first_level*的bucket的大小，然后申请一块新的空间$L_{new}$，大小为旧*first_level*的两倍（使用CAS）

   1. 这里CAS比较的是<strong>ptr</strong>，保存的旧值的*context*,但是可能其它线程已经添加了新的$L_{new}$，新的Context中的first_level指向的是新的L_new了。这时候不需要管，直接继续第三步就好了。

3. 使用Cow+CAS更新*Context*，首先复制一份*Context*，使得旧的first_level指针指向新分配的新的空间$L_{new}$，并且将flag设置为true.

   1. 这里的CAS我觉得是看first_level指针是否指向的同一个数据，保存的旧值是以前的first_level，假设有线程已经更新了first_level，比较现在指向的first_level和理想中分配的$L_{new}$大小，如果现在的大的话，那也无需操作，因为已经扩容完了，如果现在的小的话，我们需要把最大的$L_{new}$写入，继续CAS+Cow操作。

   <mark>前三步都是扩容操作，第四步开始是rehash操作</mark>

4. 重新散列最后一层中的每个项。重散列包括两个步骤:通过CAS将项的指针复制到第一级的候选bucket，然后删除最后一级的指针，如果找不到位置的话 rehash

5. 当重新哈希完成时，使用CoW + CAS原子地更新全局上下文中的最后一个级别和可选is_resizing(如果在调整大小之后只剩下两个级别)。如果CAS失败，如果当前上下文中的最后一层没有被修改，请再次尝试

