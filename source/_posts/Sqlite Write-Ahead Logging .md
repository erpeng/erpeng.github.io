---
title: Sqlite Write-Ahead Loggin
date: 2019-04-16
tags: Sqlite
---

>这篇文章译自Sqlite官方文档,介绍另一种保证Sqlite原子性的机制,Write-Ahead logging


## Sqlite WAL
原文链接:https://www.sqlite.org/wal.html

### 1.概览

Sqlite中默认保证原子性的机制是rollback journal,从3.7.0(2010-07-21)开始,Sqlite引入了另一种机制,Write-Ahead logging,简称为'WAL'.

WAL和rollback journal相比有如下四方面的优点:
* 在大多数场景下WAL要比rollback journal快
* WAL模式下,读不阻塞写,写也不阻塞读,读写可以并行执行,所以并发性能更好
* WAL模式倾向于顺序I/O
* WAL模式下fsync()的刷盘操作会更少

WAL相比rollback journal也有如下的9方面缺点:
* WAL需要VFS层面支持共享内存原语,有些操作系统可能并不支持
* 所有使用数据库的进程必须处于同一台机器,WAL在网络文件系统上不能工作(例如NFS)
* 如果一个事务中包括多个数据库文件(通过attch命令),WAL模式下只能保证每个数据库的执行是原子性的,但不保证全局的原子性
* 即使是在一个空数据库或者通过使用vacuum或者通过backup api恢复一个数据库的情况下,要想修改page size必须在rollback journal模式下
* 从3.22.0(2018-01-22)开始,如果-shm和-wal文件已经存在或者这两文件能够被创建或者数据库是immutable的情况下,能够打开一个read-only的wal模式下的数据库文件
* WAL模式在大部分读只有极少数写的情况下,会比rollback-journal模式慢1%-2%
* 因为有一个准持久化的-wal文件和一个-shm共享内存文件,导致Sqlite作为一个 applicate file-format增加了一些复杂度
* WAL模式下有一个checkpointing机制(自动执行),需要应用开发者引起注意
* 从3.11.0(2016-02-15)开始,WAL模式在大事务情形下和rollback-journal模式同样有效.但在之前的版本,大事务情行下WAL会慢一些

### 2.实现机制

传统的rollback journal在修改前先将历史数据保存在rollback journal中,然后修改数据库文件.在回滚时,直接将rollback journal中的历史数据覆盖掉被修改的数据库文件,从而恢复到数据修改之前的状态.当删除rollback journal后一个事务就结束了.

WAL和rollback journal是反过来的,数据库文件并不会直接修改,而是将修改append到一个WAL文件中,当事务的记录都已经append到WAL文件之后,事务就可以结束了.因此读操作可以继续读原来的数据库文件,多个写操作也可以同时往一个WAL文件进行append操作.(**回滚操作只要将append到WAL的内容清除即可**)

#### 2.1checkpointing

把WAL文件中的事务逐步写回数据库文件的过程称为'checkpoint'(**为什么需要写回呢?考虑读取的情况,读取时除了需要读取数据库文件,也得读取WAL中已经提交完成的事务所做的修改,当WAL很大时会影响读取效率**)

checkpoint可以通过配置SQLITE_DEFAULT_WAL_AUTOCHECKPOINT该选项决定WAL多大时执行自动的checkpoint.默认为1000 pages.当然也可以关掉该功能,通过另外的线程在空闲时去执行checkpoint.

#### 2.2并行性
当开启一个读取操作后,首先需要记录一下当前最后一个在WAL提交的有效记录,记为'end mark'.因为WAL日志在不停增长,因此不同的读取操作会有自己的'end mark'.但在整个事务中,每个读取操作的 'end mark'不会改变.

读取一个page时,首先从WAL中(本读取操作的'end mark'之前)获取一份最新的page,如果没有找到则从数据库文件中获取.为了防止读取操作每次都遍历WAL获取一个page(WAL可能会很大),在共享内存中提供了一个'wal index'的结构,能够快速定位WAL中的一个page.这也是为何WAL机制的所有连接必须处于同一台机器,并且不能通过网络文件系统使用的原因.

写入操作只是往WAL文件append记录,可以看出读写互不影响

checkpoint从WAL的开头往后顺序执行,直到到达任意一个正在执行的读取操作的'end mark',此时停止checkpoint(**如果读取超出end mark的数据并且回写到数据库文件,会使读取操作读取到不该看到的内容**).注意该停止点也保存在共享内存的wal-index中,下次启动时从此处继续执行.可以看出,一个运行很长时间的read transaction会阻止checkpoint的运行.

当一个写操作执行的时候,如果发现整个WAL文件都被checkpoint完成并且进行了刷盘并且当前没有任何读取操作的话,会重新开始从WAL的头部开始写入事务.从而避免WAL文件一直增长

#### 2.3性能考虑 
写入时由于只需要写一次(rollback journal需要写两次)并且是顺序写,因此性能会高一些
读取时需要遍历WAL文件,虽然有共享内存中的WAL-index,随着文件增大,读取性能还是会受影响.因此为了获得好的读取性能,建议定期执行checkpoint.
checkpoint时需要进行数据库文件的刷盘和寻道操作,虽然checkpoint尽量做到顺序写入page(**如何做到**),但是仍然需要花费很多寻道的时间,因此checkpoint会比写入要慢.
 
默认情况下,当WAL达到1000pages时会将WAL进行checkpoint,并且该动作由执行写操作的进程自己负责,这可能会导致大部分情况下写操作很快,但有个别事务因为执行checkpoint导致变慢.如果不希望受此影响,可以关闭自动checkpoint,由另外的线程或进程定期去执行.

读写性能可以通过配置checkpoint的页数达到一个折衷.如果频繁checkpoint,读性能会高一些,但如果尽量减少checkpoint,写性能会高一些.可以根据实际需求来配置

### 配置WAL模式
```
	PRAGMA journal_mode=WAL;
```
该配置会设置为WAL模式.如果VFS不支持共享内存原语,则会继续使用 'journal_mode = delete'即rollback journal模式

#### 3.1自动checkpoint
当WAL文件中发生一个commit之后会调用通过sqlite3_wal_hook()注册的回调函数.例如可以使用sqlite3_wal_checkpoint()或者sqlite3_wal_checkpoint_v2()来执行checkpoint.自动checkpoint也是在sqlite3_wal_hook()之上简单的进行了包装.

```
pragma wal_autocheckpoint
```
该配置用来修改自动执行checkpoint的配置

#### 3.2应用启动的checkpoint

通过调用sqlite3_wal_checkpoint可以启动一个passive类型的checkpoint,该类型的checkpoint不会影响其他连接的读写操作.
通过调用sqlite3_wal_checkpoint_v2可以启动一个full或者restart类型的checkpoint,这两种类型会将checkpoint执行直到完成

#### 3.3WAL配置的持久性
当配置如下时
```
PRAGMA journal_mode=WAL
```
即使关闭后重启database,仍然是WAL模式.
但如果PRAGMA journal_mode=truncate,关闭重启后就会返回为delete.(**不知为何**)

### WAL文件

当在WAL模式下打开一个数据库连接后,会生成一个WAL文件,命名方式为数据库文件名称加'-wal'后缀.

只要有数据库连接打开数据库,WAL文件就会存在.通常来说,当数据库上最后一个连接关闭之后,WAL文件会被自动删除.如果最后一个连接没有显示关闭连接或者SQLITE_FCNTL_PERSIST_WAL被使用,WAL文件将会在磁盘上一直存在.WAL文件需要合数据库文件保存到一块,否则可能会发生数据丢失或者数据库文件损坏.删除一个wal文件最安全的方法是首先调用sqlite3_open()打开一个数据库文件然后调用sqlite3_close()关闭该文件.

WAL文件格式是严格定义的并且跨平台.

### read-only数据库

从3.22.0(2018-01-22)开始允许一个wal模式的数据在只读媒介上打开.只需要满足如下三个条件:
* -shm和-wal文件已经存在并且可读
* 包括-shm和-wal文件的目录可写,以便创建这两文件
* 数据库连接 以'immutable query parameter'打开

虽然允许打开一个wal模式下的只读数据库,最好是在将sqlite部署到只读媒介时将pragma journal_mode修改为delete

### 如何避免生成过大的wal file
* 关闭自动checkpoint后可能会造成 wal file过大
* 当有大量的读操作时会造成checkpoint不能执行完成,可能导致wal file过大.此时应该考虑手动checkpoint（使用SQLITE_CHECKPOINT_RESTART或者SQLITE_CHECKPOINT_TRUNCATE选项）.
但注意这样会造成读操作在执行checkpoint时阻塞
* 当有一个非常大的写事务时会导致chedkpoint没法执行完成.导致wal file 过大.从sqlite 3.11.0(2016-02-15)开始,被一个事务修改的pages只会往wal文件写一次.旧版本中,如果一个事务使用数据大小超过page cache的话,会导致同一个page可能被多次写入wal file.

### wal-index在共享内存中的实现

wal-index通过将一个普通文件mmap到内存中实现.之前是将wal-index通过在/dev/shm(linux)或者/tmp(unix)中创建一个文件来实现.但这种方法会导致有不同根目录(通过chroot)的进程看到不同的文件从而使用了不同的共享内存,导致数据库损坏.其他的创建匿名共享内存的方法又不够通用.因此现在直接通过在数据库文件所在目录创建一个普通文件然后映射到内存中实现共享.

这种方法也有缺点,从共享内存刷新到磁盘可能会导致不必要的磁盘I/O.但Sqlite开发者不认为这是个问题,因为wal-index很少会超过32KB并且从不会被刷盘.进一步,wal-index 文件在最后一个数据库连接断开之后会被删除,因此不会有任何实质性的disk I/O会发生.

如果有些应用觉着这种方法不可接受,也可以自己定制.例如:如果应用只会通过在一个进程中的不同线程访问,那么wal-index可以实现到堆内存中而不是使用共享内存.

### 不适用共享内存的wal模式

SQLite3.7.4 (2010-12-07)之后,如果locking_mode在第一个请求到来之前设置为exclusive,则wal模式可以在没有共享内存的情况下使用.换句话说即保证只有一个进程会访问数据库.

### 请求返回SQLITE_BUSY的情形

大部分情况下读写请求互不阻塞,但有些情况下可能会返回SQLITE_BUSY.

例如:

* 如果一个连接以exclusive mode模式打开数据库,其他请求都会返回SQLITE_BUSY.CHROME和FIREFOX都是以exclusive mode打开的数据库,所以如果其他视图打开读取chrome和firefox数据库的请求都会返回SQLITE_BUSY.

* 当最后一个连接关闭的时候,该连接会短暂的获取一个exclusive lock以便清理WAL和shared-memory文件,如果第二个请求在此时想要打开该数据库文件,也会返回一个SQLITE_BUSY错误.

* 如果最后一个连接crash,则新的连接会首先执行一个恢复流程.该流程中也会持有一个exclusive lock,此时如果另一个连接试图读取也会返回一个SQLITE_BUSY错误.


### 后向兼容性

为了防止旧版本的Sqlite(3.7.0之前,2010-07-22,旧版本的Sqlite并不能识别wal文件和共享内存中的wal-index)试图恢复一个wal模式的数据库,数据库文件头中的file format版本号(18-19字节)从1更新为了2(WAL模式下).因此如果一个旧版本的Sqlite尝试连接一个Sqlite数据库文件时,会报一个"file is encrypted or is not a database"的错误.
```
PRAGMA journal_mode=DELETE;
```
使用该配置可以更改WAL模式为roll-back journal模式
此时版本号会从2改回1,因此旧版本Sqlite可以继续使用该数据库.









