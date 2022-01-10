---
title: raft
date: 2022-01-10
tags: raft
---
>raft论文

## 诞生

因为Paxos的难以理解,所以Raft设计之初就会考虑其易理解性











问题1:
* 一个candidate如何知道当前集群中节点的数量,从而能够判断是否收到了大多数节点的投票？
* 一个candidate如果变为了leader,是否需要立即发送心跳到各个节点？
