---
title: Mysql 索引
date: 2019-04-22
tags: Mysql
---
## 引言
理解Mysql的索引
参考极客时间**mysql45讲**之第4讲,5讲,9讲,10讲,11讲,18讲

### 不同的索引
||哈希索引|有序数组|树|
|-------|-------|----|
|增  |最快	|   慢	|   快|  
|查 | 最快	|	快	|	快|
|范围查 | 慢	|	快	|	快|

### B+树索引的类别
#### B+树保存数据的层数
磁盘随机读取需要10ms的时间

Mysql每页默认大小16KB,整型占据4字节+9字节(辅助字段）= 13字节,16Kb/13 = 1260,即整型索引每个节点可以保存1260,即1260叉树.三层为1260^3 约等于20亿条数据
增大page可以增加节点个数,减少层数

#### B+树索引类型
* 叶子节点保存数据为聚簇索引
* 叶子节点保存主键为二级索引


```
mysql> create table T(
id int primary key, 
k int not null, 
name varchar(16),
index (k))engine=InnoDB;

```
基于二级索引需要回表


#### B+树的页分裂与合并
分裂影响性能,也影响磁盘使用率（分裂完后每个page只使用50%）
自增主键有如下两点好处:
* 顺序增加,不易造成分裂
* 二级索引会保存主键,自增主键4或8个字节,比较小,省存储空间

#### 覆盖索引
*  优点:加速查询,不需要回表
*  缺点:二级索引需要保存冗余信息,浪费磁盘空间 

#### 最左前缀
建立联合索引的时候如何决定字段顺序
* 如果通过调整顺序,可以少维护一个索引,这是优先考虑的（有(a,b)联合索引后,不需要单独维护（a）这个索引）

* 空间 如果要实现有单独a,单独b,和(a,b)联合的索引,那么看a,b哪个占用空间大,从而决定使用(a),(b,a)还是(b),(a,b).

#### 索引下推 (ICP index condition pushdown)

where的过滤条件如果在联合索引中存在,可以直接过滤,省去回表过滤的过程

#### 普通索引和唯一索引
结论:如果业务能够保证唯一,尽量用普通索引
原因:普通索引可以用到change buffer,change buffer可以减少磁盘的随机读.即假设一个page不在内存中,当更新或者插入时,不需要load page,只需要把更新放到change buffer中,有后台线程会定期merge change buffer 和 旧页.当下次需要读取旧page时,load到内存中后也会直接merge

change buffer减少了磁盘io,更新和插入变为了直接操作内存.但不适用于写入之后就立马读取的业务场景,这样不仅未减少io,还增加了merge的过程

唯一索引不会用到change buffer,需要每次读取page中的数据然后校验唯一性 

#### 字符串字段加索引

增加前缀索引,例如邮箱上边
```
mysql> alter table SUser add index index1(email);
或
mysql> alter table SUser add index index2(email(6));

```
要点:
* 定义好前缀长度,使其区分度足够高,这样既能节省空间,又不用增加太多的查询成本
* 前缀长度短,省空间,但区分度不够,增加查询和回表的次数
* 前缀长度长,查询成本低但是浪费空间
* 另外,前缀索引必须回表查,用不上覆盖索引的优化

计算区分度
```
mysql> select count(distinct email) as L from SUser;
mysql> select 
  count(distinct left(email,4)）as L4,
  count(distinct left(email,5)）as L5,
  count(distinct left(email,6)）as L6,
  count(distinct left(email,7)）as L7,
from SUser;

```

特例:如果不需要范围查询,只需要等值查询,可以使用如下方法:
* 倒序存储.例如身份证号只取前六位区分度不高,则可以使用倒序之后的六位
* 哈希存储.新增加哈希字段,文中为crc32,则查询时首先走crc32字段的索引查,然后精确匹配(防止冲突)



##索引使用注意事项
```
表结构
mysql> CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

```
索引字段做函数操作
```
不会用到id索引
select * from tradelog where id + 1 = 10000
不会用到t_modified索引
select count(*) from tradelog where month(t_modified)=7;
month替换为substr(t_modified,1,2)这样更易理解.即函数会破坏掉索引的有序性,MySQL认为扫描索引不再有效
```
隐式类型转换
```
tradeid为varchar(32)
mysql> select * from tradelog where tradeid=110717;
```
mysql中的转换规则是字符串转数字,于是上述语句相当于
```
mysql> select * from tradelog where  CAST(tradid AS signed int) = 110717;
```
隐式编码类型转换

一个表是utfmb4,一个是uft8,mysql会将utf8中的字段转换为uft8mb4,还是会触发上边的规则


## 参考链接
![mysql](/img/mysql.jpeg)











