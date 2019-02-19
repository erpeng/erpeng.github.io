---
title: Redis中查找大key
date: 2019-02-13 14:11:09
tags: Redis
---

## redis-cli提供的方法
注意以下所有试验基于redis 5.0.3版本

redis-cli 提供一个bigkeys参数，可以扫描redis中的大key

```
  --bigkeys          Sample Redis keys looking for big keys.
```
执行之后输出如下所示:
```
bogon:sqlite didi$ redis-cli --bigkeys
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).
[00.00%] Biggest zset   found so far 'testzset' with 129 members
[00.00%] Biggest hash   found so far 'h2' with 513 fields
[00.00%] Biggest set    found so far 'si1' with 5 members
[00.00%] Biggest hash   found so far 'h4' with 514 fields
[00.00%] Biggest string found so far 'key' with 9 bytes
-------- summary -------
Sampled 9 keys in the keyspace!
Total key length in bytes is 27 (avg len 3.00)
Biggest string found 'key' has 9 bytes
Biggest    set found 'si1' has 5 members
Biggest   hash found 'h4' has 514 fields
Biggest   zset found 'testzset' has 129 members
1 strings with 9 bytes (11.11% of keys, avg size 9.00)
0 lists with 0 items (00.00% of keys, avg size 0.00)
2 sets with 8 members (22.22% of keys, avg size 4.00)
4 hashs with 1541 fields (44.44% of keys, avg size 385.25)
2 zsets with 132 members (22.22% of keys, avg size 66.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
```
原理比较简单,使用scan命令去遍历所有的键，对每个键根据其类型执行"STRLEN","LLEN","SCARD","HLEN","ZCARD"这些命令获取其长度或者元素个数。

该方法有两个缺点:

1.线上使用:虽然scan命令通过游标遍历建空间并且在生产上可以通过对从服务执行该命令,但毕竟是一个线上操作

2.set,zset,list以及hash类型只能获取有多少个元素。但其实元素多的不一定占用空间大

所以有没有一种方法对线上没有影响，并且能直接以topn的形式输出每个键占用的空间大小呢？

我们可以通过读取rdb文件的方式来试验一下，首先看看rdb文件的格式

## rdb文件格式

rdb是一种二进制文件格式,我们首先看看rdb文件的整体结构

![rdb](/img/rdb1.png)

首先是一个魔数,REDIS0009(redis5.0.3版本)。然后是一些附加属性字段,接着是db_num(0-15),然后是db和expire的字典大小(db和过期时间在Redis中是两个独立的hash table),接着是一个个key-value pairs，然后是一个EOF结束标志(0xFF),最后是8字节的checksum

Redis中定义了一些opcode(1字节)，去标记opcode之后保存的是什么类型的数据。如下图所示

![rdb](/img/rdb2.png)

opcode 252标记一个过期时间,248和249分别表示lru或者lfu,接着是value_type,标记值的类型,接着就是一个个key和vlaue.我们看下value_type和redis中数据类型的对应关系


数据类型 | 编码结构 | 值类型
-------|------|-----
OBJ_STRING(0)|OBJ_ENCODING_RAW(0)|RDB_TYPE_STRING(0)
OBJ_LIST(1)|OBJ_ENCODING_QUICKLIST(9) |RDB_TYPE_LIST_QUICKLIST(14)
OBJ_SET(2)|OBJ_ENCODING_INTSET(6)|RDB_TYPE_SET_INTSET(11)
OBJ_SET(2)|OBJ_ENCODING_HT(2)|RDB_TYPE_SET(2)          
OBJ_ZSET(3)|OBJ_ENCODING_ZIPLIST(5)|RDB_TYPE_ZSET_ZIPLIST(12)
OBJ_ZSET(3)|OBJ_ENCODING_SKIPLIST(7)|RDB_TYPE_ZSET_2(5)
OBJ_HASH(4)|OBJ_ENCODING_ZIPLIST(5)|RDB_TYPE_HASH_ZIPLIST(13)
OBJ_HASH(4)|OBJ_ENCODING_HT(2)|RDB_TYPE_HASH(4)

value_type就是值类型这一列，括号中的数字就是保存到rdb文件中时的实际使用数字

知道了rdb的保存格式，我们可以写代码解析rdb文件,通过value_type去获取每个value的大小

## godis-cli-bigkey使用方法
代码地址如下:

https://github.com/erpeng/godis-cli-bigkey

下载之后在将rdb文件拷贝到项目根目录,按如下方式执行

```

bogon:godis-cli-bigkey didi$ go run godis-cli-bigkey.go -h
  -debug
    	open debug mode  //debug模式，输出详细key/value信息
  -topn int
    	output topn keys (default 100)//默认输出top100的大key
  -totallen
    	get total len (key and meta) or only value len (default true)//如果该选项设置为false,只输出rdb文件中value实际占用的大小
																	 //默认为true,输出key、value和所有该key,value保存时使用的元数据总和
exit status 2
```

我们具体执行一下，输出如下:

```
bogon:godis-cli-bigkey didi$ go run godis-cli-bigkey.go
Rdb Version:0009
key:k1,valueSize:9,valueType:0,expireTime:1549533396795,lfu:0,lru:0
key:key,valueSize:9,valueType:0,expireTime:0,lfu:0,lru:0
key:ss1,valueSize:14,valueType:2,expireTime:0,lfu:0,lru:0
key:si1,valueSize:23,valueType:11,expireTime:0,lfu:0,lru:0
key:l1,valueSize:28,valueType:14,expireTime:1549537004535,lfu:0,lru:0
key:h1,valueSize:33,valueType:13,expireTime:0,lfu:0,lru:0
key:z1,valueSize:67,valueType:12,expireTime:0,lfu:0,lru:0
key:testzset,valueSize:1303,valueType:5,expireTime:0,lfu:0,lru:0
key:h3,valueSize:8845,valueType:13,expireTime:0,lfu:0,lru:0
key:h2,valueSize:11680,valueType:4,expireTime:0,lfu:0,lru:0
key:h4,valueSize:11703,valueType:4,expireTime:0,lfu:0,lru:0
```