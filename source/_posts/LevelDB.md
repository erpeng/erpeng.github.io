---
title: 如何实现一个数据库
date: 2022-01-07
tags: LevelDB
---


关于数据库大家都不陌生,例如关系型数据库Mysql,其InnoDB存储引擎以一颗B+树来组织数据;Key-Value存储Redis,数据放到内存中,以一个HashTable来组织数据,根据Value的数据类型,又会使用双向链表、SkipList等结构来组织Value数据;分布式数据库Etcd,需要使用一致性协议Raft保证多个节点数据的一致,使用单机存储BoltDB来存储具体的数据.
数据也可以直接存储到文件,例如我们平时代码中打日志就是将数据直接输出到一个文件.常见的数据操作包括增加、删除和查找,从这三种操作来继续思考打日志这个事情.首先我们会以追加的形式写入日志文件,写入磁盘时由于是顺序写入,写入效率不错.查找日志时需要使用Linux系统中的grep命令抓取关键字,grep会顺序遍历一下日志文件,如果日志很大,命令执行起来会很慢.假设不只需要查找，还需要汇总统计,那我们需要继续加管道使用wc,sort或者写一个awk脚本.删除操作呢,首先需要查找,然后执行删除.
总结一下,数据库使用各种数据结构,例如B+树、SkipList是为了平衡增加、删除和查找三种操作,使之都有不错的性能表现.而追加写日志这种操作写入速度很快,但是查找和删除比较慢,有没有办法优化查找和删除呢?
日志格式的追加写方式有一个专门的术语叫log-structured,下边我们看两种数据库,一个是Bitcask,使用的数据结构叫log-structured hash table,一个是LevelDB,使用的数据结构是log-structured merge tree(LSM).这两个数据库都适合写多读少的情况,我们详细看看他们如何去优化读取.

## Bitcask
Bitcask是一个基于log-structured hash table实现的k/v存储,提供了存储和检索k/v的接口.写入时以日志追加的模式顺序写磁盘文件,当文件大小达到一个指定的值时,关闭该文件,然后重新打开一个文件继续写入.此时旧文件变为只读.因此磁盘存储格式如下图所示:
![bitcask-disk](/img/bitcask-disk.png)
磁盘上只有一个活跃文件可以写入,剩余的文件只可以读取.继续观察每一组k/v是如何存储的,如下图:
![bitcask-entry](/img/bitcask-entry.png)
比较通用的一种保存形式,第一个字段crc来校验数据完整性,然后是一个时间戳,key的大小,value的大小,接着是key的值和value的值.
这种方式写入很简单,那么如何加快读取速度呢?Bitcask是通过在内存中建立一个hash table,hash table的格式如下图:
![bitcask-entry](/img/bitcask-keydir.png)
我们假设写入一个键值对k1=>v1.此时hash table的key就是k1,hash table的value中保存了四个字段
* file_id:可以理解为磁盘文件的名称,假设文件名称为000001.data
* value_size:k1对应的v1的长度,为2
* value_pos:k1对应的v1在文件中的位置,假设为7600
* timestamp:k1写入的时间戳
那么我们查找k1时,直接通过hash table查找到需要在000001.data文件中的偏移量7600字节处读取长度为2的数据,即可以读取到v1.熟悉c api的同学,可以直接想到seek函数:
```
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
```
通过一次磁盘io即可以将数据读取.
有的同学可能会继续思考,那么如何删除呢?如果重复写入一个key如何处理呢?删除和重复写入也是以追加写的形式写入磁盘文件,只不过删除时写入的value是一个特殊标识（tombstone）,表明该键已经删除.每次写入都会更新hash table,因此读取时通过hash table读取到的是最新值.
这种数据类型其实还有一个问题需要解决,如果键值对大量重复写入或者大量删除,磁盘文件会膨胀的很大,浪费磁盘空间,例如Redis中的AOF文件.当然解决方法也类似,Bitcask通过merge操作可以将多个文件合并为一个,文件中只保留最新的一个value,删除的key可以直接不保存,AOF文件也是通过一个单独的进程进行重写替换掉原来的文件.
最后一个问题,如果意外宕机或者机器迁移导致关机重启,内存中的hash table如何重建.当然首先可以通过磁盘文件重建,但这样会拖慢启动速度.具体方法大家可以参考链接中的论文,只有6页,留待自己挖掘.
附带提一下kafka,我们知道kafka是一个消息中间件,可以吞吐大量写入,他的消息其实也是保存为日志追加的格式,学习完Bitcask对kafka消息保存格式看着会分外眼熟.

## LevelDB




## 参考链接
* https://riak.com/assets/bitcask-intro.pdf
* 












