---
layout:     post
title:      3.Persistent Memory Architecturey
subtitle:   PMDK
date:       2020-12-15
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - PMDK
---

> 第三章主要是介绍操作系统如何将持久内存作为平台资源来进行管理

## 1.**Operating System Support for Memory and Storage**

<img src="https://cdn.jsdelivr.net/gh/HustDsy/Picture/image-20201215160149828.png" alt="image-20201215160149828" style="zoom:50%;" />

如图3-1，易失性主存通过内存总线直接连接到CPU。操作系统直接将内存区域映射到应用程序的可见内存地址空间。storage运行速度通常比CPU慢的多，它通过一个I/O控制器连接。操作系统通过加载到I/O子系统中的设备驱动模块来处理对storage的访问。

## 2.Persistent Memory As Block Storage

<img src="https://cdn.jsdelivr.net/gh/HustDsy/Picture/image-20201215161239452.png" alt="image-20201215161239452" style="zoom:50%;" />

操作系统拥有的第一个能力就是检测持久性内存模块的存在并且加载驱动器到I/O subsystem中。驱动器拥有两个功能，第一个功能就是为使用者提供一个接口来配置和监视持久内存硬件的状态。第二个功能类似于storage设备驱动器。

HDD和SSD保证了512B或者4KB大小的块要么全更新，要么不更新，持久性内存充当块设备的时候也提供了类似的保证

## 3.**Persistent Memory-Aware File Systems**

