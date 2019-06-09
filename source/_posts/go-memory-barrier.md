---
title: 硬件角度看内存屏障
date: 2019-06-02
tags: architecture
---
>本文参考论文 Memory Barriers:a Hardware View for Software Hackers

## cpu cache

cpu能够在1纳秒执行数十条指令,但是从内存中获取一个数据需要几百纳秒,有两个数量级的差距,因此在现代的cpu上会有多达几MB的cache.数据在cache和内存之间以固定大小的blocks来流动,称之为'cache lines',一般大小为2的幂次,从16Bytes-256Bytes.cache lines可以理解为一个硬件的hash表,"two way"表示每个槽中可以放置两个entry,如果是256bytes对齐,那么低8bits都是0,可以取8-12bits为hash值.例如,cpu cache槽为0x0-0xF,则0x12345E00的8-12bits为E,会放置到0xE这个槽中.

现代cpu一般会有多个core,每个core有自己的cache,因此必然涉及到缓存一致性的问题

## 缓存一致性协议

Cache-coherency protocols管理cache line的状态,防止不一致或者丢失数据

### MESI

MESI代表'modified','exclusive','shared'和'invalid',代表缓存的四种状态.因此一个cache line除了物理地址和数据之外,会单独使用2bits(tag)去保存这个状态

* modified:处于该状态的cpu持有唯一最新的数据.因此该cache负责将数据写回内存或者传递给其他cache,并且在该cache保存其他数据之前必须先执行此步骤.
* exclusive:类似于modified,但是还没有被修改.因此内存中的该数据是最新的.
* shared:处于该状态表明数据在至少两个cpu cache中存在,因此该cpu必须在询问其他cpu之后才能够保存该数据
* invalid:invalid状态的数据可以被替换

因为所有cache line中的数据必须保持一致性的视图,因此cache-coherency protocols提供了消息机制保持系统间的交流

### MESI消息

* Read:消息中包括要读取的物理地址
* Read Response:Read的响应,可以是内存或者其他cache来响应.例如如果一个cache处于modified状态,那么他的数据是最新的,必须响应一个Read Response消息
* Invalidate:Invalidate消息包括一个cache保存的物理地址,所有其他cache必须将相应的数据清除并响应
* Invalidate Acknowledge:收到Invalidate消息的CPU清除相应数据后需要回复Invalidate Acknowledge消息
* Read Invalidate:Read和Invalidate的结合体
* Writeback:处于modified的cache通过该消息eject lines从而腾出空间.writeback消息包括地址和数据,可以写回内存,也可以snooped到其他CPU的cache.


### MESI state转换

M->E:writeback message,但仍保留修改的权利
E->M:写入cache line.不需要消息
M->I:read invalidate message,将数据发给需要的CPU并且将本地缓存失效
I->M:对不在自己cache中的数据原子性的执行read-modify-write操作.首先发送一个read invalidate消息,通过read response收到数据,并且收到全部的invalidate acknowledge响应之后执行该动作.
S->M:对在自己cache中并且只读的数据原子性的执行read-modify-write操作,首先发送invalidate消息,等待接收到invalidate acknowledge之后执行该动作
M->S:收到read消息,将数据同步到需要的CPU甚至writeback回memory,本地保存一份只读的数据
S->E:该CPU需要修改本地的只读数据,因此首先发送invalidate消息,并且需要收到所有的invalidate response.其他CPU需要将数据writeback回内存
E->I:其他CPU需要对该CPU中的数据原子性的执行read-modify-write操作.该操作在收到read invalidate信息之后恢复一个read response和invalidate acknowledge
I->E:该CPU发送read invalidate并且收到有效回复后可以从I状态切换到E状态,并且很可能会继续从E->M
I->S:发送一个read消息后并且收到数据后可以切换到S状态
S->I:收到invalidate消息并且回复之后会从S切换到I状态

## CPU的Stores Result

假设CPU0需要写一个在CPU1中持有的cacheline,那么CPU0需要等待CPU1传递该cacheline的值,这时候CPU0必须stall一段时间
但是CPU0其实没必要等待,因为不论CPU1发什么数据给CPU0,CPU0最终都会无条件的覆盖它

### Store Buffers
为了防止该stall,CPU和cache之间加了一层store buffers,CPU0只需要简单的将其写入放到store buffer并且继续执行.当CPU1的cacheline数据到达之后,再从store buffer中将数据移动到cacheline中.
这种假设中store buffer的数据得先放到cache中,然后load的时候会从cache中load

### Store Frowarding
考虑如下代码:
```
a = 1;
b = a+1;
assert(b == 2);
```
如果按上边的假设会导致断言失效,b计算出来为1.因此修改上文的假设为:
load的时候不需要将store buffer中的数据先放入cache,而是直接从cache和store buffer中一起读取.
如果store buffer中有值直接使用

**可以具体参考论文,论文中有举例说明**

### Store Buffers 和 Memory Barriers
如下代码:
```
void foo(void)
{
	a = 1;
	b = 1;
}

void bar(void)
{
	while (b == 0) continue;
	assert (a == 1);
}
```
此种情形下也会导致断言失败,CPU0首先将a = 1放入store buffer,然后执行b = 1(假设CPU0执行foo,并且持有b,CPU1执行bar,并且持有a.).CPU1读到了b = 1但是a仍然在store buffer中还没有被处理

由于CPU无法知道变量之间的关联关系,因此提供了memory-barrier这个指令允许软件开发者能够告诉CPU这种关联关系.代码改为如下:
```
void foo(void)
{
	a = 1;
	smp_mb();
	b = 1;
}

void bar(void)
{
	while (b == 0) continue;
	assert (a == 1);
}
```

memory-barrier会告诉CPU在应用后续cache line之前必须先把store buffer中的数据flush.或者是通过stall一段时间直到store buffer清空,或者使用store buffer把后续指令也缓存,直到之前的指令已执行完成.

## store result引发的新问题

由于加memory-barrier之后,会将后续指令都放入store buffer,但是store buffer空间有限,此时需要等待invalidate完成之后才能够将store buffer腾出空间继续使用.因此解决方案为使invalidate acknowledge更快到达,因此每个cpu引入了一个 invalidate queues.

### Invalidate Queues
CPU可以在没有真正invalidate一个cache之前先发送invalidate消息然后将要invalidate的cache放到一个queue中

放入invalidate queue之后就可以发送invalidate acknowledge,但是继续对invalidate queue中涉及到的cacheline操作之前,必须先将invalidate queue中的数据先处理完毕

类似store buffer的引入是为了解决cpu stall,但会导致一些问题,因此需要memory-barrier,同理,invalidate queue的引入也会导致一些问题.

```
void foo(void)
{
	a = 1;
	smp_mb();
	b = 1;
}

void bar(void)
{
	while (b == 0) continue;
	assert (a == 1);
}

```


继续看这段代码,假设a是shared,b被cpu0拥有(处于exclusive或者modified状态).假设CPU0执行foo()并且cpu1执行bar()

加了invalidate queue之后该段代码还是会导致断言失败,因此需要继续修改

```
void foo(void)
{
	a = 1;
	smp_mb();
	b = 1;
}

void bar(void)
{
	while (b == 0) continue;
	smb_mb();
	assert (a == 1);
}
```

此处的memory-barrier的作用是:在继续执行断言之前,必须先处理完毕invalidate queue中的信息.因此可以保证断言成功


## Read and Write Memory Barriers

read memory barriers专门处理invalidate queue相关,而write memory barriers处理store buffers相关

因此上述代码修改为:
```
void foo(void)
{
	a = 1;
	smp_wmb();
	b = 1;
}

void bar(void)
{
	while (b == 0) continue;
	smb_rmb();
	assert (a == 1);
}
```

## 小结

论文中举了很多示例,图文并茂,很好理解.需要详细了解memory barriers的可以阅读论文.
通过上文的讲解,可以看到,CPU的优化是一个按下葫芦起了瓢的过程,每个优化引入一个新的组件,新的组件又会导致一个新的问题.其实所有的架构设计都会面临这样一个取舍问题.

