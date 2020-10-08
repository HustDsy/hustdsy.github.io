---
layout:     post
title:      Revisiting Hash Table Design for Phase Change Memory
subtitle:   PFHT
date:       2020-09-06
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Hash索引
---

## Revisiting Hash Table Design for Phase Change Memory

> PFHT(PCM Friendly Hash Table),它考虑了PCM的一些特定的属性，与现有的基于DRAM设计的Hash相比，它在性能，写次数和负载系数方面提供了很好的权衡。

## 1.背景

PCM具有写寿命有限以及读写不均衡的两大特点，因此限制写次数很有必要。

- $chained\ hashing$(链式哈希)：局部性差，对于插入和删除操作，会导致频繁的堆分配和删除(*heap allocation and de-allocation*)。

![image-20201007164855552](https://gitee.com/hustdsy/blog-img/raw/master/image-20201007164855552.png)

- *Linear probing* (线性探测)：负载因子越高，插入和寻找操作所需要的时间越久
- *Array hashing*(数组哈希)：有着很好的局部性，但是对于插入和删除操作仍然有着频繁的*realloc*操作

![image-20201007164834042](https://gitee.com/hustdsy/blog-img/raw/master/image-20201007164834042.png)

- *Hopscotch hashing*：局部性良好，并发性好，但是有级联写的问题。
- *Cuckoo hashing*：局部性良好，但是有级联写的问题

![image-20201007171533441](https://gitee.com/hustdsy/blog-img/raw/master/image-20201007171533441.png)

## 2.设计

#### 2.1 主要的组成部分

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201007171854357.png" alt="image-20201007171854357" style="zoom:50%;" />

主要包括*Main table*和*Stash*两大部分，下面介绍一下这两个部件。

*Main Table:*相当于一个二维数组，主要用来存储*Key-value*键值对。

*Stash*:一个附加存储，主要用来提高负载因子的，作者这里使用的是链式哈希进行实现，不使用线性探测的原因是查找性能差，不使用跳房子哈希和布谷鸟哈希是因为有着级连写。

#### 2.2设计的优点

*Load balanced inserts*:每个*key*有两个位置，*PFHT*并不是随机选择一个桶，每个桶都记录着它有多少个数据，这里的话*PFHT*选择数据少的一个桶插入。

*No cascading writes:*Cuckoo哈希插入的时候，当两个桶都满的话，会不停的替换。而对于*PFHT*只允许一次置换，它检测所有的条目看看可不可以移动到它另外的位置，如果有的话，那么就移动，没有的话，就将它插入Dash桶里面。

*High CPU cache utilization:*PFHT将散列到相同位置的键存储在相邻的bucket中，从而提高了CPU缓存性能。PFHT使用较大的桶来实现高负载系数。但是，使用大容量的桶会增加查找延迟。因此，为了在负载因子和查找延迟之间取得良好的平衡，PFHT将桶大小设置为两条缓存线。

## 3.评估

<strong>配置要求</strong>：a 6-core Intel Xeon 2.0 GHz machine with 6MB CPU cache and 24GB of DRAM memory. We evaluate all hash table designs on PCM using the<strong> Mnemosyne framework </strong>.

*WO(Write Once)*：一次性插入大量的键值对，达到给定的负载因子

*WH(Write Heavy)*：先使用*WO*达到给定的负载因子，然后连续的删一个数据再增加一个数据。

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201007195230371.png" alt="image-20201007195230371" style="zoom:50%;" />

<strong>结果分析</strong>:

*WO*查找：Cuckoo,Hopscotch >PFHT>Pro,Pro需要线性探测性能最差，PFHT可能需要查找附加存储stash桶，所以性能较差

*WO*插入：Pro>PFHT>Cuckoo>Hopscotch，主要是Pro和PFHT的写次数减少，Hopscoch需要维护位图性能最差，负载因子越大，差距				  越明显。

*WH*查找：Cuckoo,Hopscotch >PFHT>Pro，无效查找的时候Pro性能十分差，其它与WO差不多。

*WH*插入：Pro>PFHT>Hopscotch>Cuckoo 主要是Pro和PFHT的写次数减少，由于写次数频繁，Cuckoo,Hopscotch 和PFTH，Pro差距更				 加明显。

<strong>PFHT适用于插入操作频繁的场景</strong>

## 4.总结

*PFHT*通过舍弃空间，获得了良好的无效查找性能和写吞吐量。设计*hash*可以从以下几点考虑

*Large bucket size*：桶大小过大的话，容易导致查询速度很慢，但是可以缓存更多的键值对，这里的话大小最好和*cacheline*对齐。

*Multi-hash functions*：使用多个哈希函数可以提高写性能，因为有多个槽给KV对选择。但是，使用大量的函数会增加定位KV对的平均位置									数，因此会增加查找延迟。

*Use of a small stash*：可以提高负载因子，对于查找性能有一定的影响。

*Balanced design：PFHT*每次插入尽可能插入到负载较小的那个桶当中。





