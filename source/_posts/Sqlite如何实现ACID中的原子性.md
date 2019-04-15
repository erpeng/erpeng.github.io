---
title: Sqlite如何实现ACID中的原子性
date: 2019-04-13 09:42:00
tags: Sqlite
---

>这个文章序列关注Sqlite的原子性实现.首先翻译一篇官方文章,介绍Sqlite的 rollback journal,然后结合一些问题通过源码分析一下具体实现.后续几篇关注Sqlite3.7.0之后的另一种保证原子性的实现 WAL(write ahead log).
> 注意Sqlite以静态或者动态库的形式提供给程序调用,并不是C/S模式.另外,Sqlite每个库都保存在一个单独的文件之中.通过attch命令能够将其他文件中的库引入.

## Sqlite如何实现ACID中的原子性
原文链接:https://www.sqlite.org/atomiccommit.html
翻译中有些无关的省略了.具体信息可以参看原文

### 简介
关系型数据库需要实现ACID,其中A-Atomic commit,即单个事务中的语句要么全部执行要么一个都不执行.如果在事务的执行过程中操作系统crash或者掉电,事务仍旧能够保证原子性.本篇文章只适用于Sqlite配置为rollback mode的时候.(还有一种保证原子性的模式为开启wal-write ahead log,后文另述)

### Sqlite实现中对硬件的一些假设
磁盘以sector为单位进行写操作.如果要修改小于一个sector的区域,也得首先把包括待修改区域的整个sector读取出来,修改完毕后再将整个sector写回去.

传统的机械硬盘读和写都是以一个sector为单位,在ssd中,读的单位要比写单位要小很多.Sqlite只关注最小的写单位,所以本文中sector都是指最小的写单位.

Sqlite 3.3.14之前默认一个sector的大小为512bytes...

Sqlite假设一个sector的写入并不是原子性的,从3.5.0版本开始,sqlite实现了一个新的接口VFS(virtual file system),VFS用来和底层的文件系统交互.VFS接口中有一个方法xDeviceCharacteristics,通过该方法可以获取到底层sector的写入是否是原子性的.如果是,则 **sqlite能够利用该特性做一些写入的优化**

Sqlite假设一个写请求并不会直接落盘而是被操作系统先缓存起来.所以在一些节点需要执行flush或者fsync将数据真正刷新到磁盘.

Sqlite假设一个文件写入时先更新文件大小然后写入具体内容,**因此Sqlite做了一些额外的工作去保证在更新文件大小和写入文件内容之间如果发生掉电的话不会导致数据库的损坏**.VFS中的xDeviceCharacteristics函数也能够获取到文件系统是否会首先写入内容然后更新大小（通过SQLITE_IOCAP_SAFE_APPEND这个属性）,如果具有该属性,Sqlite就不需要做额外的一些保护措施,减少磁盘I/O.

**Sqlite假设文件的删除操作是原子性的,即如果删除过程中发生了掉电,机器恢复后要么完全看不到该文件,要么文件是完整的,不会出现删除了部分的情况(注意只是用户视角,即用户进程询问操作系统一个文件是否存在时,操作系统只会给出yes or no,而不会有部分存在的回复)**

Sqlite不会在数据文件中增加冗余信息去校验文件是否损坏.

Sqlite假设操作系统写入一个范围时不会因为掉电或os crash而改变该范围之外的数据.该功能称为 powersafe overwrite.

### 单文件的提交
首先全局观看sqlite在单文件状态下的原子性保证.后文讨论多文件状态下的原子性保证.
#### 初始状态

左边是用户进程的内存空间,中间是操作系统缓存,右边是磁盘存储.每个小框代表一个sector,蓝颜色的sector表明存储的是原始数据.

![图0](/img/s0.gif)

#### 获取一个读锁

在写入数据库之前,首先会读取数据.即使是插入数据也得先读取**sqlite_master**获取表的schema和决定将数据存储于哪个位置.

首先获取一个共享锁,一个数据库文件可以同时有多个共享锁,但是当存在共享锁时,其他连接获取不到写锁.

注意共享锁存在于操作系统的缓存中,而非磁盘上.因此当操作系统crash或者掉电或者创建共享锁的进程退出时锁会自动释放

![图1](/img/s1.gif)

#### 从数据库文件读取信息

获得共享锁后,首先从磁盘读取数据到操作系统的缓存,然后从操作系统缓存到用户空间内存中,

通常只有部分pages会读取到,本例中我们共有8个page,但只读取了3个.真实场景中可能会有成千上万个pages

![图2](/img/s2.gif)

#### 获取一个reserved lock

在写入之前首先获取一个reserved lock,注意此时其他连接仍然能够获取shared lock,但不能进一步获取reserved lock,即reserved lock在一个数据库文件中只能有一个存在.

这个锁存在的意义在于说明有进程即将写入一个数据库文件,但是还没有开始写入.此时其他进程如果也尝试写入的话会失败(因为获取不到reserved lock)

![图3](/img/s3.gif)

#### 创建rollback journal file

在修改数据库文件之前,Sqlite首先创建一个独立的rollback journal file,并且将要修改的pages写入rollback journal file.(**注意是以page为单位写入**)

rollback journal file中绿色的部分是header,header中会包括数据库文件的原始大小(**即包括多少个page**).每一个page保存到rollback journal中时前边四字节会保存该page的page number.

![图4](/img/s4.gif)

图4中有两个地方需要注意:

* rollback journal创建之后并没有实际落盘,只是保存在操作系统的缓存中
* rollback journal以page为单位,但每个page前四字节会有一个page number的记录,后四字节有一个checksum

#### 在用户空间修改数据库文件

![图5](/img/s5.gif)

修改用户进程空间内的数据库文件,注意不同的用户进程有自己的私有内存空间,因此此时的修改并不影响其他进程的读取操作

#### 将rollback journal落盘

将rollback journal刷新到磁盘,该步骤是保持原子性很关键的一步,保证即使掉电或者操作系统crash,Sqlite也能恢复到原来的状态.

(**进行到该步也能看出reserved lock的作用,这种锁是一个中间状态,既能为即将写入做准备,又不影响其他进程的读取操作,提高并行度**)
![图6](/img/s6.gif)

该步需要刷两次盘,第一次将rollback journal的内容刷到磁盘,第二次在header中记录第一步中刷到磁盘的page个数,然后将header刷盘

#### 获取排他锁

![图7](/img/s7.gif)
在实际写入数据库文件之前,需要获取一个排他锁.获取过程分两步
 * 获取一个pending lock
 * 将pending lock升级为排他锁

获取到pending lock之后,在数据库文件上已经获取到共享锁的进程可以继续读取,但不允许其他进程继续获取共享锁.该锁存在的意义在于防止write starvation(即有大量的读取连接时,一直有新的共享锁产生,导致获取不到排他锁).当所有已经存在的共享锁都释放后,此时该pending lock即可以升级为排他锁

#### 写入数据库文件

![图8](/img/s8.gif)

获取到排他锁后,说明已经没有其他进程在读取该数据库文件.此时可以安全的写入数据库文件.注意也只是写入操作系统的缓存中,并没有落盘 

#### 数据库文件刷盘

![图9](/img/s9.gif)

此时将数据库文件刷新到磁盘.

#### 删除rollback journal

![图10](/img/s10.gif)

因为数据库文件已经安全落盘,此时可以删除掉rollback journal.若删除之前系统crash或者掉电,则重启后会恢复到事务开始前的状态,如果删除之后系统crash或者掉电,因为数据库文件已经落盘,相当于事务已经执行完成.(**那么会不会在删除一半时系统crash或者掉电呢?注意上文中关于硬件的一些假设,删除操作必须是原子性的,即不会发生这种情况**)

因为删除一个文件也是一个耗时的操作,因此Sqlite提供了两种方式减少删除过程的耗时.
* 将一个文件truncate为0
* 将journal file header清0.清0操作并不是原子性的,但只要header中有一个byte被清0,该文件就会被识别为无效的格式

#### 释放锁

![图11](/img/s11.gif)

此时可以释放掉排他锁.

注意图11中的用户空间缓存也已经被清除.但新版本的Sqlite并不会清除用户空间缓存,以免下一个事务开启式需要重新读取该数据.但在复用该缓存信息时需要首先获取一个共享锁然后检测是否在当前事务开启之前有其他事务已经修改过数据库文件.如果已经修改过则不能复用.检测修改的逻辑也很简单,数据库文件的第一个页中保存了一个计数器,通过比较该计数器即可知道是否发生过修改

### 回滚

#### 出错时
![图12](/img/sr0.gif)


假设将数据库文件刷盘时掉电,会出现图12所示的情况.我们本来需要修改三个page,但这时只修改成功一个page,另一个page只写了一部分,而第三部分还没开始写

注意此时rollback journal已经安全落盘但还没删除

#### hot rollback journal
![图13](/img/sr1.gif)

掉电或操作系统crash恢复后,Sqlite会首先获取一个共享锁,然后检测是否有hot rollback journal存在.注意hot rollback journal只有在一个事务没有提交完成时存在.那么如何检测是否是一个hot rollback journal呢,我们通过检测如下几点来决定:

* rollback journal file存在
* rollback journal file不为空
* rollback journal file header格式正确,即没有被清0
* rollback journal file 指定位置没有master journal file的名称,或者如果有一个master journal的名称并且这个master journal也存在(后文叙述,涉及到多文件的事务)

一个hot journal的存在说明数据库处于一个不一致的状态.在读取之前需要先进行修复

#### 获取排他锁
![图14](/img/sr2.gif)

进行数据库恢复之前先获取一个排他锁.防止多个进程同时修复一个数据库文件.

(**注意此时不需要先获取reserved lock或者pending lock,此时需要立刻进行修复,不需要考虑其他进程的读取.但假设此时有其他进程持有shared lock,会直接获取到排他锁还是会等待shared lock失效?猜测应该为前者**)

#### 回滚

![图15](/img/sr3.gif)
获取到排他锁后可以开始执行恢复.分两步:
* 将rollback journal中的相应page写回到database file
* rollback journal中有记录database file的原始大小,将database file truncate 到原始大小.

此时数据库内容和大小都会恢复到原始状态

#### 删除hot journal
![图16](/img/sr4.gif)

当数据库文件恢复并且落盘后,可以清除掉hot journal

#### 继续读取
![图17](/img/sr5.gif)

将排他锁降级为共享锁.此时崩溃的事务好像没有发生过一样

### 多文件事务

Sqlite允许一个数据库连接中通过attach database命令同时操作多个数据库文件.当多文件在一个事务中修改时,Sqlite保证其原子性.即要么所有文件中都更新成功,要么所有文件都不更新.

#### 每个文件都有单独的rollback journal

![图18](/img/sm0.gif)

图中每个文件有单独的reserverd lock和rollback journal,类似单文件情形.但此时rollback journal 并没有刷盘,数据库文件也没有更新 

#### master journal file

![图19](/img/sm1.gif)

下一步需要创建一个master journal file.文件名称仍然是使用sqlite3_open打开的原始数据库文件名称,但会增加一个-mjHHHHHHHH的后缀,HHHHHHHH是一个随机的32bit十六进制数字.每个master journal的的随机后缀都不相同

master journal中并不保证数据库文件的原始内容,保存的是每一个rollback journal的完整路径和名称

master journal创建完成后需要直接刷盘(**在Unix系统中,包括该master journal的目录也需要刷盘**)

master journal的存在是为了保证多文件事务的原子性.但如果Sqlite中设置了pragma synchronous=off 或者 pragma mode=memory,则并不会创建master journal.当然在此配置下也不能保证完整性

#### 更新rollback journal的header

![图20](/img/sm2.gif)

更新每一个rollback journal的header,将master journal的完整路径及名称写入rollback journal的header（单文件事务中该字段为空）.在更新header之前和之后都需要将rollback journal进行刷盘(**此处类似单文件事务中刷新纪录之后再刷新纪录的计数,Sqlite需要确保在header写入成功之前content已经写入成功**).

#### 更新数据库文件 

![图21](/img/sm3.gif)

在每个数据库文件都获取到排他锁之后更新数据库文件并且刷盘

#### 删除master journal file
![图22](/img/sm4.gif)

此时可以删除master journal file.如果此时发生了掉电或者操作系统crash,根据之前hot journal的认定规则,此时也不会进行回滚,虽然仍然存在各个数据库文件的rollback journal.

#### 清除rollback journal file

![图23](/img/sm5.gif)
删除rollback journal并且释放排他锁.

### 其他commit时的细节

#### page和sector

Sqlite写入rollback journal时必须是按sector大小的整数倍写入.sector在大部分情况下为512bytes,page为4096bytes,此时按page写入没有什么问题.
但是如果sector为4096bytes,而page为512bytes,此时即使只修改一个page,也会将该page属于的sector整个写入rollback journal.

#### 处理journal中的垃圾内容

前文中提到过Sqlite的一些硬件假设,其中一条为文件首先扩充大小然后将真正的内容写入.如果文件扩充大小之后,写入真正内容之前发生了掉电,此时journal中会包含一些垃圾信息,此时rollback会将垃圾信息也写入数据库文件,从而发生错误.

为此Sqlite做了一些保护性措施,其一为在rollback journal的header中记录该rollback journal包括的page的个数,并且初始化为0.只有当真正的内容刷盘之后,会将header中的记录个数更新并且刷盘.注意header不论大小都会占用一个sector的大小,因此可以独立刷新,不影响journal的内容.

Sqlite默认是如下配置
```
	PRAGMA synchronous=FULL;
```
如果修改该配置的值为NORMAL以下,则Sqlite对journal的刷盘流程修改为只刷新一次.如下:
```
	write content -> write header ->flush
```
虽然header的写入仍然在content之后,但不保证操作系统实际刷新时可能会先刷新header,再刷新content.因此sqlite在每个rollback journal content的page内容之后附加了4字节的checksum.即rollback journal的格式如下
```
	| header | pgno|page content|checksum|pano|page content|checksum|
```
rollback时会检查每个page content的checksum是否正确,如果不正确则放弃该次rollback.

#### cache spill
当大量的更新导致用户空间的缓存不足以保存所有需要更新的page时会有cache spill的情况发生.
cache spill首先刷新journal file到磁盘,然后获取一个排他锁, 然后写入数据库更新.此时因为更新还没有全部完成,需要在追加一个header到journal file,循环执行如上三个步骤(第二步获取排他锁因为已经完成,实际上不需要再次获取),直到所有更新完成.因为cache spill会增加占用排他锁的时间并且会有更多的io,会严重影响性能.因此应该尽量避免cache spill

### 优化措施

Sqlite耗时大部分在io上, 因此本节主要考虑如何在保证原子性的前提下能够降低io

#### 事务之间的缓存保存

上文也提到过,在数据库文件的database file第一个page的24-27字节有一个counter字段,每次数据库发生更改之后都会更新该字段.因此sqlite在一个事务执行完成需要释放锁时,会首先获取该字段的值,当下一个事务获取锁之后会比较之前缓存的counter值和当前的counter值,据此决定是否可以复用该缓存

#### 排他性访问模式

适合于单进程访问的情况,不详述

#### freelist page不写入journal

当Sqlite删除一些数据后,只保存删除信息的page会放入freelist.后续的写入会首先从freelist中获取可用的page而不用增加数据库文件的大小

Sqlite中的freelist有两种类型,trunk page和leaf page.leaf page不包含任何有用的信息.因此如果Sqlite的修改涉及到leaf page,leaf page不需要写入journal(因为leaf page中没有任何有用信息).该步能够降低io

#### 单页修改以及原子性的sector写入

如果一个磁盘支持ector的原子性写入并且page大小等于sector,并且一次修改只涉及到一个page.那么此时会直接写database file而不用journal.

#### safe append 

如果一个磁盘支持safe append,那么journal file的header部分写入的page records数量永远为-1,不需要二次刷盘.实际记录由journal的size大小计算得出.并且发生cache spill时也不需要在与原来的journal后边追加header.直接追加content即可

#### 持久化rollback journal
```
PRAGMA journal_mode=PERSIST;
```
当配置此项之后,commit事务时不删除journal file,而是将journal file的header清0.该方法会节省如下几步:
* 不需要更新inode中的journal file文件大小
* 不需要处理释放后的磁盘空间
* 下一个事务可以直接覆盖该journal file而不是创建或者追加文件内容,覆盖会比追加更快
* 不需要更新包括该文件的目录信息

```
PRAGMA journal_mode=TRUNCATE;
```
该模式会将一个文件大小truncate为0.由于不需要更新文件所属目录信息,因此truncate比delete要快.
但truncate比persist模式还是会慢,因为覆盖文件要比追加文件快.

### 测试Sqlite的原子性

因为Sqlite开发者使用了大量的自动化的密集的'crash tests'模拟Sqlite在掉电或者系统crash时后的恢复情况,所以对Sqlite的可靠性很有信心.

Sqlite的'crash tests'使用一个修改过的vfs模拟系统crash或者掉电后的文件系统损坏.例如sector只写了一部分,写操作未完成导致pages中有垃圾信息,写操作乱序等等不同的节点.通过模拟不同的场景一遍遍测试Sqlite事务,验证其最终是否能够保持原子性.

'crash tests'已经发现了许多Sqlite恢复机制中的微妙的bug(都已经修复),这些bug光靠代码分析和审查是很难发现的.(总之是Sqlite开发者对Sqlite的原子性保证很有信心)

### 可能发生问题的一些情况

#### 锁的实现

Sqlite使用文件系统锁去保证同时只有一个进程或者连接操作数据库文件.文件系统锁在VFS层实现并且不同的操作系统有不同的实现方法.如果文件系统锁实现的有问题导致同时有多个进程或连接修改同一个数据库文件就会发生严重的损坏.

....(举例说明NFS这种网络文件系统就可能会导致发生损坏)

#### 刷盘的实现

Sqlite在Unix使用fsync,windows使用FlushFileBuffers去刷盘,但是这两个函数在许多系统上可能并不可靠.例如我们收到反馈说Windows可以使用注册表禁用禁用掉FlushFileBuffers,一些老的Linux版本的fsync在有些文件系统下不执行任何操作.即使fsync和FlushFileBuffers能够正常工作,一些IDE磁盘执行之后也只是把数据保存在控制器缓存中而不是真正落盘.

....(Mac相关的配置...)

#### 文件删除操作不保证原子性

文件删除操作从应用视角看需要保持原子性,即删除部分数据后掉电的话应用要么完全看不到该文件要么需要看到一个完整的文件,不能只看到删了部分的文件(rollback journal如果只删了一半,会执行恢复操作,但是因为journal已经不完整了,恢复的数据也会不完整)

#### 文件中有垃圾数据

Sqlite的数据库文件就是一个普通的磁盘文件,如果文件中被其他进程写入垃圾数据,或者其他原因导致垃圾数据产生,会导致数据库文件不可用

#### 删除或者重命名一个journal file

journal file和数据库文件的命名有一定的关系,如果数据库文件或者journal file的名字被修改了,会导致没法roll back.

(举了一些可能的例子....)
Sqlite将journal file刷盘的同时也会刷新journal file所在的目录,以免修改名称然后掉电导致重启后找不到该文件...

### 未来的一些方向和结论

虽然Sqlite保证原子性的这个机制bug越来越少,但最好还是保持警惕,发现问题后Sqlite开发者会尽快修改.

Sqlite开发者也在尽量优化提交过程.现在Sqlite都会对操作系统的VFS实现做一些悲观的假设,然后从Sqlite层面去保证正确性.例如现今许多现代的操作系统都能保证safe append和原子性的sector write,但这些功能必须很确定之后, 我们才会从Sqlite层面减少对这些功能做的一些正确性保证.



