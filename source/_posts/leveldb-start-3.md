---
title: leveldb源码解析3之compact
date: 2019-06-28
tags: leveldb
---

## compact时机

首先看如下代码:
```

  bool NeedsCompaction() const {
    Version* v = current_;
    return (v->compaction_score_ >= 1) || (v->file_to_compact_ != NULL);
  }
```
这段代码决定是否需要compact,两个变量决定,compaction_score_和file_to_compact_.我们看下leveldb中如何更新这两个变量:

compaction_score_计算逻辑的调用链为VersionSet::LogAndApply->VersionSet::Finalize,其中Finalize中有如下代码:
```
  v->compaction_level_ = best_level;
  v->compaction_score_ = best_score;
```
compaction_score_按如下方法计算:
* level 0的计算逻辑为level 0文件个数除以4
* 其他level计算逻辑为该level所有文件大小除以该level允许的最大文件大小(level1为10M,level2为100M依此类推)

接着看看file_to_compact_的赋值逻辑,调用链为DBImpl::Get->Version::UpdateStats
其中UpdateStats函数如下
```
bool Version::UpdateStats(const GetStats& stats) {
  FileMetaData* f = stats.seek_file;
  if (f != NULL) {
    f->allowed_seeks--;
    if (f->allowed_seeks <= 0 && file_to_compact_ == NULL) {
      file_to_compact_ = f;
      file_to_compact_level_ = stats.seek_file_level;
      return true;
    }
  }
  return false;
}
```
如果一个文件无效查找(即在该文件中未找到要查找的key)次数过多,则将file_to_compact_赋值为该文件

## compact逻辑

首先分析如上两种compact条件,第二种直接compact指定文件即可.第一种只是确定了level,那么从哪个文件开始压缩呢?

leveldb中使用compact_pointer_这个变量记录每次压缩的文件,下次压缩时从大于之前压缩文件最大值的地方开始继续压缩,类似于一种轮询策略.当然,这个变量会记录到version中并落盘到manifest文件.

待compact的所有文件的选取逻辑如下:

* level0选取文件时会将key重叠的都选取出来
* 从level+1中取出与level层选取文件的key有重叠的文件;并且在level+1层文件不变的情况下,继续从level层选取与level+1层有key重叠的文件,如果有并且level+1层的文件范围不需要继续扩大,则增加该文件到compact的列表中

至此,可以进行leveldb的多路归并排序了.

















