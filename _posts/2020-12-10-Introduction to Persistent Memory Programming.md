---
layout:     post
title:      1.Introduction to Persistent Memory Programming
subtitle:   PMDK
date:       2020-12-15
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - PMDK
---

> 这周一定抓紧时间吧PMEMOBJ这一块看完！！！先从第一章节开始吧，这一章节主要是为了对持久性内存编程做一个介绍

下面是一个简单的使用libpmemkv的例子，主要就是创建一个cmap引擎，然后插入三个键值对再读取出来全部的键值对

```c++
#include <iostream>
#include <cassert>
#include <libpmemkv.hpp>

using namespace pmem::kv;
using std::cerr;
using std::cout;
using std::endl;
using std::string;

/*
 * for this example, create a 1 Gig file
 * called "/daxfs/kvfile"
 */
auto PATH = "/daxfs/kvfile";
const uint64_t SIZE = 1024 * 1024 * 1024;

/*
 * kvprint -- print a single key-value pair
 */
int kvprint(string_view k, string_view v) {
	cout << "key: "    << k.data() <<
		" value: " << v.data() << endl;
	return 0;
}

int main() {
	// start by creating the db object
	db *kv = new db();
	assert(kv != nullptr);

	// create the config information for
	// libpmemkv's open method
  //配置文件信息，包括路径，大小等
	config cfg;

	if (cfg.put_string("path", PATH) != status::OK) {
		cerr << pmemkv_errormsg() << endl;
		exit(1);
	}
	if (cfg.put_uint64("force_create", 1) != status::OK) {
		cerr << pmemkv_errormsg() << endl;
		exit(1);
	}
	if (cfg.put_uint64("size", SIZE) != status::OK) {
		cerr << pmemkv_errormsg() << endl;
		exit(1);
	}


	// open the key-value store, using the cmap engine
 	//使用cmap引擎
	if (kv->open("cmap", std::move(cfg)) != status::OK) {
		cerr << db::errormsg() << endl;
		exit(1);
	}

	// add some keys and values
	if (kv->put("key1", "value1") != status::OK) {
		cerr << db::errormsg() << endl;
		exit(1);
	}
	if (kv->put("key2", "value2") != status::OK) {
		cerr << db::errormsg() << endl;
		exit(1);
	}
	if (kv->put("key3", "value3") != status::OK) {
		cerr << db::errormsg() << endl;
		exit(1);
	}

	// iterate through the key-value store, printing them
	kv->get_all(kvprint);

	// stop the pmemkv engine
	delete kv;

	exit(0);
}
```

## 1.持久性内存的KV数据库和传统的使用易失性内存的数据库的区别

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201214164514782.png" alt="image-20201214164514782" style="zoom:50%;" />

​																		               <strong>传统storage中的键值数据库</strong>

如上图，当应用程序想要获得一个KV项时必须在内存中分配一个缓冲区来保存结果。因为这些值存储在块存储中。应用程序无法对其进行寻址。访问一个值的唯一方法就是将它带到内存中，唯一的方法就是从存储设备读取完整的块，这只能通过块IO进行访问，就是从外存读取数据到缓冲区。

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201214165916382.png" alt="image-20201214165916382" style="zoom:50%;" />

持久性键值存储，应用程序可以直接访问值，而不需要首先在内存中分配缓冲区，还一个优点就是基于storage的键值数据库当需要进行小的更新时，它必须将包含着64字节的存储块读入内存缓冲区，然后再写入到整个块使其持久。这是因为storage的访问只能使用块IO进行，通常一次是4K字节，因此更新64字节的任务需要先读取4K，然后再写入4K。但是对于持久性内存。更改64字节的内容会直接将64字节的更新写入持久性内存。

## 2.**The Performance Difference**

图1-3展示了不同类型的存储介质的延迟层次结构

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201214172034605.png" alt="image-20201214172034605" style="zoom:50%;" />

可以看出来持久性内存提供了与内存类似的延迟

## **3.Program Complexity**

libpmemkv提供了高级的API抽象除了数据结构在持久性内存中的试试，但是在最低层次上，直接向原始持久性内存编程需要了解诸如原子性，缓存刷新和事务等方面的详细只是。libpmemkv这样的高级库将所有这些复杂性抽象出，并且提供更加简单不容易出错的接口。

## 4.**How Does libpmemkv Work?**

libpmemkv这样组的高级库隐藏了所有的复杂性，图1-4展示了应用程序使用libpmemkv时所涉及的完整软件栈

<img src="https://gitee.com/hustdsy/blog-img/raw/master/image-20201215100841705.png" alt="image-20201215100841705" style="zoom:50%;" />

- 持久性内存硬件通常连接到系统内存总线，并使用load/store操作进行访问。
- pmem-aware文件系统将持久性内存以文件的形式暴露给应用程序。可以对这些文件进行内存映射，从而应用程序可以直接访问这些文件（DAX）
- libpmem库是PMDK的一部分，这个库抽象了一些底层的硬件指令，比如缓存刷新指令
- libpmemobj是一个用于持久内存的功能齐全的事务和内存分配库，主要用它来实现一些需要的数据结构，还有c++版本
- cmap引擎，基于持久新内存优化的的哈希map
- libpmemkv提供文章开头的一些接口
- 应用程序使用这些高级接口

