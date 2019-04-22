---
title: Mysql redo binlog 主从及其延迟
date: 2019-04-20 
tags: Mysql
---
## 引言
通过构造示例,理解Mysql的redo log,binlog,主从及其延迟
参考极客时间**mysql45讲**之第2讲,12讲,15讲,23讲,24讲,25讲,26讲,27讲,28讲

## redolog与binlog

### redolog
* 固定大小,循环写入
* redolog相比直接在磁盘中写入修改的好处是将随机写转换为顺序写,并且有组提交的优化
* checkpoint是redo中的一个位置,该位置之前的redolog可以覆盖掉,因为已经写入了磁盘

### binlog
* statement 完整的语句
* row 行的内容 update时是一行修改前的一行修改后的  insert和delete也是会将完整的数据记录下来
* 因为statement格式可能会导致主从上执行该语句时产生不一致,所以有了row格式.但是row格式占用空间过大,所以有了mixed格式,由Mysql决定是使用statement还是row
* row格式有个好处是会保存完整的数据,所以只要反向操作就可以恢复数据.因此还是建议保存为row格式
* 事务提交的时候才会刷binlog,如果一个事务很大,必须等事务执行完成才写binlog,才会传给从,会导致主从延迟过大
```
回放mysql binlog
mysqlbinlog master.000001  --start-position=2738 --stop-position=2973 | mysql -h127.0.0.1 -P13000 -u$user -p$pwd;
```

### redolog与binlog的差异

* 一个是引擎层的日志,一个是server层的日志
* 一个是物理日志,即某个数据页做了什么修改;一个是逻辑日志,记录语句的原始逻辑
* 一个是循环写,一个是追加写

### 两阶段提交
* 写入redolog,处于prepare状态
* 写入binlog
* 提交事务,处于commit状态

处于commit状态的直接恢复,如果处于prepare但是未commit的,根据XID找binlog中的事务是否完整,完整则认为该事务是可以提交的,如果不完整则回滚之
```
innodb_flush_log_at_trx_commit = 1 即每次事务提交时的redolog都落盘
innodb_flush_log_at_trx_commit = 0 即每次事务提交时的redolog都只在redo log buffer中
innodb_flush_log_at_trx_commit = 2 即每次事务提交时的redolog都只write,在操作系统的page cache中
innodb有一个后台线程会定期将redo log buffer中的数据write到page cache,然后fsync到磁盘.即使一个事务未提交也会刷盘


sync_binlog = 1 每次事务的binlog都落盘
sync_binlog = 0 只write不fsync,由操作系统定期去fsync
sync_binlog = N (N>=2)只write,累计N个事务后才会fsync
```
### fsync的优化
由于fsync很耗时,所以redo有一个组提交的优化,刷新时会将其他事务写的log一起刷新到磁盘


## MySQL为什么有时候会"抖"一下

为什么某条SQL语句会突然变慢了
* redo过小,导致write pos即将超出checkpoint,所以需要刷新脏页,推进checkpoint
* 内存空间buffer pool过小,导致需要刷脏页 
  mysql中如果内存中有数据,则数据肯定是最新的,如果内存中没有数据,则磁盘中的文件是新的(得经过change buffer merge之后)

注意还有两种情况也会刷新脏页:
* 系统空闲时,刷脏页频率快导致
* 正常关机也会刷脏页

```
innodb_io_capacity 该参数定义mysql所在主机的io能力 SSD可设置为20000,机械硬盘为300
innodb_max_dirty_pages_pct控制脏页比例上限
Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total计算脏页比例
根据最新写入日志的lsn(log sequence number)和checkpoint相减的值判断checkpoint进度
innodb_flush_neighbors控制是否在刷脏页的时候刷新邻居页,SSD建议关闭
```

## 主从延迟
* 备库配置差
* 备库配置同主库,但压力过大,例如执行了大量的报表查询及备份等工作.可以通过设置一主多从来分摊压力
* 大事务,即如果主上边执行10分钟,从上也可能执行10分钟,导致延迟.例如删除大量数据,再例如大表的DDL,会占着表的MDL写锁,导致没法对该表进行读写,导致主从延迟
* 从库的并行复制能力

### 并行复制
* io thread负责将内容写入relay log,sql thread负责回放
* slave_parallel_workers设置5.6之后并行回放的线程个数
基本原则:
* 以transaction为基本单位,否则会影响事务的隔离性
* 对同一行更新操作的不同事务需要放到同一个worker

按表分发:
* 每个worker有一个hash表,记录db-table:事务个数
* 分发时,如果没有任何冲突,直接分配给空闲worker
* 如果有多个worker有table的更新,则等待直到只有一个worker更新该表
* 分配给该worker

如果碰到热点表,会退化为单线程模式

按行分发:
* 每个worker同样有hash表,记录db-table:各个唯一键+各个唯一值.有多少唯一值记录多少项


mysql5.7:
slave-parallel-type
* DATABASE 按库并行复制
* LOGICAL_CLOCK 模拟主库的组提交,同时处于commit状态的肯定可以并行

##主从切换

找同步位点

主动跳过一个事务:
```
set global sql_slave_skip_counter=1;
start slave;

```
```
slave_skip_errors 跳过1032/1062错误,即插入时唯一键冲突/删除时找不到行
```
上述两种方式tricky,并且容易造成错误

5.6可以开启gtid:
```
gtid_mode=on
enforce_gtid_consistency=on
```
开启gtid模式
如果有冲突,跳过冲突的gtid方法:
```
set gtid_next='aaaaaaaa-cccc-dddd-eeee-ffffffffffff:10';
begin;
commit;
set gtid_next=automatic;
start slave;
```
该事务aaaaaaaa-cccc-dddd-eeee-ffffffffffff:10就以空事务的状态加入了从的已执行事务集合中

有了gtid,备库不需要指定位点,由主库根据差集取出备库需要执行的gtid,然后开始主从同步

## 解决过期读的问题

* 强制走主
* 判断sbm,seconds_behind_master,当没有延迟时再去读,但是间隔是s,可能会不精确
* 对比位点,确保同步过来的已经全部执行.只能确定已经同步的执行完了,有可能还有未同步的
* 对比获取到的gtid_set和回放到的gtid_set,判断是否已经全部执行,只能确定已经同步的执行完了,有可能还有未同步的
* 结合semi-sync(主的binlog必须已经传给至少一个从,从回复ack),一主多从不适用.
* 上述除了强制走主的方案,如果并发量大,可能会看到位点,sbm,gtid一直不同步的情况
终极大法:
* 等主库位点
```
select master_pos_wait(file, pos[, timeout]);
```
如果能够知道某条命令执行后的位点,在从库执行该命令,异常返回null,超过Ns返回-1,已经执行过返回0
所以如果>=0,则可以在从库开始查询了,否则放弃或者走主

5.7.6开始允许执行完后把gtid返回客户端
```
 select wait_for_executed_gtid_set(gtid_set, 1);
 session_track_gtids = OWN_GTID
但得从API中解析出来,具体得看相应的API接口
```
超时返回1,执行返回0










