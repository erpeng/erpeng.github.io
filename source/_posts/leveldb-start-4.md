---
title: Rocksdb
date: 2020-11-02
tags: 
- leveldb
- rocksdb
---

>主要介绍Rocksdb与Leveldb的不同之处,以及为何如此设计

## 基础概念

* 列簇（Column Family):默认为default,CF将一个数据库实例划分为多个partition.借鉴于Hbase.实现时WAL共用,保证跨CF的原子性,但Memtable和SSTable都是独立的,保证可以独自配置
* Iterator和snapshot:都会提供某个时点的视图,但前台任务或者短暂的扫描适用于Iterator,长时间任务或者后台任务适合于snapshot.Iterator会将每个使用的文件引用次数加1 ,因此直到Iterator释放Compaction才可以删除文件.而snapshot不会阻止删除文件,只会保证不会删除掉快照使用的key
* Memtables:
  * Pluggable MemTables:LevelDB中的memtable是一个skiplist,适用于写入和范围扫描,但并不是所有场景都同时需要良好的写入和范围扫描,此时用skiplist就有点大材小用了.因此RocksDB的Memtable是一个插件式的,可以是skiplist,可以是vector,可以是prefix-hash memtable.vector是无序的,适用于大量写入,dump为sstable时再排序写入L0,但不适用于范围扫描.prefix-hash memtable适用于get,put和scan-within-a-key-prefix.
  * Memtable Pipelining:当Memtable写满之后,会将其赋值给一个immutalbe memtable,然后由后台线程dump到磁盘生成一个sstable.但如果此时有大量的写入,Memtable迅速再次写满,此时就会阻塞写入.因此RocksDB使用一个队列,将immutable memtable放入,依次由后台线程处理,及可以同时存在多个immutable memtable.当使用慢速存储时,能够加大写吞吐量
  * Garbage Collection:memtable dump为一个sstable时,会将重复的写入或者已经删除的key之前的写入都清理掉,这样能够减少存储大小和写放大效应
* Avoiding Stalls:突发大量写入时,如果immutalbe memtable达到了设置的最大值,仍然会阻塞写,因此后台的compaction线程可以保留一小部分专门用来将memtable dump为sstable,防止stalls
* concurrent compaction:L0到L1不能并行.但多个无重叠的compaction可以并行执行.Memtable的flush和compaction可以使用单独的线程各自独立执行
* Thread Pool
  ```
  #include "rocksdb/env.h"
  #include "rocksdb/db.h"

  auto env = rocksdb::Env::Default();
  env->SetBackgroundThreads(2, rocksdb::Env::LOW); //低优先级队列
  env->SetBackgroundThreads(1, rocksdb::Env::HIGH);//高优先级队列
  rocksdb::DB* db;
  rocksdb::Options options;
  options.env = env;
  options.max_background_compactions = 2; //用来 compaction
  options.max_background_flushes = 1;  // 用来flush memtable
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  assert(status.ok());
  ...
  ```

* compaction:LevelDB L0的文件之间有重叠,每个文件都是有序的;L1及之上的层级不仅单个文件内key有序,并且全局有序,文件之间的key不会重叠.CompactionL1及以上时,基本上是Ln的一个文件和Ln+1的多个文件进行compaction.L0时略有不同,需要将L0中的所有重叠文件选出并且compaction.RocksDB提供了不同的compaction方法,例如将Ln的所有文件和Ln+1的一个文件进行compaction.此种情况下Ln(n>=1)并不是全局有序,不同文件之间的key会重叠.

* PlainTable Format:RocksDB通过自定义SSTable的格式,使其适用于SSD硬盘.例如PlainTable Format.大体思路为通过记录每行记录的偏移量作为索引,直接通过随机IO读取该行记录.比SSTable简单不少,不需维护LRUCache,Block Cache,避免了大量的内存拷贝

* Merge Operator:RocksDB支持三种类型的操作,Put,Delete,Merge.当Compaction时如果遇到了merge操作,会回调业务层定义的Merge Operator,Merge能够将一个Put和多个Merge合并为一个操作.有此项功能之后,业务层可以自定义和扩展接口,例如Incr,将一个counter进行加1操作,Append,将一个字符串附加到一个value之后等等
















