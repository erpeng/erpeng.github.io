---
title: leveldb源码解析
date: 2019-06-26
tags: leveldb
---

## Put操作

继续看该段代码:

```
    leveldb::WriteOptions writeOptions;
    for (unsigned int i = 0; i < 2; ++i)
    {
        ostringstream keyStream;
        keyStream << "Key" << 0;
        
        ostringstream valueStream;
        valueStream << "Test data value: " << i;
        
        db->Put(writeOptions, keyStream.str(), valueStream.str());

    }
```
写入两次Key0,第一次value值为"Test data value: 0",第二次value值为"Test data value: 1"
上一篇文章讲述了如何插入log文件,这篇文章讲述如何插入memtable文件

在如下位置打断点,并且看下调用路径:

```
(gdb) b /home/xiaoju/leveldb/db/write_batch.cc:128
(gdb) bt
#0  leveldb::WriteBatchInternal::InsertInto (b=0x7fffffffd550, memtable=0x606640) at db/write_batch.cc:130
#1  0x00007ffff7d98f59 in leveldb::DBImpl::RecoverLogFile (this=0x603440, log_number=3, last_log=true,
    save_manifest=0x7fffffffd93f, edit=0x7fffffffd8a0, max_sequence=0x7fffffffd738) at db/db_impl.cc:426
#2  0x00007ffff7d988ed in leveldb::DBImpl::Recover (this=0x603440, edit=0x7fffffffd8a0, save_manifest=0x7fffffffd93f)
    at db/db_impl.cc:346
#3  0x00007ffff7d9e253 in leveldb::DB::Open (options=..., dbname="./testdb", dbptr=0x7fffffffdd08)
    at db/db_impl.cc:1499
#4  0x0000000000401501 in main (argc=1, argv=0x7fffffffdeb8) at sample.cpp:16
```

该函数代码如下:
```
Status WriteBatchInternal::InsertInto(const WriteBatch* b,
                                      MemTable* memtable) {
  MemTableInserter inserter;
  inserter.sequence_ = WriteBatchInternal::Sequence(b);
  inserter.mem_ = memtable;
  return b->Iterate(&inserter);
}
```
其中WriteBatch的Iterate函数会调用inserter的put/delete方法依次将数据插入.我们看下put方法调用
首先打印出在skiplist中的保存格式:

```
(gdb) p encoded_len
$9 = 32
(gdb) p *buf@32
$10 = "\fKey0\001\003\000\000\000\000\000\000\022Test data value: 0"
```
skiplist中的保存格式如下图:

![leveldb2-1](/img/leveldb2-1.png)

对应gdb解析出的数据,可以看到八进制"\f"表示key总长度为12,"Key0"占4个字节,sequence<<8|type占用8个字节,可以看到type为1,sequence为3.接着是value的长度18及value值"Test data value: 0"

注意comparator比较时首先按key值的字母序比较,如果key值相同,则sequence大的会放到前边.

## Get和Delete操作

```
    leveldb::ReadOptions readOptions;
    std::string getValue;
    db->Get(readOptions,"Key0",&getValue);
    cout << "get Key0: "<<getValue;
```

Get时会根据当前的sequence或者snapshot的sequence查找一个最新的key,即sequence最大的key.然后根据type判断是否删除,如果删除返回不存在该元素.否则返回相应的value

当然,Get如果在memtable中未查找到,会继续去immutable memtable和sstable中查找.immutable的查抄同memtable.sstable的查找后文详述

Delete操作也是一个Put操作,只不过type是删除类型.

默认比较操作见dbformat.cc的如下函数
```
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const 

```

## leveldb中的comparator

笔者在查看该处代码时对comparator类型有颇多疑问,所有单独加此一节介绍一下comparator类型

首先看一下调用链:
```
#0  leveldb::InternalKeyComparator::Compare (this=0x6069e8, akey=..., bkey=...) at db/dbformat.cc:55
#1  0x00007ffff7dad4a9 in leveldb::MemTable::KeyComparator::operator() (this=0x6069e8, aptr=0x606db8 "\fKey0\001\v",
    bptr=0x606df0 "\fKey1\001\f") at db/memtable.cc:38
#2  0x00007ffff7dae882 in leveldb::SkipList<char const*, leveldb::MemTable::KeyComparator>::KeyIsAfterNode (
    this=0x6069e8, key=@0x7fffffffd5e8, n=0x606dd8) at ./db/skiplist.h:258
#3  0x00007ffff7dae468 in leveldb::SkipList<char const*, leveldb::MemTable::KeyComparator>::FindGreaterOrEqual (
    this=0x6069e8, key=@0x7fffffffd5e8, prev=0x7fffffffd520) at ./db/skiplist.h:268
#4  0x00007ffff7dae1c4 in leveldb::SkipList<char const*, leveldb::MemTable::KeyComparator>::Insert (this=0x6069e8,
    key=@0x7fffffffd5e8) at ./db/skiplist.h:341
#5  0x00007ffff7dad71c in leveldb::MemTable::Add (this=0x6069a0, s=12, type=leveldb::kTypeValue, key=..., value=...)
    at db/memtable.cc:105
#6  0x00007ffff7dc08b5 in leveldb::(anonymous namespace)::MemTableInserter::Put (this=0x7fffffffd7a0, key=...,
    value=...) at db/write_batch.cc:118
#7  0x00007ffff7dc055e in leveldb::WriteBatch::Iterate (this=0x7fffffffd930, handler=0x7fffffffd7a0)
    at db/write_batch.cc:59
#8  0x00007ffff7dc09ac in leveldb::WriteBatchInternal::InsertInto (b=0x7fffffffd930, memtable=0x6069a0)
    at db/write_batch.cc:133
#9  0x00007ffff7d9ceef in leveldb::DBImpl::Write (this=0x603440, options=..., my_batch=0x7fffffffd930)
    at db/db_impl.cc:1233
#10 0x00007ffff7d9e08e in leveldb::DB::Put (this=0x603440, opt=..., key=..., value=...) at db/db_impl.cc:1479
#11 0x00007ffff7d9cb77 in leveldb::DBImpl::Put (this=0x603440, o=..., key=..., val=...) at db/db_impl.cc:1187
#12 0x00000000004016d9 in main (argc=1, argv=0x7fffffffdeb8) at sample.cpp:37
```

首先,db_impl.cc中的DB::Open函数如下语句:
```
    impl->mem_ = new MemTable(impl->internal_comparator_);
```
打印出impl->internal_comparator_,
```
(gdb) p impl.internal_comparator_.Name()
$14 = 0x7ffff7dd5c61 "leveldb.InternalKeyComparator"
```
memtable.cc中的构造函数如下:
```
MemTable::MemTable(const InternalKeyComparator& cmp)
    : comparator_(cmp),
      refs_(0),
      table_(comparator_, &arena_) {
}
```
我们看下memtable.h中各变量的定义:
```
class MemTable {
	...
	private:
	...
  	struct KeyComparator {
  		const InternalKeyComparator comparator;
 	    explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) { }
    	int operator()(const char* a, const char* b) const;//重载操作符()
  	};
  	typedef SkipList<const char*, KeyComparator> Table;

 	KeyComparator comparator_;

 	Table table_;

 	...
 }
```
KeyComparator构造函数将comparator赋值为c,并且重载括号操作符.重载后为:
```
int MemTable::KeyComparator::operator()(const char* aptr, const char* bptr)
    const {
  // Internal keys are encoded as length-prefixed strings.
  Slice a = GetLengthPrefixedSlice(aptr);
  Slice b = GetLengthPrefixedSlice(bptr);
  return comparator.Compare(a, b);
}

```
InternalKeyComparator的Compare函数为:
```
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```
首先按key的字母序排,然后按sequence number,sequence大的排在前边.
调用时按如下方法:
```
template<typename Key, class Comparator>
bool SkipList<Key,Comparator>::KeyIsAfterNode(const Key& key, Node* n) const {
  return (n != NULL) && (compare_(n->key, key) < 0);
}
```
重载括号操作符后会调用MemTable::KeyComparator::operator()函数.详见上文第一步bt后的调用链.














