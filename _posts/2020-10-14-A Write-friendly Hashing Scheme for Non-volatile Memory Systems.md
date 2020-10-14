---
layout:     post
title:      A Write-friendly Hashing Scheme for Non-volatile Memory Systems
subtitle:   MSST’2017 Path Hash
date:       2020-10-14
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - C++ Primer
---

> 这一篇论文提出了一种新型的解决哈希冲突的方案，全文8页，现在已存的Hash方案容易造成级联写，对于寿命有限的PCM并且写延迟高的PCM来说不太友好。这里主要讲一下大致的结构，以及增删改查操作。

## 1.The Design Of Path Hashing

#### 1.1*Position Sharing*

![image-20201014165124658](https://gitee.com/hustdsy/blog-img/raw/master/image-20201014165124658.png)

这里可以看出对于一个X，Hash到2的位置，那么下面的从这个叶节点到根节点的路径上的位置均可以访问。它和3共享一条路径。

#### 1.2*Double-path Hashing*

![image-20201014165418567](https://gitee.com/hustdsy/blog-img/raw/master/image-20201014165418567.png)

假设只有一条路径的话，那么这条路径很容易满，作者为了提供多条路径，采取两个Hash函数，这样子的话X就可以有两条Path。

#### 1.3*Path Shortening*

![image-20201014165631843](https://gitee.com/hustdsy/blog-img/raw/master/image-20201014165631843.png)

对于上层的叶子节点，提供的可以解决冲突的位置越来越少，但是层数越多，搜索路径越多，我们可以减少一定的层数，从而减少查找路径。

#### 1.4*Physical Storage Structure*

![image-20201014165808283](https://gitee.com/hustdsy/blog-img/raw/master/image-20201014165808283.png)

对于这样子的到二叉数结构，不使用树结构来进行存储，而是采用一块连续的地址来进行存储。映射关系如下图。

![image-20201014170224963](https://gitee.com/hustdsy/blog-img/raw/master/image-20201014170224963.png)