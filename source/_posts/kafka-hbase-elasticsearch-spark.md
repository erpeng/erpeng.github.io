---
title: kafka/spark/elasticsearch/hbase笔记
date: 2019-06-09
tags: architecture
---
>有赞日志平台架构(详见后文参考链接)中用到的一些组件,通过查阅文档,进行初步的了解.

## 各组件作用
在有赞日志平台中,kafka是日志中心,所有收集到的日志都会通过不同的topic发送到kafka,利用kafka的高吞吐量和消息中间件的异步解耦特性做一个中间桥梁.spark消费日志并且做一些处理逻辑然后或者写入elasticsearch做查询,或者匹配告警逻辑后做一些监控告警.其中elasticsearch保存索引,hbase保存原始数据,具体做法可参考链接3,这样可以使elasticsearch保存的数据量减小从而匹配filesystem cache,做到高效的查询(查询过程为通过es查询到对应的docid,然后以docid为键去hbase中查询,查到完整的数据).

## 各组件介绍

### kafka
待补充

### hbase

* 源于Google的"Bigtable: A Distributed Storage System for Structured Data".关键特性是可实时的随机读写超大规模的数据集.可通过增加节点来实现线性扩展
* 数据模型
	* **列族**:同一个列族的所有成员具有相同的前缀.并且列族作为表模式定义的一部分必须预先给出.但是列族成员可以动态添加.以一个图片表作为示例,例如有两个列族,contents以及info,contents:image为图片内容,info:format为图片格式,info:geo为图片拍摄的坐标值
	* 所有的调优和存储都是在列族的格式上进行的,所以最好使所有列族成员具有相同的访问模式和大小特征.
	* **单元格**:cell.行列交叉处为一个cell,cell是有版本的,默认为插入时的时间戳.(**默认保存多少个版本?并且查询时返回多少个版本?**)
	* 行是升序排列的,按字节序.行的键值和cell的内容都是二进制字节流,但是列族的前缀必须是可打印字符
	* **区域**:region.每个区域由它所属于的表,它所包含的第一行及其最后一行(左闭右开区间)来表示.
	区域是在集群上分布数据的最小单位.

* 实现

![Hbase](/img/Hbase.png)

* Master:启动一个安装,把区域分配给注册的regionserver,恢复regionserver的故障.
* RegionServer:负责零个或者多个区域的管理以及响应客户端的读写请求.
* Hbase通过HDFS来持久化存储数据
* hbase:meta保存在zookeeper,维护着当前集群上所有区域的列表、状态和位置.区域名作为键,区域名由所属的表名、区域起始行、区域的创建时间及其整体的MD5组成.
* 客户端首先通过zookeeper查找hbase:meta的位置,然后通过区域名获取用户空间区域所在节点及其位置,接着可以直接和regionserver交互.为了较少交互需要缓存hbase:meta信息直到碰到错误之前会一直直接使用缓存
* 写操作:先写commit log,然后是内存中的memstore,然后会被刷入文件系统
* 读操作:读memstore找到就返回,否则依次从新到旧查flush file.会对文件系统的flush file进行压缩合并,并且有一个进程会检测是否超出区域的大小,超出会进行分割.(**类似leveldb**)

#### 延伸阅读 zookeeper

Hbase集群使用了zookeeper,延伸了解一下zookeeper

* 数据模型:
	* 两种znode,一种是暂时的(ephemeral),一种是持久的(persistent).创建znode的客户端断开连接时,短暂znode都会被ZooKeeper服务删除.持久的则不会.一个znode被存储的数据限制在1MB以内
	* 顺序znode.sequential znode,该znode名称之后会附加一个值,为一个单调递增的计数器所添加
	* 可以watch一个znode,则znode被修改时可以通知观察者

* 操作:
	* 集合更新 multi操作可以原子性的执行一批基本操作
	* sync:**Zookeeper允许客户端读到的数据滞后于zookeeper服务的最新状态,因此客户端使用sync操作来获取数据的最新状态**
	* exists,getChildren和getData这些读操作上可以设置观察,这些观察可以被写操作create、delete、setData触发.
* 实现:
	* zookeeper的目标是将znode树的每一个操作都会被复制到集合体重超过半数的机器上
	* 读取操作:任意一个机器可以为读请求提供服务.
	* 写操作:所有的写请求都会被转发给领导者,再由领导者将更新广播给跟随者,半数以上持久化之后领导者才会提交这个更新.类似于两阶段提交
	* 每一个对znode树的更新会赋予一个全局唯一的ID,称为zxid

* zookeeper可以看做一个具有高可用特性的文件系统.但没有文件和目录,而是统一使用节点的概念,称为znode.
* 删除一个znode时需要提供节点路径和版本号.乐观锁,如果版本号不同则不会被删除,但是将版本号设置为-1可以绕过该版本检测机制,不管znode的版本号是什么而直接将其删除.

* 分布式锁的实现:
	* 分布式锁首先需要获得大多数节点的同意才能获得锁
	* 必须有过期时间,以免持有锁之后不能释放导致其他进程无法获取锁
	* 释放锁时必须比较是否是自己申请的锁,即不能将其他进程的锁释放
	* 常用的有redis分布式锁,分别通过对多台redis实例申请锁,并且设置key的过期时间以及value值设置为一个随机值来解决上述问题
* zookeeper分布式锁:
	* 在锁/dl/lock-下create一个sequential ephemeral node, 返回值为/lock-1、/lock-2等等
	* 查询/dl下的所有子节点并且设置一个观察
	* 如果第一步中获取到的znode在/dl中具有最小值则获取到锁,观察中同样如此处理
	* 过期通过zk的ephemeral node实现,zk本身是提供分布式特性


### elasticsearch

待补充

### spark

* Apache Spark是用于大数据处理的集群计算框架.最突出的表现在于它能将作业与作业之间产生的大规模的工作数据集存储在内存中
* RDD(Resilient Distributed Dataset)弹性分布式数据集是所有Spark程序的核心,RDD的创建有三种方法
	* 来自一个内存中的对象集合
	* 使用外部存储器（例如HDFS)中的数据集
	* 对现有RDD进行转换
* 转换和动作:转换时从现有RDD生成新的RDD,而动作则触发对RDD的计算并对计算结果执行某种操作,要么立即返回给用户,要么保存到外部存储器中
* 适合于迭代算法,上次的结果可以保存到内存中供下次使用,并且提供了不同的持久化方法,可以直接保存,可以序列化后保存,也可以保存到磁盘上

spark统计词频
```
val conf = new SparkConf().setAppName("wiki_test") // create a spark config object
val sc = new SparkContext(conf) // Create a spark context
val data = sc.textFile("/path/to/somedir") // Read files from "somedir" into an RDD of (filename, content) pairs.
val tokens = data.flatMap(_.split(" ")) // Split each file into a list of tokens (words).
val wordFreq = tokens.map((_, 1)).reduceByKey(_ + _) // Add a count of one to each token, then sum the counts per word type.
wordFreq.sortBy(s => -s._2).map(x => (x._2, x._1)).top(10) // Get the top 10 words. Swap word and count to sort by count.
```

* **DAG和Spark?** Spark的运行机制

## 参考链接
* https://mp.weixin.qq.com/s?src=3&timestamp=1560006478&ver=1&signature=HHMaY77rbjCfJN64XhOjQA3CUIcxQGO*S6TehE6i1pYL0PZaLATH*b42bk-cJxS7A12UaAJMim8t0F-pPi4uhZLKVEo7Nn47HPEi0YIzk9RglXtTpsx9Xulq78BdX6C*CwXGPWruB4i5JPnOZAIwQeGviW2163HrP5ezsM6XwnM=
* https://tech.youzan.com/you-zan-bai-yi-ji-ri-zhi-xi-tong-jia-gou-she-ji/
* https://mp.weixin.qq.com/s?timestamp=1560004232&src=3&ver=1&signature=0YiMg6yPTGXZErJPIXfzTLKMKA3Znx9cRPhRWILHNrKb-bmE28E4O9SsFvLK7QMDf7VBAhPYeEisfdoPMaaM4PNej8fMuVqwfEvbeMfvTR*k9Phh7FBWK9I3KN3dS2SZGJNnam6ozsxuwNoFoGzryTUGUqe8a13eTecH5lQA*3I=


