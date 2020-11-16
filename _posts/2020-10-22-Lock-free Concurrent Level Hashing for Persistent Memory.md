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

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201028100200445.png" alt="image-20201028100200445" style="zoom:50%;" />

Clevel有一个全局的Context，它包括两个指针以及一个flag。其中last_level指向最底层的level，first_level指向最上层的level，flag用来标识Hash Table是否正在进行rehash操作。每个线程都保留了一个<strong>指向全局Context的指针ptr</strong>。*Resizing*操作主要包括五个步骤：

1. 每个线程Copy一份指向*Context*的指针副本。

2. 解引用指针，读取旧的*first_level*的bucket的大小，然后申请一块新的空间$L_{new}$，大小为旧*first_level*的两倍（使用CAS）

   1. 这里CAS比较的是<strong>ptr</strong>，保存的旧值的*context*,但是可能其它线程已经添加了新的$L_{new}$，新的Context中的first_level指向的是新的L_new了。这时候不需要管，直接继续第三步就好了。

3. 使用Cow+CAS更新*Context*，首先复制一份*Context*，使得旧的first_level指针指向新分配的新的空间$L_{new}$，并且将flag设置为true.

   1. 这里的CAS我觉得是看first_level指针是否指向的同一个数据，保存的旧值是以前的first_level，假设有线程已经更新了first_level，比较现在指向的first_level和理想中分配的$L_{new}$大小，如果现在的大的话，那也无需操作，因为已经扩容完了，如果现在的小的话，我们需要把最大的$L_{new}$写入，继续CAS+Cow操作。

   <mark>前三步都是扩容操作，第四步开始是rehash操作</mark>

4. 重新散列最后一层中的每个项。重散列包括两个步骤:通过CAS将项的指针复制到第一级的候选bucket，然后删除最后一级的指针，如果找不到位置的话 rehash

5. 当重新哈希完成时，使用CoW + CAS原子地更新全局上下文中的最后一个级别和可选is_resizing(如果在调整大小之后只剩下两个级别)。如果CAS失败，继续尝试。

## 3.Lock-free Concurrency Control

为了减少对共享资源的争用，作者提出了无锁的并发控制

#### 3.1 Search

查询操作主要面临两个问题，1）指针的解引用开销很高，由于clevel只保存指向真实的key的指针，一个一个读取指针进行遍历的话将会造成很多不必要的读。2）由于数据移动而丢失插入项。同时调整大小会移动到最后一层的项，因此，不带任何锁的搜索可能会遗漏插入的项。

<strong>Solution 1：</strong>对于第一个问题，作者采用的是类似于fingerprint的技术，这里称作为tag。对于每个key我们保存一个16位的tag，tag的计算可以去hash(key)的前2B的字节的数据（相当于保存部分key）。这样字的话，读取先比较tag，tag再对的话，就去读取指针，然后解引用读取key。这里不使用额外的元数据进行存储tag。因为指针只需要48位即可，也就是6B的指针大小即可。因此作者使用指针的前2B去当做tag。

<strong>Solution 2：</strong>对于第二个问题，作者采用的搜索策略是自底向上搜索，因为扩容操作是将下面的数据移动到上面。但是有一种会出现丢失数据的问题。那就是其它线程，通过一个新线程将我们要搜索的数据重新rehash到它新申请的级别中，所以当我们所搜不到的时候，就会CAS比较Context内容是否一致，如果不一致的话，那就重新进行搜索操作。

## 3.2 Insertion

在查询之前首先检测有没有重复的数据，有的话就进行更新操作。没有重复数据并且有空位置的话，就原子写入指向key和value的指针。当没有空位置的时候就进行扩容操作。但是无锁操作可能会有两个问题

1）可能插入一样的数据，并发线程可能会将具有相同的key的项插入不同的槽中，这会导致删除和更新的失败

2）新插入的数据插入到旧的last_level中的话，旧的last_level重哈希之后被释放的话就会丢失数据。

<strong>Solution 1：</strong>对于第一个问题，作者在插入和删除操作中解决。

<strong>Solution 2：</strong>对于第二个问题，作者提出了一种

