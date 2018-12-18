---
title: Spark 常用参数设置
date: 2018-12-11
categories: [development]
tags: [ufs,防火墙,DoS,安全]
language: zh
---

记录一些常用的必须要设置的参数

#### num-executors
- 说明：该参数用于设置Spark作业总共要用多少个Executor进程来执行
- 建议：设置的太少，无法充分利用集群资源；设置的太多的话，大部分队列可能无法给予充分的资源。建议 50~100个

#### .executor-cores
- 说明：用于设置每个Executor进程的CPU core数量。
- 建议：如果设置过少（例如设为1），无法发挥单个 work 进程的并发能力，如果设置过大，如果有 HDFS 的 IO 操作，会使单个节点网络压力较大。建议设为 2~4

#### .executor-memory
- 说明：该参数用于设置每个Executor进程的内存。
- 建议：每个Executor进程的内存设置4G~8G较为合适。

#### .driver-memory
- 说明：该参数用于设置Driver进程的内存。
- 建议：如果有大的 collect 操作的话，需要设大一点，保证 Driver 内存能装得下

#### .spark.default.parallelism
- 说明：该参数用于设置每个stage的默认task数量。
- 建议：设置该参数为num-executors * executor-cores的2~3倍较为合适

#### .yarn.executor.directMemoryOverhead / .yarn.driver.directMemoryOverhead
- 说明：控制 Driver 和 Executor 的 Direct memory ，如果执行过程中出现 direct memory oom 异常，可以将这两个值适当调大一些
- 建议：默认为 256m ，建议 512m

##### 补充说明
Direct memory 也叫做 off heap memory， 是在 jvm heap 之外的由操作系统直接管理的本地内存。 也就是说，这部分内存并不会被 jvm 的拉圾回收器管理。 jvm 本身有参数 `MaxDirectMemorySize` 控制 jvm 进程能用的最大的 offheap 内存，超出这个内存就会报出 Direct memory oom 。由于 Spark 本质上也是分布式的 jvm 程序，所以 direct memory 的最大使用量也最终由这个参数控制。
在Spark中，direct memory分为三部分：
- jvmOverhead
 `jvmOverhead` 是 jvm 自身一些lib库，以及线程栈, 内部数据结构占用的堆外内存
- directMemoryOverhead
 `directMemoryOverhead` 是 spark 框架本身进行一些网络 io 操作/文件 io 操作/ execution(shuffle,join等)/storage(cache 数据)使用的堆外内存
- otherOverhead
 `otherOverhead` 是其他的一些框架占用的direct内存，比如 HBase，或者用户代码中使用的堆外内存，默认大小为0，一般可以忽略。

