---
layout:     post
title:      FPTree A Hybrid SCM-DRAM Persistent and Concurrent B-Tree for Storage Class Memory
subtitle:   FPTree
date:       2020-09-05
author:     HustDsy
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - 树索引
---

#### 1.目标

- 持久性，FP-Tree必须从任何软件奔溃或电源故障的场景中，自动恢复到一致性的状态；
- 接近DRAM的性能
- FP-Tree的性能必须要适应更高的SCM延迟
- FP-Tree的恢复时间要比临时的B+树重建速度要更更快
- 高的可伸缩性
- FP-Tree支持可变大小的键

<div align = "center">FP-Tree构造</div>

![FP-Tree构造](https://tva1.sinaimg.cn/large/007S8ZIlly1gig7luyy6gj30se08y0u2.jpg)

如图二所示，非叶子节点有序的存储在DRAM中，叶子节点放在SCM中，并且叶子节点以链表的形式连接。这主要有以下两点的好处：

1. 允许范围查询
2. 允许在恢复过程中遍历所有叶节点来重建内部节点
3. 与完全存储在DRAM的之前的树的设计来说，只是访问叶子节点的性能有所差距，所以性能差距不大。

#### 2.设计原则

##### 2.1 选择持久性

对于DRAM的非叶子节点，作者并不打算持久化它，而是通过叶子节点去重建它，只要叶子节点的一致性以及持久性得到的保证，那么非叶子节点就可以顺利的重建。

##### 2.2.FingerPrint

- <font color="red" face="黑体">这里的平均查找次数主要指的时访问Key的次数，而不是查找次数</font>
- <font color=red face="黑体">Finger主要的作用是来减少访问Key的次数，而不是减少查找次数</font>

首先显而易见，对于一个无序链表的平均查找时间是$o(\frac{n+1}{2})$，这样子的线性扫描是昂贵的，是可以继续优化的。NV-Tree就是典型的代表。WB-Tree的节点是有序的，它的查找时间复杂度是o($log_2n$)，Fp-Tree如果节点是有序的话，那么写延迟带来的性能影响或许不可接受。因此作者采用fingerprint对数据进行过滤，fingerprint相当与key的hash值。下面来证明，这样子的查找期望值为1。假设现在有n(假设用一个字节来表示fingerprint那么它的值就是256)个fingerprint，m个键值对。

大该的思路就是由于hash产生的多余的搜索次数是来自于碰撞，对于任何一个fingerprint产生碰撞的次数为1-m,最好的情况就是不产生碰撞，一次就命中了，如果产生碰撞的话次数就会增加。最多搜索m次。我们接下来假设这个期望值为$E_k$。

1.假设出现的次数为i，出现i的概率为 $P(i)$.那么 $E_k$的求法就是

\\[
E_k=\sum_{i=1}^{m}iP(i)
\\]

2.接下来思考$P(i)$的求法，我们假设两个事件A，B。

- A：刚刚好只碰撞i次概率
- B：至少碰撞一次的概率

容易知道出现A的情况肯定包括B，求A，我们如果这个键存在的话，也就是我们要查找的值存在的话，至少得查找一次。

查找一次的概率记作$P(B)$，查找i次的概率记作$P(A)$.

根据联合概率公式有

$$
P(i)=P(A|B)=\frac{P(A\bigcap{B})}{P(B)}
$$

这么一来我们就只要算$P(B)$和$P(A\bigcap{B})$的概率即可。根据题目意思$P(B)$代表至少碰撞1次的概率，那么我们用1减去碰撞为0次的概率，也就是搜索不到的概率$P(\overline{B})$。

$$
P(B)=1-P(\overline{B})
$$

现在查找某个fingerprint，m个key没有一个的hash值是这个key。对于一个key，产生的hash值刚好是这个fingerprint的概率为$\frac{1}{n}$所以$P(B)$的数值为

$$
P(B)=1-(1-\frac{1}{n})^m
$$

接下来求$P(A|B)$:

$$
P(A|B)=\mathrm{C}{_m^i}(\frac{1}{n}){^i}(1-\frac{1}{n}){^{m-i}}
$$

到这里公式推导到一半了，现在$P(i)$的公式就出来了

$$
P(i)=\frac{\mathrm{C}{_m^i}(\frac{1}{n}){^i}(1-\frac{1}{n})^{m-i}}{1-(1-\frac{1}{n}){^m}}
$$

下面复杂度的推导换个思路，对于m个元素，碰撞次数的期望值为$E_k$，现在把这个冲突的$E_k$个数单独取出来分析，对于这$E_k$个数来说，它的遍历时间相当于一次线性遍历的时间，也就是$o(\frac{n+1}{2})$。当然这里的n就是我们的$E_k$了。所以FP-Tree的平均查找时间为$E_T$

$$
\begin{aligned}
E_T=\frac{1}{2}(1+E_K)
&=\frac{1}{2}(1+\sum_{i=1}^{m}i\frac{\mathrm{C}{_m^i}(\frac{1}{n}){^i}(1-\frac{1}{n})^{m-i}}{1-(1-\frac{1}{n}){^m}})&\text{............1}\\
&=\frac{1}{2}({1+\frac{(\frac{n-1}{n})^m}{1-(\frac{n-1}{n})^m}\sum_{i=1}^{m}\frac{i\mathrm{C}{_m^i}}{(n-1)^i}})&\text{............2}\\
&=\frac{1}{2}({1+\frac{(\frac{n-1}{n})^m}{1-(\frac{n-1}{n})^m}(\frac{m}{n-1})\sum_{i=0}^{m-1}\frac{\mathrm{C}{_{m-1}^i}}{(n-1)^i}})&\text{............3}\\
\end{aligned}
$$

对于式子3的后缀部分使用二项式定理$(1+x)^n=\mathrm{C}{_n^i}x^i$其中i的取值范围是$0\text{到}(n-1)$,所以3式可以化成

$$
\begin{aligned}
E_T
&=\frac{1}{2}({1+\frac{(\frac{n-1}{n})^m}{1-(\frac{n-1}{n})^m}(\frac{m}{n-1})(\frac{n}{n-1})^{m-1})}\\
&=\frac{1}{2}(1+\frac{m}{n(1-(\frac{n-1}{n}){^m})})
\end{aligned}
$$

一般来说，fingerprint是一字节大小的，可以表示256个不同的hash值，n也就是为256，当m取32的时候。代入公式得$E_K$为1.这个我用计数器算了一下，大概是接近于与，代入m=32，n=256即可。

##### 2.3Amortized persistent memory allocations(分摊持久性内存分配)

持久性内存的分配带来的性能代价是昂贵的，在分裂阶段的持久性内存分配对于FP-Tree的性能有一定的影响。作者建议采用一次性分配多个叶节点块来解决这个问题。

![FPTree内存分配](https://tva1.sinaimg.cn/large/007S8ZIlly1gig7p4h3njj30tg0au0u3.jpg)

如图五所示，作者一次性分配一组空闲的节点。下面L1->L2->L3->L4用来代表实际在FP-Tree被使用的节点。而对于没被使用的节点作者采用一个易失性的数组来进行索引。接下来讲述一下如何获取节点以及删除linkd list of groups叶组里面的节点。

- **GetLeaf**：当需要新节点的时候，从易失性数组（下面称vector）拿，如果这个vector不为空的话，那么就采用pop_back操作，将没被使用的已经分配的叶子节点取出来。如果这个数组为空的话，那么再分配一个leaf groups。重复刚刚不为空的机会即可。
- **FreeLeaf**：当一个叶子节点需要被free的时候，首先将这个叶节点放到vector中，如果这个叶节点所在的叶组全部在vector中，那么就在leaf groups中把对应的gruop全部free掉。

==这里我觉得它这里会带来一系列的增删改查的性能消耗，但是可能相比较于在分裂阶段的持久性内存分配所带来的性能消耗要小一点，所以是可行的==

这里我讲一下自己的几点疑问：

- <font color="red">易失性的vector如何重建</font>

- <div style="color:red">如何判断一个group里面的叶节点都被free了</div>

- <div style="color:red">当free掉一个叶节点将其放入vector的时候,是有序的还是无序的</div>

- <font color="red">这个数组的一致性如何保证</font>
- <font color="red">作者在算法中的描述没有体现出这一点，而且引入这个确实给伸缩性带来一定的挑战，因此作者并发FP-Tree没有实现它，只在单线程的FP-Tree中实现了它</font>

##### 2.4选择一致性

对于存储在DRAM部分的非叶子节点，采用HTX，硬件事务内存来进行保证一致性，而对于SCM部分的叶子节点，保证一致性，作者采取的加锁的方式。

#### 3.实现，算法的伪代码的讲解

这一部分主要是针并发性FP-Tree的讲解，主要是实现了上面原则的选择持久性，选择一致性以及FingerPrint这三个点，首先十分简单的说一下TSX事务，x_begin()代表事务开始，x_abort()代表事务中断,x_end()代表事务结束，其中事务中断的话会回滚到x_begin()再进行操作。

![FPTree操作](https://tva1.sinaimg.cn/large/007S8ZIlly1gig7q5e2dej30w00fajtl.jpg)

##### 3.1Find函数

直接上伪代码，Find函数不涉及数据的修改操作，相对于来说比较简单，只需要一个TSX事务即可。

```c++
ConcurrentFind(Key K):
  while TRUE do 
    speculative_lock.acquire(); //相当于x_begin()
    Leaf = FindLeaf(K);//包含这个K的叶子节点
    if Leaf.lock == 1 then //如果这个叶子节点正在被占用的话，从begin重新开始
      speculative_lock.abort();//相当于x_abort()
      continue;
		//fingerprint的作用就是减少不必要的K的读取时间
    for each slot in Leaf do
      set currentKey to key pointed to by Leaf.KV[slot].PKey
      //如果这个slot有效的话，并且FingerPrint也对的上的话，再去访问真实的Key，由于碰撞次数期望值为1，因此基本访问一次即可
      if Leaf.Bitmap[slot] == 1 and Leaf.Fingerprints[slot] == hash(K) and currentKey == K then
        Val = Leaf.KV[slot].Val;
        Break; 
    speculative_lock.release(); //相当于x_end()
    return Val;
```

##### 3.2Insert函数

话不多说 继续上伪代码，插入之前肯定先找到一个叶子节点的一个空白的槽，这一段过程在TSX事务即可，提现可选型持久性和选择一致性。之后判断是否要分裂

```c++
//并发插入函数
ConcurrentInsert(Key K, Value V)
  Decision = Result::Abort;
  while Decision == Result::Abort do
  	speculative_lock.acquire(); //x_begin，TSX事务的开始
 	  (Leaf, Parent) = FindLeaf(K);//找到要插入的叶子节点，以及该节点的父亲节点
 	 	if Leaf.lock == 1 then //一致性需要，如果被锁住了，等待
      Decision = Result::Abort; Continue;
  	Leaf.lock = 1; /* Writes to leaf locks are never persisted 插入操作，上锁*/
  	Decision = Leaf.isFull() ? Result::Split : Result::Insert;//判断要插入的叶子节点是否需要分裂
  	speculative_lock.release();//x_end TSX事务结束，
	if Decision == Result::Split then//如果需要进行分裂，先进行分裂操作
		splitKey = SplitLeaf(Leaf);	
	//分裂完成开始插入操作
	slot = Leaf.Bitmap.FindFirstZero(); //找到第一个空的slot
  Leaf.KV[slot] = (K, V); //将K，V值写入这个槽
	Leaf.Fingerprints[slot] = hash(K);//写入fingerprint
 	Persist(Leaf.KV[slot]); //持久化这个KV槽
	Persist(Leaf.Fingerprints[slot]);//持久化指纹
  Leaf.Bitmap[slot] = 1; //将位设为1，只有这样子 这个槽才算有效
	Persist(Leaf.Bitmap);//持久化
	//DRAM部分的非叶子节点在TSX事务中进行更行
  if Decision == Result::Split then
    speculative_lock.acquire();
		UpdateParents(splitKey, Parent, Leaf);//update函数应该包含两种情况，parent拆分与不拆分
  	speculative_lock.release();
	Leaf.lock = 0;
```
//分裂函数，采取日志来进行操作的
SplitLeaf(LeafNode Leaf)

```c++
  get μLog from SplitLogQueue;
  set μLog.PCurrentLeaf to persistent address of Leaf; //PCurrentLeaf 要分裂的叶子节点地址
  Persist(μLog.PCurrentLeaf);//持久化 step1
  Allocate(μLog.PNewLeaf, sizeof(LeafNode))//PNewLeaf指向新分配的节点的地址 step2
  set NewLeaf to leaf pointed to by μLog.PNewLeaf; //PNewLeaf 已经分裂的新节点的地址
  Copy the content of Leaf into NewLeaf;//将旧节点的内容copy到新节点去                             ----赋值操作
  Persist(NewLeaf);//将NewLeaf持久化
  (splitKey, bmp) = FindSplitKey(Leaf);//得到要插入的KV值的key和bmp(这一部分是在旧数据当中)
  NewLeaf.Bitmap = bmp;//将旧数据的bmp拷贝给NewLeaf
  Persist(NewLeaf.Bitmap);//原子写持久化bitmap
  Leaf.Bitmap = inverse(NewLeaf.Bitmap);//旧节点的数据拷贝走了，置为空 step3                       ------继续操作
  Persist(Leaf.Bitmap);//持久化bitmap 
  set Leaf.Next to persistent address of NewLeaf; //修改链表，leaf的下一个节点指向newLeaf
  Persist(Leaf.Next);//持久化next指针
  reset μLog;//重置日志
```

```c++
if μLog.PCurrentLeaf == NULL then 
	return;
if μLog.PNewLeaf == NULL then 
	/* Crashed before SplitLeaf:4 */ 
	reset μLog;
else
	if μLog.PCurrentLeaf.Bitmap.IsFull() then
		/* Crashed before SplitLeaf:11 */
		Continue leaf split from SplitLeaf:6;
	else
		/* Crashed after SplitLeaf:11 */ 
		Continue leaf split from SplitLeaf:11;
```

- step1这里崩溃的话，也就是PCurrentLeaf分配出错的话，那么就重置 直接返回
- step2这里奔溃的话 ，也就是PNewLeaf分配出错的话，看PNewLeaf是否为空，空的话reset日志，非空的话，往下操作
- Step3这里崩溃的话，也就是将旧节点的内容赋值给新节点。查看日志，检查旧叶子节点的位图是否满了，如果满了的话，那说明赋      值没完成，继续赋值。如果没满的话，那就继续操作。

##### 3.3delete函数

delete分为三种情况：

1. 在叶子中找不到要删除的key
2. 叶子包含要删除的key以及其它key
3. 叶子中只包含要删除的key（<font color="red">**可能涉及回收操作**</font>）

对应的处理情况如下：

1. 直接返回即可
2. TSX事务中搜索该key，然后给叶子加锁，bitmap置为0，即可
3. 这个的话涉及free操作。TSX事务搜索该叶子，但这个不用加锁，因为非叶子节点更新之后，其它线程无法访问它。搜索操作以及DRAM中的非叶子节点的更新都是在TSX事务之内进行的。在TSX事务之外的就是更新叶子节点链表了。比如A->B->C要变为A->C。如果删除的是头节点的话那么就要变为B->C。因此删除这里也是一个判断的。

<div style="color:green" align="center" >
  <strong>ConcurrentDelete(Key K)</strong>
</div>

```c++
Decision = Result::Abort;
while Decision == Result::Abort do
	speculative_lock.acquire(); //TXS事务开始
  /* PrevLeaf is locked only if Decision == LeafEmpty */ 
  (Leaf, PPrevLeaf) = FindLeafAndPrevLeaf(K); //寻找key所在的节点
  if Leaf.lock == 1 then //被锁住了 继续下一个循环
  	Decision = Result::Abort; Continue; 
	if Leaf.Bitmap.count() == 1 then//这个叶子节点的key只有一个的话，说明删完之后要free了
  	if PPrevLeaf->lock == 1 then//因为要处理左节点，所以左节点被锁住了的话，继续等待
  		Decision = Result::Abort; Continue; 
		Leaf.lock = 1; 
		PPrevLeaf->lock = 1; //锁住要处理的节点
		Decision = Result::LeafEmpty;//标记删完之后叶子就要被free了
  else//如果出了要删除的key还有其他key
		Leaf.lock = 1; //只处理这个叶子就好，clock
		Decision = Result::Delete; //标记只要删除操作 不用free
	speculative_lock.release();//前面的操作在TSX事务就行
  if Decision == Result::LeafEmpty then //如果需要free的话
    DeleteLeaf(Leaf, PPrevLeaf); //删除这个叶子
		PrevLeaf.lock = 0;//释放锁
  else //不用free的话，将bitmap置为0就行
  	slot = Leaf.FindInLeaf(K);
		Leaf.Bitmap[slot] = 0;
		Persist(Leaf.Bitmap[slot]); 
		Leaf.lock = 0;
```

<div style="color:green" align="center" >
  <strong>DeleteLeaf(LeafNode Leaf, LeafNode PPrevLeaf)</strong>
</div>

```c++
get the head of the linked list of leaves PHead//得到头节点
get μLog from DeleteLogQueue;//同理，删除叶节点的一致性也是用日志来保证
set μLog.PCurrentLeaf to persistent address of Leaf; 
Persist(μLog.PCurrentLeaf);//用一个持久性的指针指向要删除的节点
if μLog.PCurrentLeaf == PHead then//如果要删除的节点是头结点
	/* Leaf is the head of the linked list of leaves */
  PHead = Leaf.Next;//直接将头结点换成leaf.next也就是A->B->C,变成B->C
	Persist(PHead);
else//这种情况就是A->B->C变为A->C
  μLog.PPrevLeaf = PPrevLeaf; 
  Persist(μLog.PPrevLeaf); //记录要要删除的节点的
  PrevLeaf.Next = Leaf.Next; //直接A->C
  Persist(PrevLeaf.Next)//持久化
Deallocate(μLog.PCurrentLeaf); //删除
reset μLog;
```

<div style="color:green" align="center" >
  <strong>RecoverDelete(DeleteLog μLog))</strong>
</div>

```c++
get head of linked list of leaves PHead;
if μLog.PCurrentLeaf != NULL and μLog.PPrevLeaf != NULL then
	/* Crashed between lines DeleteLeaf:12-14 */
	Continue from DeleteLeaf:12;
else
	if μLog.PCurrentLeaf != NULL and μLog.PCurrentLeaf == PHead 
  then
		/* Crashed at line DeleteLeaf:7 */
		Continue from DeleteLeaf:7;
else
	if μLog.PCurrentLeaf != NULL and μLog.PCurrentLeaf→Next== PHead then
		/* Crashed at line DeleteLeaf:14 */
    Continue from DeleteLeaf:14;
	else
		reset μLog;
```




