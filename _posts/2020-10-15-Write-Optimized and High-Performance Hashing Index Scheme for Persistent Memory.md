---
layout:     post
title:      Write-Optimized and High-Performance Hashing Index Scheme for Persistent Memory
subtitle:   OSDI’18 Level Hash
date:       2020-10-15
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Hash索引
---

> 这篇文章是左鹏飞师兄的第二篇Hash文章，简称Level Hash。

## 1.背景

#### 1.1 *Question*

- *High Overhead for Consistency Guarantee*:持久性内存的数据结构应该避免数据的不一致性，比如<mark>部分更新</mark>和<mark>数据丢失</mark>对内存的写入顺序往往会被CPU，内存控制器进行排序，通常使用*cache line*的刷新指令以及*memory*屏障指令来确保我们理想的写入顺序。现在原子写单元的大小一般是8Bytes，如果写入数据的大小比这个原子写单元的大小大的话，那么就需要昂贵的日志或者写时复制（*COW*）来保证一致性。
- *Performance Degradation for Reducing Writes*:对于写友好的哈希策略来说，PFHT和Path Hasing结构都牺牲了一定的读性能。
- *Cost Inefficiency for Resizing Hash Table.*:当达到一定的负载因子时，哈希冲突的概率大大增加，这就需要进行扩容操作，往常的静态Hash的Resize操作通常是对整个Hash Table进行扩容，这是低效的。

#### 1.2*Hashing Index Structures for NVM*

传统的*Hash*策略包括链式哈希，开放地址哈希，BCH(Bucket Cuckoo Hash)，已有的NVM Hash主要是在以往的Hash基础上减少了写次数，比如*Path Hash*，*PFHT*

![image-20201015095901672](https://gitee.com/hustdsy/blog-img/raw/master/image-20201015095901672.png)

## 2.设计

#### 2.1Write-optimized Hash Table Structure

![image-20201016150937144](https://gitee.com/hustdsy/blog-img/raw/master/image-20201016150937144.png)

- 每个桶包含多个槽
- 有两个*Hash*函数，类似于*Cuckoo Hash*
- 允许一次级连写
- 两层的结构，下面的桶作为上面的桶的共享存储的桶

#### 2.2Cost-efficient In-place Resizing

![image-20201016152825684](https://gitee.com/hustdsy/blog-img/raw/master/image-20201016152825684.png)

1. 先申请一块新的内存地址作为新的TL，旧的*TL*和*BL*对应的向下移动一层，这样子的话旧的*TL*就变为新的*BL*,而且这一部分是无需进行扩容处理的，原来在旧*TL*为2的项目，现在对应4,5，刚好满足要求。
2. 现在只需要对*old BL*进行*ReHasing*操作即可

<strong><mark>首先来看插入操作，Level Hash的操作好像没有唯一性检测</mark></strong>

rehash操作带来的后果，<strong><mark>首先</mark></strong>我们来看插入操作，插入操作好像没做重复性检测，这样子的话可能会导致查询出来的数据并不是准确的数据，这个得具体看代码去实现，可能插入之前先查询一遍有没有这个$Key$，那这样子的话 ，是十分耗时的。<strong><mark>其次就是</mark></strong>这样子rehash的话，TL的数据变得稀疏，BL层数据反而变得密集，由于查找的时候由于先查找TL层，再去查找BL层，*rehash*之后容易带来查找的性能损失。作者提出了一种动态的搜索方式，那就是搜索之前比较*TL*和*BL*的数据密集程度，先搜索密集程度大的。<strong><mark>最后一个问题</mark></strong>就是只有当一个*key*在TL的两个位置满了，并且一次移动也不能进行，之后就会去看BL,BL的两个位置满了，然后一次移动也不能进行的话，这样子的话才会就会导致*ReHash*操作，现在BL层是满的，而TL层是稀疏的，这样子的话容易降低负载因子。

<font color="red">这里可以这样子理解，BL层越密集，越容易导致ReHash操作</font>

#### 2.3一致性保证

这里先说一下背景，现在保证奔溃一致性流行的方法是通过日志或者写时复制等方法，这样子的开销过大。原子写只支持8B大小的数据。作者的方法就是加标志位。来实现低开销的一致性保证。一致性主要就是防止丢失之前的数据，或者部分更新了现有的数据。

![image-20201016162558515](https://gitee.com/hustdsy/blog-img/raw/master/image-20201016162558515.png)

###### 2.3.1删除操作的一致性保证

将要删除的*key*的*token*置为0，之后使用原子写将这个*token*写入，这样子的话，保证了删除的一致性保证

###### 2.3.2插入操作的一致性保证

这分为两个不同的情况，第一种插入不需要条目移动，第二种需要条目移动。

- 对于不需要条目移动的插入，找到空槽，先写入数据，在更改token，再原子写将token写入NVM。这样子的话，只有*token*被写入了，这个数据才有效。
- 对于需要条目移动的插入，首先找到需要移动的槽，使用需要条目移动的插入将数据写入找到的空位，然后执行删除操作，将原来的条目所在的位置*token*置为0，这样子的话就有了一个空位了，执行不需要条目移动的插入即可。这样子的话唯一的一个问题就是造成数据冗余，如果程序在执行删除操作之前崩溃的话，那就多了一个重复的数据。

###### 2.3.3Rehash操作的一致性保证

对于*Rehash*操作，作者先计算出IL层数据的新值，再插入到新的TL中，更新Token，再更新IL数据的Token的数据，达到删除的目的。

###### 2.3.4更新操作的一致性保证

这也分为两种情况，第一种情况更新的数据处有空位置，第二种情况更新的数据处没有空位置

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201019102805718.png" alt="image-20201019102805718" style="zoom:50%;" />

- 对于有空位置的情况，先将需要更新的新数据写入到这个空位置，然后修改新数据的标记位为1，旧数据的标记位为0。再使用原子写将*token*写回。
- 对于没有空位置的情况，这就需要采用日志来进行处理了，首先将旧的数据写日志，然后再原地更新旧数据，如果产生崩溃的话，这个时候就需要采用日志来进行恢复。

#### 2.4Concurrent Level Hashing

在并发级哈希中，当不同的线程并发地读/写同一个槽时，就会发生冲突。因此，我们为每个槽分配一个细粒度的锁。当读/写一个槽时，线程首先锁定它。因为级别哈希允许每个插入最多移动一个现有项，所以插入操作最多锁定两个插槽，即当前插槽和项目将被移动到的目标插槽。然而，插入导致移动的概率非常低。插入在大多数情况下只锁住一个插槽



