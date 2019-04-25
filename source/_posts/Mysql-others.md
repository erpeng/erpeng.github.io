---
title: Mysql 其他
date: 2019-04-25
tags: Mysql
---
## 引言
理解Mysql的 order by,count(*),MDL语句,onlineDDl,insert即自增主键
参考极客时间**mysql45讲**之第6讲,13讲,14讲,16讲,17讲,29讲,31讲,33讲,39讲,40讲

## 重建表
```
可以减少数据空洞,将复用的page清除掉
alter table t engine=InnoDB；
```
* 持有MDL写锁
* 降为MDL读锁
* 做DDL操作
* 升级成MDL写锁
* 释放MDL锁
onlineDDL
重建过程中不再持有MDL写锁,而是将更新记录到一个临时缓冲区,最后重放.DDL操作过程中可以正常进行读写

## 统计总行数
* Myisam直接记录了总行数,innodb为什么不直接记录呢?
    **innodb实现了mvcc,同一行对不同事务可见性不同,因此在不同事务中统计的总行数也不同**
* show table status也可以显示table_rows,但是不准确.通过在N个page中统计行数取平均值然后乘以总的page数来计算

* 那么,如何保存计数呢?首先需求是这样的,获取计数并且展示最新的100条数据.
* 缓存保存:redis中保存计数,MySQL中保存数据.并发情况下会有计数中有但是数据没有或者反之的情况.
* 将计数更新和数据插入放入一个事务中,由事务的隔离性保证

### count(*),count(1),count(id),count(field)差别
* count语义:聚合函数,根据字段返回值判断,如果不是null,+1

count(id),count(field)innodb会将id和field取出来传给server层,server层做统计
count（1)不需要取值
count（*)有做专门的优化,意思为取行数
所以 count(*) 约等于 count(1) > count(id) > count(field)

**聚合是在server层做的?**

## 排序
* sort_buffer_size决定排序空间,如果排序数据太大,则使用文件
* max_length_for_sort_data该值决定需要查询的字段超过多长时会使用rowid排序


排序分为如下两种:
* 全字段排序 所有需要查询的字段都放到排序空间中,排序完成后直接返回
* rowid排序 只将主键和需要排序的字段放入排序空间,排序完成后通过主键回表查询获取需要查询的字段然后返回.多了一个回表的过程,但会减少或不适用文件排序

mysql优先选择全字段排序,因为回表会造成多余的io

## 随机取值
* order by rand()的实现原理
```
mysql> select word from words order by rand() limit 3;
```
1. 创建临时表,memory引擎,两字段,第一个字段是double类型随机数,依次取一个word生成随机数插入一行
2. 取出临时表中数据根据随机数字段排序
3. 取出排序后前三个位置的位置信息,从临时表取出返回
**这个方法随着行数增加很耗时**


直观考虑:
生成三个行数范围内的随机数,依次取出.例如三个数分别为X1,X2,X3
```
select word from words limit X1,1;
select word from words limit X2,1;
select word from words limit X3,1;
```

可优化为:
假设X1,X2,X3最小为X1,最大为X3
```
select word from words limit X1,X3-X1+1;
```
或者
```
select id1,word from words limit X1,1;
select id2,word from words where id > id1 limit X2-X1,1
select id3,word from words where id > id2 limit X3-X2,1
```

**这些方法也是大数据分页时常用的套路**

limit N,如果N比较小的时候,5.6以后会使用优先队列排序算法,通过构造元素为N的最大或者最小堆,遍历一遍数据,就能获取到limit指定的N条数据

## 如何判断一个数据库有故障

* 外部检测不够及时
* 直接select 1不行,因为无法判断innodb层是否可用
```
innodb的并发线程上限,达到之后会阻塞其他线程.进入锁等待的线程不计入该计数
set global innodb_thread_concurrency=3
```
* 查询语句不行,因为不能判断出来磁盘满这种情况
* 只能通过写入语句判断

* 内部检测
```
统计单次IO请求时间是否超过200ms
mysql> select event_name,MAX_TIMER_WAIT  FROM performance_schema.file_summary_by_event_name where event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;
```

因为该统计值影响性能,所以只开启需要的统计
```
mysql> update setup_instruments set ENABLED='YES', Timed='YES' where name like '%wait/io/file/innodb/innodb_log_file%';
```
## 误删库处理

### 通过delete删除
如果开启binlog_format=row和binlog_row_image=FULL,通过flashback可以恢复(修改binlog,反向操作,删改为insert)
如何预防呢?
* 将sql_safe_updates=on,delete或者update不带where或者where条件没有索引的话报错
* SQL审计,开启general log,查看每一条sql

### 通过truncate/drop删除

不能通过flashback,只能使用 全量备份+mysqlbinlog（跳过清表语句）
* mysqlbinlog 只能指定 -database但没法指定表
* 只能单线程执行

### 预防方法
* 搭建延迟备库 CHANGE MASTER TO MASTER_DELAY = N,保持跟主库有N秒的延迟

* 账号权限管控
1. 开发只有DML权限,不给truncate/drop权限
2. 通常只使用只读权限账号

* 操作只能通过管理平台进行
1. 删除表前必须先对表做改名操作,例如增加后缀 _to_be_deleted
2. 只能通过管理平台删除,并且只能删除固定后缀的表

### rm删除数据

* 数据做跨机房、跨城市备份

## show processlit
* "sending to client" 客户端可能要求的数据多,处理逻辑慢,返回一条处理一把.通过设置服务端的net_buffer_length可以缓解,因为该状态是mysql的状态,如果调大该值,能完全保存数据,则认为已经发完了.或者客户端使用mysql_store_result将数据都保存到本地后再执行处理逻辑
* "sending data"  一个查询开始后就会将状态置为此,所以有可能是执行语句在进行锁等待等情况 

* lru策略:分为young/old两段,young区默认占5/8,新插入的page放到该处
  old区策略:
  ```
  innodb_old_blocks_time
  ```
  该参数控制一个old区的page如果多长时间内被用到则移动到young区,默认1s

## 自增主键能保证连续递增吗
* 自增id什么时候分配?如果是insert执行的时候分配,那么t1插一行,t2插两行(t1插入在t2的两条插入之间),此时rollback t2,那么t1的id会变为  id id+1 id+3 id+5这种类型
* 手动指定值插入id,也会造成不连续
* 自增id的分配在唯一键的校验之前,当唯一键冲突时也会造成自增id不连续
```
innodb_autoinc_lock_mode = 1
0.自增锁语句执行完毕之后才释放
1.自增锁申请之后立即释放.但是在insert into .. select ...或者load data ...或者 replace ..select...语句下等语句执行完才释放
2.不论何种情况自增锁申请之后都立即释放 并发性能好但是需要保证binlog_format = row,在statement格式下会造成主备自增id不同
```
insert into .. values (),(),()情况下会一次性将所有自增id分配
insert into ... select ...时会按1个2个4个级数递增的情况分配,如果分配多余就会造成自增id不连续

如果binlog_format = statement,那么insert into ... select ...会将select的表加行锁和gap锁
否则可能造成主备数据不一致



## 参考链接
![mysql](/img/mysql.jpeg)

```
















