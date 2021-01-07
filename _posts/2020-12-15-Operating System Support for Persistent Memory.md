---
layout:     post
title:      2.Operating System Support for Persistent Memory
subtitle:   PMDK
date:       2020-12-15
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - PMDK
---

> 第二章主要介绍持久性内存的架构，内容比较多

## **1.Persistent Memory Characteristics**

- 持久性内存的性能比NAND好的多，但可能比DRAM慢
- 持久性内存是持久的，它的寿命比NAND flash好几个数量级
- 持久性内存的容量比DRAM DIMMS大的多，并且可以在相同的内存通道上共存
- 支持持久性内存的应用程序可以就地更新数据，而不需要对数据进行序列化或者反序列化
- 持久存储器是字节寻址类似内存。应用程序只能更新所需的数据，而没有任何读-修改-写开销。（Optane DC访问粒度为256字节，还是有一定给的写放大的）
- 数据是CPU缓存一致的
- 持久内存提供直接内存访问(DMA)和远程DMA (RDMA)操作
- 写入到持久内存中的数据当掉电的时候是不会丢失的
- 在检查完权限之后，位于持久性内存中的数据可以从用户空间上进行访问。数据路径中没有内核代码，文件系统页缓存或者中断

## 2.Cache Hierachy

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201215111221276.png" alt="image-20201215111221276" style="zoom:50%;" />

以Intel体系结构为例。CPU缓存通常用三个不同的级别：L1，L2，L3。越接近cpu速度越快，容量越小。如图2-1所示，

- L1(级别1)高速缓存是计算机系统中最快的内存，其中L1代表指令缓存，L1 D代表数据缓存
- L2(级别2)缓存的容量比L1缓存大，但速度慢。L2缓存保存下一个可能被CPU访问的数据。在大多数现代CPU中，L1和L2缓存都存在于CPU核心上，每个核心都有专用的缓存
- L3(级别3)缓存是最大的缓存内存，但它也是三个缓存内存中最慢的。它也是CPU上所有核心之间共享的资源，可以内部分区，以允许每个核心拥有专用的L3资源

## 3.**Power-Fail Protected Domains**

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201215145146690.png" alt="image-20201215145146690" style="zoom:50%;" />

简单的理解当数据到达ADR中，数据掉电不丢失。位于L1，L2，L3中的数据需要通过clwb+fence等指令刷新到ADR中。EADR可以理解为增强的ADR，其包括L1，L2，L3级缓存。

一旦接受到数据，内存控制器的WPQ将向writer确认收到了数据。当数据掉电的时候，存储在CPU缓存中的数据会丢失，但是在ADR中的数据不会丢失。如果这个平台支持EADR的话，那么在CPU缓存中的数据也掉电不丢失。将持久性域扩展到CPU缓存所要面临的挑战是CPU缓存非常大，并且将需要更大的能量。这意味着平台必须包含电池或使用外部不间断电源，为每台支持持久性内存的服务器都配备电池是不实际的，并且成本昂贵。

## 4.The Need for Flushing, Ordering, and Fencing

Non-temporal stores意味着写入的数据不会很快被再次读取，因此我们绕过了CPU缓存。也就是说，不存在时间局部性，因此将数据保存在处理器的缓存中没有好处，而且如果存储的数据取代了缓存中的其他有用数据，可能会有惩罚。由于缓存刷新具有不确定性，可能发生崩溃之后持久性内存中的数据结构不一致的问题。

<img src="https://cdn.jsdelivr.net/gh/HustDsy/Picture/image-20201215152242643.png" alt="image-20201215152242643" style="zoom:50%;" />

由于缓存刷新顺序不一致，因此可能导致崩溃之后数据结构的不一致性。因此加入内存屏障解决这个问题。内存屏障可以保证刷新顺序的一致性，但是不能保证数据的完整性。一个完美的解决方案应该使用事务来确保数据是原子更新的。下图是使用屏障之后的理想表现

<img src="https://cdn.jsdelivr.net/gh/HustDsy/Picture/image-20201215153301226.png" alt="image-20201215153301226" style="zoom:50%;" />

## 5.Intel Machine Instructions for Persistent Memory

下标展示了一下常用指令

![image-20201215154255684](https://cdn.jsdelivr.net/gh/HustDsy/Picture/image-20201215154255684.png)

![image-20201215154342768](https://cdn.jsdelivr.net/gh/HustDsy/Picture/image-20201215154342768.png)

## 6.**Detecting Platform Capabilities**

不同平台的指令不一样，PMDK中的libpmem提供了对这些库的检测

<img src="https://cdn.jsdelivr.net/gh/HustDsy/Picture/image-20201215154700751.png" alt="image-20201215154700751" style="zoom:50%;" />

