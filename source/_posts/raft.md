---
title: raft笔记
date: 2022-01-12
tags: raft
---
>raft论文笔记,备忘和参考.读取之前需要读一下链接中的论文
## 服务状态
每个服务都会保存如下状态:
* CurrentTerm:当前的term值.如果接收到请求或者响应的term T > CurrentTerm,则变更CurrentTerm为T,并且变更角色为follower
* VotedFor:投票的候选人ID.如果没有投票或者候选人的candidateId等于votedFor并且候选人的日志比当前服务要新或者一致,则投票给候选人.日志比较规则为:如果最后一条日志的term小于候选人日志的term,或者term相等,但是日志索引小于候选人的日志索引,则候选人日志比当前服务新.如果最后一条日志的term和index与候选人一致,则当前服务和候选人的日志一致.
* log[]:日志条目,每个日志条目包括term和index两个指标,分别表示该条日志被leader接收到时的term以及日志条目在数组中的索引
* CommitIndex:commited日志索引.
* LastApplied:应用到状态机的日志索引.如果lastapplied < commitIndex,则apply lastApplied处的日志.并且增加lastapplied

Leader中独有:
* nextIndex[]:对每一个服务,默认需要发送给服务的下一个log entry的index
* matchIndex[]:对每一个服务,已经被复制到该服务上的最大的log entry的index

![status](/img/raft-status.png)
## 两种rpc请求

* AppendEntry:
![appendEntry](/img/raft-AppendEntries.png)


* RequestVote:
![requestVote](/img/raft-RequestVote.png)

## 三种角色
* Leader、Follower、Candidate,写入请求由Leader统一处理,当Leader发送AppendEntry请求,并且大多数follower都响应之后,此时认为该请求是处于commited状态,Leader将该条日志应用到状态机之后返回
* 所有处于commited的请求,假定此时发生了Leader crash,重新选举的Leader也必然会有该条日志
* 初始状态为Follower,当在election timeout范围之内没有收到Leader的心跳（也为AppendEntry RPC）,则转换为Candidate进行选举
* 转换为Candidate角色之后,首先将CurrentTerm加1,并且给自己投一票,重置定时器,然后给其他服务发送RequestVote RPC请求,请求投票.如果收到大多数成员的投票,则转换为Leader,如果在election timeout之内没有收到大多数投票并且没有新的Leader发送心跳信息,则重新选举,如果收到新的Leader发送的心跳信息,则转换为Follower.


## 五个完备性

![properties](/img/raft-properties.png)

## 成员变更

为了防止成员变更时同时出现两个主,增加了一个中间状态,称之为M.
假设原来有三个节点,A,B,C,增加了两个节点D,E,此时C的配置已经变更,那么A可以被选为Leader,投票人为A,B（旧配置有三个节点,两票已经符合大多数）.E节点也可以被选为Leader,投票人为C,D,E(因为C已经更新为最新配置,新配置有5个节点,3票符合大多数)

增加中间状态M后,不会直接变更为新配置,M状态下新旧配置同时存在,通过提交一个新旧配置同时存在的command,该command发送到新旧所有的服务,需要新的大多数和旧的大多数同时通过才可以commit,然后再创建一个新配置的command,此时需要新服务的大多数同意才能commit.确保不会有一个时刻,新旧配置能够同时生效

还有三点需要注意:
* 新加入节点需要同步log,时间可能会比较久,因此增加了一个步骤,使该服务此时只同步不投票,赶上日志之后再进入正常状态(Learner角色？)
* 有可能Leader不属于新的配置,此时提交新的配置命令时,由一个不属于集群的节点管理集群,并且Leader也不属于大多数
* 下线的服务收不到心跳会变为candidate,持续发起选举.因此如果在election timeout之内收到选举请求时不予理睬

## Log Compaction

为了防止日志太大,占用太多磁盘空间,raft通过将日志打快照并在快照中保存一些元信息,例如last included index和last included term,也就是状态机应用的最后一条log entry的index和term.也会保存最近的配置信息.

为了帮助一些慢follower或者新加入节点同步日志,实现一个InstallSnapshot RPC,专门用来同步快照.


## 客户端交互
写入通过Leader进行,幂等性通过客户端传唯一键,状态机验重.为了读取时不会读取到stale data,每次leader当选后首先提交一个no op log,并且通过给大多数节点发送心跳并收取回复来保证数据的正确性.

## 参考链接

# [raft](https://raft.github.io/raft.pdf)








