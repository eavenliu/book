# 基于Flink的实时统一计算框架

目前开源大数据计算引擎有很多选择，流计算如 Storm、Samza、Flink、Kafka Stream 等，批处理如 Spark、Hive、Pig、Flink 等。而同时支持流处理和批处理的计算引擎，只有两种选择:一个
是 Apache Spark,一个是 Apache Flink。

从技术，生态等各方面的综合考虑，首先，**Spark 的技术理念是基于批来模拟流的计算**。**而Flink则完全相反，它采用的是基于流计算来模拟批计算**。

从技术发展方向看，用批来模拟流有一定的技术局限性，并且这个局限性可能很难突破。而 Flink基于流来模拟批，在技术上有更好的扩展性。从长远来看，阿里决定用 Flink 做一个统一的、通
用的大数据引擎作为未来的选型。
## Flink的发展
Flink 诞生于欧洲的一个大数据研究项目 StratoSphere。该项目是柏林工业大学的一个研究性项 目。早期，Flink 是做 Batch 计算的，但是在 2014 年，StratoSphere 里面的核心成员孵化出 Flink， 同年将 Flink 捐赠 Apache，并在后来成为 Apache 的顶级大数据项目，同时 Flink 计算的主流方 向被定位为 Streaming,即用流式计算来做所有大数据的计算，这就是 Flink 技术诞生的背景。

2014 年 Flink 作为主攻流计算的大数据引擎开始在开源大数据行业内崭露头角。**区别于 Storm、 Spark Streaming 以及其他流式计算引擎的是:它不仅是一个高吞吐、低延迟的计算引擎，同时 还提供很多高级的功能。比如它提供了有状态的计算，支持状态管理，支持强一致性的数据语义以及支持 Event Time,WaterMark 对消息乱序的处理**。

![image](http://note.youdao.com/yws/res/16497/D9429D67E49D418CBE11870AC00E0E2C)
## Flink核心概念
**Flink 最区别于其他流计算引擎的，其实就是状态管理**。

### 什么是状态？
例如开发一套流计算的系统或者任务做数据处理，可能经常要对数据进行统计，如 Sum、Count、Min、Max,这些值是需要存储的。因为要不断更新，这些值或者变量就可以理
解为一种状态。如果数据源是在读取 Kafka、RocketMQ，可能要记录读取到什么位置，并记录
Offset，这些 Offset 变量都是要计算的状态。

**Flink 提供了内置的状态管理**，可以把这些状态存储在 Flink 内部，而不需要把它存储在外部系
统。这样做的好处是：
1. 第一降低了计算引擎对外部系统的依赖以及部署，使运维更加简单；
2. 第二，对性能带来了极大的提升：如果通过外部去访问，如 Redis，HBase，它一定是通过网络及 RPC。
如果通过 Flink 内部去访问，它只通过自身的进程去访问这些变量。
3. 同时 Flink 会定期将这些状态做 Checkpoint 持久化，把 Checkpoint 存储到一个分布式的持久化系统中，比如 HDFS。这样
的话，当 Flink 的任务出现任何故障时，它都会从最近的一次 Checkpoint 将整个流的状态进行恢复，然后继续运行它的流处理。对用户没有任何数据上的影响。

**Flink 是如何做到在 Checkpoint 恢复过程中没有任何数据的丢失和数据的冗余？来保证精准计算的？**

原因是 Flink 利用了一套非常经典的** Chandy-Lamport 算法**，它的核心思想是把这个流计算看成一个流式的拓扑，定期从这个拓扑的头部 Source 点开始插入特殊的 Barriers，从上游开始不断的向下游广播这个 Barriers。每一个节点收到所有的 Barriers,会将 State 做一次 Snapshot，当每个节点都做完 Snapshot 之后，整个拓扑就算完整的做完了一次 Checkpoint。接下来不管出现任何故障，都会从最近的 Checkpoint 进行恢复。

> 分布式快照(Chandy-Lamport算法)：这里不做深入分析，详见：https://yq.aliyun.com/articles/688764

![image](https://note.youdao.com/yws/res/16466/F31D405B7C324F43A7420CB5B62B645C)

Flink 利用这套经典的算法，保证了强一致性的语义。这也是 Flink 与其他无状态流计算引擎的核心区别。

### Flink是如何解决乱序问题
在流计算中，所有消息到来的时间，和它真正发生在源头，在线系统 Log 当中的时间是不一致的。在流处理当中，希望是按消息真正发生在源头的顺序进行处理，不希望是真正到达程序里的时间来处理。Flink 提供了 **EventTime** 和 **WaterMark** 的一些先进技术来解决乱序的问题。使得用户可以有序的处理这个消息。这是 Flink 一个很重要的特点。

## Flink中涉及的关键术语
### 窗口：Window
### EventTime和WaterMark
### Flink 新的分布式架构
阿里巴巴重构了Flink的分布式架构，将Flink的Job调度和资源管理做了一个清晰的分层和解耦。
- 首要好处是 Flink 可以原生的跑在各种不同的开 源资源管理器上。经过这套分布式架构的改进，Flink 可以原生地跑在 Hadoop Yarn 和 Kubernetes 这两个最常见的资源管理系统之上。
- 同时将 Flink 的任务调度从集中式调度改为了分布式调度， 这样 Flink 就可以支持更大规模的集群，以及得到更好的资源隔离。

![image](http://note.youdao.com/yws/res/16491/F77A50E78ABB4F229F3D7BDE5554F00B)

### 增量Checkpoint机制
Flink 提供了有状态的计算和定期的 Checkpoint 机制，如果内部的数据越来越多，不停地做 Checkpoint, Checkpoint 会越来越大，最后可能导致做不出来。

提供了增量的 Checkpoint 后，Flink 会自动地发现哪些数据是增量变化，哪些数据是 被修改了。同时只将这些修改的数据进行持久化，整个系统的性能会非常地平稳。

![image](http://note.youdao.com/yws/res/16495/3CDEB8E62FD5490F9E7FD8AFCDC702B9)
### 基于Credit-based的网络流控机制
### Streaming SQL

## Flink在流批统一处理方面的努力
### 统一的API Stack
Flink 有 2 套基础的 API，一套是 **DataStream**，一套是 **DataSet**。DataStream API 是针对流式处理 的用户提供，DataSet API 是针对批处理用户提供，但是这两套 API 的执行路径是完全不一样的，甚至需要生成不同的 Task 去执行。所以这跟得到统一的 API 是有冲突的，而且这个也是不完善 的，不是最终的解法。在 Runtime 之上首先是要有一个批流统一融合的基础 API 层，我们希望 可以统一 API 层。

因此，我们在新架构中将采用一个 DAG(有限无环图)API，作为一个批流统一的 API 层。对于 这个有限无环图，批计算和流计算不需要泾渭分明的表达出来。只需要让开发者在不同的节点， 不同的边上定义不同的属性，来规划数据是流属性还是批属性。整个拓扑是可以融合批流统一 的语义表达，整个计算无需区分是流计算还是批计算，只需要表达自己的需求。有了这套 API 后，Flink 的 API Stack 将得到统一。

![image](http://note.youdao.com/yws/res/16515/1418DC41324D4BA89ACDEEE462460095)

### 统一的SQL方案
流和批的 SQL， 可以认为流计算有数据源，批计算也有数据源，我们可以将这两种源都模拟成数据表。可以认 为流数据的数据源是一张不断更新的数据表，对于批处理的数据源可以认为是一张相对静止的 表，没有更新的数据表。整个数据处理可以当做 SQL 的一个 Query，最终产生的结果也可以模 拟成一个结果表。

对于流计算而言，它的结果表是一张不断更新的结果表。对于批处理而言，它的结果表是相当 于一次更新完成的结果表。从整个 SQL 语义上表达，流和批是可以统一的。此外，不管是流式 SQL，还是批处理 SQL，都可以用同一个 Query 来表达复用。这样以来流批都可以用同一个 Query 优化或者解析。甚至很多流和批的算子都是可以复用的。

![image](http://note.youdao.com/yws/res/16519/8E3984FDB119467E8A29757EAC6663FE)

## 具体实例


## 本地安装
环境说明：
```
root@ubuntu:~# java -version
openjdk version "1.8.0_222"
OpenJDK Runtime Environment (build 1.8.0_222-8u222-b10-1ubuntu1~18.04.1-b10)
OpenJDK 64-Bit Server VM (build 25.222-b10, mixed mode)
```
ubuntu 18.04

1. 下载Flink

```
root@ubuntu:/mnt/flink# wget http://mirrors.tuna.tsinghua.edu.cn/apache/flink/flink-1.8.1/flink-1.8.1-bin-scala_2.12.tgz
```
2. 解压并启动

```
root@ubuntu:/mnt/flink# tar -xvf flink-1.8.1-bin-scala_2.12.tgz
root@ubuntu:/mnt/flink# cd flink-1.8.1/
root@ubuntu:/mnt/flink/flink-1.8.1# ./bin/start-cluster.sh
Starting cluster.
Starting standalonesession daemon on host ubuntu.
Starting taskexecutor daemon on host ubuntu.
```