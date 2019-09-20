# Zookeeper-分布式协调服务

# 简介

详细资料：https://zookeeper.apache.org/doc/r3.5.5/zookeeperProgrammers.html

大数据、分布式协调请使用zookeeper（Zookeeper本质上是一个CP系统），而交易、服务发现使用其他

## 解决什么问题（设计目标）
Zookeeper致力于为分布式应用提供高性能、高可用且具有严格的顺序访问控制能力（主要是写操作）的分布式协调服务，提供了诸如统一命名服务、服务发布/订阅、配置管理和分布式锁等分布式的基础服务。在分布式一致性方面使用了ZAB的一致性协议。

### 1、可以提供一个简单的数据模型（如单点的注册中心）
Zookeeper使得分布式程序可以通过一个共享的、树形结构的名字空间来进行相互协调。具体结构下面详细分析。

==Zookeeper将全量数据存储在内存中，以此来实现提高服务器吞吐、减少延迟的目的。==

### 2、Zookeeper可以构建集群实现高可用
一个Zookeeper集群通常由一组服务器组成，一般3-5台就可以组成一个可用的Zookeeper集群了。

![image](https://note.youdao.com/yws/res/12088/786A743E47574069A9955008B0B544AF)

### 3、实现请求的顺序访问
对于来自客户端的每个更新请求，Zookeeper都会分配一个全局唯一的递增编号，这个编号反映了所有事务操作的先后顺序，应用程序可以通过使用Zookeeper这个特性来实现更高层次的同步原语（synchronization primitives）。

### 4、高性能的访问
由于Zookeeper将全量数据存储在内存中，并直接服务于客户端的所有非事务请求，因此它尤其适用于以读操作为主的应用场景。读和写的效率比大概是10：1。

## Zookeeper自身的设计特性
Zookeeper可以保证如下的分布式一致性特性：
### 顺序一致性
从同一个客户端发起的事务请求，最终会严格地按照其发起的顺序被应用到Zookeeper中。

### 原子性
所有事务请求的处理结果在整个集群中所有机器上的应用情况都是一致的，即要么所有机器都成功的应用了某一个事务，要么都没有应用。

### 单一视图
无论客户端连接集群的哪一个服务器，看到的服务端数据模型都是一致的。

### 可靠性
一旦服务端成功的应用了一个事务，并完成对客户端的响应，那么该事务所引起的服务端状态变更将会一直被保留，除非另一个事务又对其进行了变更。

### 及时性
Zookeeper仅仅保证在一定的时间范围内，客户端最终一定能够从服务器端读取到最新的数据状态。

## Zookeeper里的核心概念

### 会话（Session）
Session指客户端会话，在Zookeeper中具体指客户端和服务器之间的一个TCP连接。当客户端启动，通过第一次TCP三次握手建立连接开始，客户端会话的生命周期就开始了，通过这个连接，客户端可以通过心跳检测和服务器保持有效的会话，也能向服务器发送请求并接受响应，同时还能通过这个连接接收来自服务器的Watch事件通知。

Session的sessionTimeOut值表示一个客户端会话的超时时间，所以即使由于网络原因客户端断开连接，但在sessionTimeOut的有效时间内，客户端可以通过上一次的sessionId重新跟服务器连接。

![image](https://note.youdao.com/yws/res/12757/71D35F4E5C3545DEBD4A6C7B78364A4C)

### ZNode（数据节点和临时节点）
Zookeeper的数据结构是一个树形结构（ZNode Tree），Znode间是由斜杠（/）分隔的路径元素序列。简单来说其数据模型类似于一个文件系统，而Znode之间的层级关系就像文件系统的目录结构一样，且ZNode都可以保存自身的数据内容和属性信息。

存储在命名空间中每个znode的数据以原子方式读取和写入。读取获取与znode关联的所有数据字节，写入替换所有数据。每个节点都有一个访问控制列表（ACL），限制谁可以做什么。

ZooKeeper也有==临时节点==的概念。它的生命周期和客户端会话保持一致，只要创建znode的会话处于活动状态，就会存在这些znode。会话结束时，znode将被删除。当您想要实现[tbd]时，短暂节点很有用。==但是，只有叶子节点才能被定义为临时节点。==

数据模型和分层命名空间如下图：

![image](https://note.youdao.com/yws/res/12127/4701FD43F0E84A37A1866077C290FFAC)

### 版本（Version）
==对于每个Znode，Zookeeper都会为其维护一个Stat结构，Stat中记录该ZNode的三个数据版本，分别是：version（当前ZNode的版本）、cversion（当前ZNode子节点的版本）、aversion（当前ZNode的ACL版本）==。其中包括数据更改，ACL更改和时间戳的版本号，以允许缓存验证和协调更新。每次znode的数据更改时，版本号都会增加。例如，每当客户端检索数据时，它也接收数据的版本。

### 事件监听器（Watcher）
==Watcher是Zookeeper一个很重要的特性（实现分布式协调服务），允许客户端可以在znode上设置Watcher==。当znode特定事件触发时，Zookeeper服务端会将事件通知到对应的客户端。触发监视事件时，客户端会收到一个数据包，指出znode已更改。如果客户端与其中一个ZooKeeper服务器之间的连接中断，则客户端将收到本地通知。这些可以用于[tbd]。

### （权限控制）ACL
==Zookeeper采用（Access Control List）策略来进行权限控制，类似于Unix文件系统的权限控制==。

Zookeeper的ACL，可以从三个维度来理解：一是scheme; 二是user;三是permission，通常表示为scheme:id:permissions,下面从这三个方面分别来介绍：

-==scheme==: scheme对应于采用哪种方案来进行权限管理，zookeeper实现了一个pluggable的ACL方案，可以通过扩展scheme，来扩展ACL的机制,zookeeper-3.4.4内置以下策略：
- world:anyone代表任何人，zookeeper中对所有人有权限的结点就是属于world:anyone的
- auth: 它不需要id, 只要是通过authentication的user都有权限（zookeeper支持通过kerberos来进行authencation, 也支持username/password形式的authentication)
- digest: 它对应的id为username:BASE64(SHA1(password))，它需要先通过username:password形式的authentication
- ip: 它对应的id为客户机的IP地址，设置的时候可以设置一个ip段，比如ip:192.168.1.0/16, 表示匹配前16个bit的IP段
- x509: 使用客户端X500 Principal作为ACL ID标识。ACL表达式是客户端的确切X500主体名称。使用安全端口时，会自动对客户端进行身份验证，并设置x509方案的身份验证信息。

==id==: id与scheme是紧密相关的，具体的情况在上面介绍scheme的过程都已介绍，这里不再赘述。

==permission==：定义了以下五种权限：
- CREATE：创建子节点权限
- READ：获取节点数据和子节点列表权限
- WRITE：更新节点数据的权限
- DELETE：删除子节点的权限
- ADMIN：设置节点ACL的权限

==ACL仅适用于特定的znode，特别是它不适用于子节点的权限，ACL并不是递归的。==

==CREATE和DELETE都是父节点针对子节点操作的。==

定义一个对所有人有操作权限的结点：
```
Id id = new Id("world", "anyone");
ACL acl = new ACL(ZooDefs.Perms.ALL,id);
```


# 核心协议-ZAB协议
Zookeeper没有完全采用Paxos算法，而是使用了一种称为Zookeeper Atomic Broadcast（ZAB，Zookeeper原子广播协议）的协议作为其数据一致性的核心算法。

==ZAB协议是为分布式协调服务Zookeeper专门设计的一种支持崩溃恢复的原子广播协议。== 基于该协议，Zookeeper实现了一种主备模式的系统架构来保持集群中各副本之间的数据一致性。


# 安装
## 安装环境：

> ubuntu18.04、openJDK8

下载稳定版zookeeper，这里用3.5.5版本：

```
root@ubuntu:/mnt/zookeeper# wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/stable/apache-zookeeper-3.5.5-bin.tar.gz

root@ubuntu:/mnt/zookeeper# ls
apache-zookeeper-3.5.5.tar.gz
```
安装过程不再赘述
## 伪集群搭建
zoo.cfg配置：

```
# 服务器与客户端之间交互的基本时间单元（ms）
tickTime=2000
# The number of ticks that the initial 
# 此配置表示允许follower连接并同步到leader的初始化时间，它以tickTime的倍数来表示。当超过设置倍数的tickTime时间，则连接失败。
initLimit=10
# Leader服务器与follower服务器之间信息同步允许的最大时间间隔，如果超过次间隔，默认follower服务器与leader服务器之间断开链接。
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/mnt/data/zookeeper
# 变更默认的对外暴露端口2181
clientPort=3248
# 限制连接到zookeeper服务器客户端的数量
#maxClientCnxns=60
#cluter
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```
## 启动
因为zookeeper是使用java语言编写的，所以可以通过jar包的方式启动。

也可以使用内置的及保本启动，zookeeper下，$ZK_HOME/bin有几个常用的脚本：

| 脚本                | 说明                                                    |
| ------------------- | ------------------------------------------------------- |
| zkCleanup           | 清理zookeeper的历史数据，包括事务日志文件和数据快照文件 |
| zkCli               | 简易的客户端                                            |
| zkEnv               | 设置环境变量                                            |
| zkServer            | 服务器启动、停止、重新启动脚本                          |
| zkServer-initialize |                                                         |
| zkTxnLogToolkit     |                                                         |

服务器启动停止：

```
./zkServer.sh start/stop/restart
```
集群下可能出现异常：

```
org.apache.zookeeper.server.quorum.QuorumPeerConfig$ConfigException: Error processing /mnt/zookeeper/zookeeper/bin/../conf/zoo.cfg
	at org.apache.zookeeper.server.quorum.QuorumPeerConfig.parse(QuorumPeerConfig.java:154)
	at org.apache.zookeeper.server.quorum.QuorumPeerMain.initializeAndRun(QuorumPeerMain.java:113)
	at org.apache.zookeeper.server.quorum.QuorumPeerMain.main(QuorumPeerMain.java:82)
Caused by: java.lang.IllegalArgumentException: myid file is missing
	at org.apache.zookeeper.server.quorum.QuorumPeerConfig.checkValidity(QuorumPeerConfig.java:734)
	at org.apache.zookeeper.server.quorum.QuorumPeerConfig.setupQuorumPeerConfig(QuorumPeerConfig.java:605)
	at org.apache.zookeeper.server.quorum.QuorumPeerConfig.parseProperties(QuorumPeerConfig.java:420)
	at org.apache.zookeeper.server.quorum.QuorumPeerConfig.parse(QuorumPeerConfig.java:150)
	... 2 more
Invalid config, exiting abnormally

```
这是因为配置了：

```
#cluter
server.1=127.0.0.1:2888:3888
server.2=127.0.0.1:2889:3889
server.3=127.0.0.1:2890:3890
```
需要在配置的目录dataDir=/mnt/data/zookeeper下增加myid文件，文件内容就是对应的节点对应的集群id：myid。

这里server.myid=IP:PORT1:PORT2,第一个端口号（port1）是从follower连接到leader机器的端口，第二个端口（port2）是用来进行leader选举时所用的端口

依次启动三个节点，然后分别查询三个zookeeper节点状态：可以通过Mode发现它们之间的角色状态
```
root@ubuntu:/mnt/zookeeper/zookeeper3/bin# ./zkServer.sh  status
ZooKeeper JMX enabled by default
Using config: /mnt/zookeeper/zookeeper3/bin/../conf/zoo.cfg
Client port found: 3348. Client address: localhost.
Mode: follower
root@ubuntu:/mnt/zookeeper/zookeeper3/bin# cd ../../zookeeper1/bin/
root@ubuntu:/mnt/zookeeper/zookeeper1/bin# ./zkServer.sh  status
ZooKeeper JMX enabled by default
Using config: /mnt/zookeeper/zookeeper1/bin/../conf/zoo.cfg
Client port found: 3148. Client address: localhost.
Mode: follower
root@ubuntu:/mnt/zookeeper/zookeeper1/bin# cd ../../zookeeper2/bin/
root@ubuntu:/mnt/zookeeper/zookeeper2/bin# ./zkServer.sh  status
ZooKeeper JMX enabled by default
Using config: /mnt/zookeeper/zookeeper2/bin/../conf/zoo.cfg
Client port found: 3248. Client address: localhost.
Mode: leader

```
## 客户端脚本
我们使用zkCli脚本来参看管理zookeeper节点运行状态：

```
./zkCli.sh -server 127.0.0.1:3248
```
执行创建、查看、更新、删除
```
WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: 127.0.0.1:3248(CONNECTED) 0] create /zk-test helloWorld
Created /zk-test
[zk: 127.0.0.1:3248(CONNECTED) 1] create /zk-test/test 22222
Created /zk-test/test
[zk: 127.0.0.1:3248(CONNECTED) 2] ls
ls [-s] [-w] [-R] path
[zk: 127.0.0.1:3248(CONNECTED) 3] ls /
[zk-test, zookeeper]
[zk: 127.0.0.1:3248(CONNECTED) 4] ls /zookeeper
[config, quota]
[zk: 127.0.0.1:3248(CONNECTED) 5] ls /zk-test
[test]
[zk: 127.0.0.1:3248(CONNECTED) 6] ls /zk-test/test
[]
[zk: 127.0.0.1:3248(CONNECTED) 7] get /zk-test
helloWorld
[zk: 127.0.0.1:3248(CONNECTED) 8] get /zk-test/test
22222
[zk: 127.0.0.1:3248(CONNECTED) 9] set /zk-test/test 33333
[zk: 127.0.0.1:3248(CONNECTED) 10] get /zk-test/test
33333
[zk: 127.0.0.1:3248(CONNECTED) 11] delete /zk-test
Node not empty: /zk-test
[zk: 127.0.0.1:3248(CONNECTED) 12] delete /zk-test/test
[zk: 127.0.0.1:3248(CONNECTED) 13] get /zk-test/test
org.apache.zookeeper.KeeperException$NoNodeException: KeeperErrorCode = NoNode for /zk-test/test
[zk: 127.0.0.1:3248(CONNECTED) 14]
```
删除路径节点时，下级如果存在节点则不能直接级联删除。

## 集群选举
有上面启动情况可知，zk2为集群的Leader，其余两台都为follower，若在此时将zk2停掉，其余两个节点zk1、zk2状态怎么变化呢？

zk1和zk3都有感知zk2离线：

```
2019-07-18 19:22:24,293 [myid:3] - WARN  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):Follower@96] - Exception when following the leader
java.io.EOFException
	at java.io.DataInputStream.readInt(DataInputStream.java:392)
	at org.apache.jute.BinaryInputArchive.readInt(BinaryInputArchive.java:63)
	at org.apache.zookeeper.server.quorum.QuorumPacket.deserialize(QuorumPacket.java:85)
	at org.apache.jute.BinaryInputArchive.readRecord(BinaryInputArchive.java:99)
	at org.apache.zookeeper.server.quorum.Learner.readPacket(Learner.java:158)
	at org.apache.zookeeper.server.quorum.Follower.followLeader(Follower.java:92)
	at org.apache.zookeeper.server.quorum.QuorumPeer.run(QuorumPeer.java:1271)
2019-07-18 19:22:24,296 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):Follower@201] - shutdown called
java.lang.Exception: shutdown Follower
	at org.apache.zookeeper.server.quorum.Follower.shutdown(Follower.java:201)
	at org.apache.zookeeper.server.quorum.QuorumPeer.run(QuorumPeer.java:1275)
2019-07-18 19:22:24,297 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):LearnerZooKeeperServer@165] - Shutting down
2019-07-18 19:22:24,297 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):ZooKeeperServer@558] - shutting down
2019-07-18 19:22:24,299 [myid:3] - WARN  [RecvWorker:2:QuorumCnxManager$RecvWorker@1176] - Connection broken for id 2, my id = 3, error = 
java.io.EOFException
	at java.io.DataInputStream.readInt(DataInputStream.java:392)
	at org.apache.zookeeper.server.quorum.QuorumCnxManager$RecvWorker.run(QuorumCnxManager.java:1161)
2019-07-18 19:22:24,310 [myid:3] - WARN  [RecvWorker:2:QuorumCnxManager$RecvWorker@1179] - Interrupting SendWorker
2019-07-18 19:22:24,311 [myid:3] - WARN  [SendWorker:2:QuorumCnxManager$SendWorker@1092] - Interrupted while waiting for message on queue
```
这里，当两个fellower几乎同时发现现在的老大挂掉了，变更了自己的状态为LOOKING，然后立马开始向外发出选举自己的通知：

```
zk1：New election. My id =  1, proposed zxid=0x10000002b

2019-07-18 19:22:24,333 [myid:1] - INFO  [SyncThread:1:SyncRequestProcessor@169] - SyncRequestProcessor exited!
2019-07-18 19:22:24,334 [myid:1] - WARN  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):QuorumPeer@1318] - PeerState set to LOOKING
2019-07-18 19:22:24,334 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):QuorumPeer@1193] - LOOKING
2019-07-18 19:22:24,339 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):FastLeaderElection@885] - New election. My id =  1, proposed zxid=0x10000002b
2019-07-18 19:22:24,342 [myid:1] - INFO  [WorkerReceiver[myid=1]:FastLeaderElection@679] - Notification: 2 (message format version), 1 (n.leader), 0x10000002b (n.zxid), 0x2 (n.round), LOOKING (n.state), 1 (n.sid), 0x1 (n.peerEPoch), LOOKING (my state)0 (n.config version)
2019-07-18 19:22:24,343 [myid:1] - WARN  [WorkerSender[myid=1]:QuorumCnxManager@677] - Cannot open channel to 2 at election address localhost/127.0.0.1:3889
java.net.ConnectException: Connection refused (Connection refused)
2019-07-18 19:22:24,346 [myid:1] - INFO  [WorkerReceiver[myid=1]:FastLeaderElection@679] - Notification: 2 (message format version), 3 (n.leader), 0x10000002b (n.zxid), 0x2 (n.round), LOOKING (n.state), 3 (n.sid), 0x1 (n.peerEPoch), LOOKING (my state)0 (n.config version)
2019-07-18 19:22:24,348 [myid:1] - INFO  [WorkerReceiver[myid=1]:FastLeaderElection@679] - Notification: 2 (message format version), 3 (n.leader), 0x10000002b (n.zxid), 0x2 (n.round), LOOKING (n.state), 1 (n.sid), 0x1 (n.peerEPoch), LOOKING (my state)0 (n.config version)
2019-07-18 19:22:24,349 [myid:1] - WARN  [WorkerSender[myid=1]:QuorumCnxManager@677] - Cannot open channel to 2 at election address localhost/127.0.0.1:3889
java.net.ConnectException: Connection refused (Connection refused)

2019-07-18 19:22:24,554 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):QuorumPeer@1269] - FOLLOWING
2019-07-18 19:22:24,554 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):ZooKeeperServer@938] - minSessionTimeout set to 4000
2019-07-18 19:22:24,563 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):ZooKeeperServer@947] - maxSessionTimeout set to 40000
2019-07-18 19:22:24,563 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):ZooKeeperServer@166] - Created server with tickTime 2000 minSessionTimeout 4000 maxSessionTimeout 40000 datadir /mnt/data/zookeeper/zk1/version-2 snapdir /mnt/data/zookeeper/zk1/version-2
2019-07-18 19:22:24,564 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):Follower@69] - FOLLOWING - LEADER ELECTION TOOK - 9 MS
2019-07-18 19:22:25,598 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):Learner@391] - Getting a diff from the leader 0x10000002b
2019-07-18 19:22:25,599 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):Learner@546] - Learner received NEWLEADER message
2019-07-18 19:22:25,611 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):Learner@529] - Learner received UPTODATE message
2019-07-18 19:22:25,611 [myid:1] - INFO  [QuorumPeer[myid=1](plain=/0:0:0:0:0:0:0:0:3148)(secure=disabled):CommitProcessor@256] - Configuring CommitProcessor with 1 worker threads.
2019-07-18 19:22:38,909 [myid:1] - INFO  [NIOWorkerThread-2:NIOServerCnxn@518] - Processing srvr command from /127.0.0.1:51368
```
有上面的日志可以看出zk1自己向外发布消息开始进行选举，发出一个投票，然后收到了一条投票选举消息：
Notification: 2 (message format version), 1 (n.leader), 0x10000002b (n.zxid), 0x2 (n.round), LOOKING (n.state), 1 (n.sid), 0x1 (n.peerEPoch), LOOKING (my state)0 (n.config version)

然后继续向其他机器发送消息，由于zk2已经挂掉，所以会抛出连接异常

```
zk3：New election. My id =  3, proposed zxid=0x10000002b

2019-07-18 19:22:24,337 [myid:3] - INFO  [SyncThread:3:SyncRequestProcessor@169] - SyncRequestProcessor exited!
2019-07-18 19:22:24,337 [myid:3] - WARN  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):QuorumPeer@1318] - PeerState set to LOOKING
2019-07-18 19:22:24,337 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):QuorumPeer@1193] - LOOKING
2019-07-18 19:22:24,340 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):FastLeaderElection@885] - New election. My id =  3, proposed zxid=0x10000002b
2019-07-18 19:22:24,345 [myid:3] - INFO  [WorkerReceiver[myid=3]:FastLeaderElection@679] - Notification: 2 (message format version), 1 (n.leader), 0x10000002b (n.zxid), 0x2 (n.round), LOOKING (n.state), 1 (n.sid), 0x1 (n.peerEPoch), LOOKING (my state)0 (n.config version)
2019-07-18 19:22:24,350 [myid:3] - WARN  [WorkerSender[myid=3]:QuorumCnxManager@677] - Cannot open channel to 2 at election address localhost/127.0.0.1:3889
java.net.ConnectException: Connection refused (Connection refused)
2019-07-18 19:22:24,352 [myid:3] - INFO  [WorkerReceiver[myid=3]:FastLeaderElection@679] - Notification: 2 (message format version), 3 (n.leader), 0x10000002b (n.zxid), 0x2 (n.round), LOOKING (n.state), 1 (n.sid), 0x1 (n.peerEPoch), LOOKING (my state)0 (n.config version)
2019-07-18 19:22:24,353 [myid:3] - INFO  [WorkerReceiver[myid=3]:FastLeaderElection@679] - Notification: 2 (message format version), 3 (n.leader), 0x10000002b (n.zxid), 0x2 (n.round), LOOKING (n.state), 3 (n.sid), 0x1 (n.peerEPoch), LOOKING (my state)0 (n.config version)
2019-07-18 19:22:24,555 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):QuorumPeer@1281] - LEADING
2019-07-18 19:22:24,572 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):Leader@66] - TCP NoDelay set to: true
2019-07-18 19:22:24,573 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):Leader@86] - zookeeper.leader.maxConcurrentSnapshots = 10
2019-07-18 19:22:24,573 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):Leader@88] - zookeeper.leader.maxConcurrentSnapshotTimeout = 5
2019-07-18 19:22:24,575 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):ZooKeeperServer@938] - minSessionTimeout set to 4000
2019-07-18 19:22:24,575 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):ZooKeeperServer@947] - maxSessionTimeout set to 40000
2019-07-18 19:22:24,577 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):ZooKeeperServer@166] - Created server with tickTime 2000 minSessionTimeout 4000 maxSessionTimeout 40000 datadir /mnt/data/zookeeper/zk3/version-2 snapdir /mnt/data/zookeeper/zk3/version-2
2019-07-18 19:22:24,578 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):Leader@464] - LEADING - LEADER ELECTION TOOK - 23 MS
2019-07-18 19:22:24,586 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):FileTxnSnapLog@372] - Snapshotting: 0x10000002b to /mnt/data/zookeeper/zk3/version-2/snapshot.10000002b
2019-07-18 19:22:25,586 [myid:3] - INFO  [LearnerHandler-/127.0.0.1:60684:LearnerHandler@406] - Follower sid: 1 : info : localhost:2888:3888:participant
2019-07-18 19:22:25,594 [myid:3] - INFO  [LearnerHandler-/127.0.0.1:60684:ZKDatabase@295] - On disk txn sync enabled with snapshotSizeFactor 0.33
2019-07-18 19:22:25,595 [myid:3] - INFO  [LearnerHandler-/127.0.0.1:60684:LearnerHandler@708] - Synchronizing with Follower sid: 1 maxCommittedLog=0x10000002b minCommittedLog=0x100000001 lastProcessedZxid=0x10000002b peerLastZxid=0x10000002b
2019-07-18 19:22:25,595 [myid:3] - INFO  [LearnerHandler-/127.0.0.1:60684:LearnerHandler@752] - Sending DIFF zxid=0x10000002b for peer sid: 1
2019-07-18 19:22:25,600 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):Leader@1296] - Have quorum of supporters, sids: [ [1, 3],[1, 3] ]; starting up and setting last processed zxid: 0x200000000
2019-07-18 19:22:25,602 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):CommitProcessor@256] - Configuring CommitProcessor with 1 worker threads.
2019-07-18 19:22:25,610 [myid:3] - INFO  [QuorumPeer[myid=3](plain=/0:0:0:0:0:0:0:0:3348)(secure=disabled):ContainerManager@64] - Using checkIntervalMs=60000 maxPerMinute=10000
2019-07-18 19:22:44,118 [myid:3] - INFO  [NIOWorkerThread-2:NIOServerCnxn@518] - Processing srvr command from /127.0.0.1:56320

```
zk3的逻辑跟zk1的逻辑基本一致，但最终zk3成为了新的leader，具体原因我们先分析zookeeper的选举机制。

### 选举机制 

#### 服务运行期的Leader选举
==集群启动的时候流程基本一致，第一个节点启动了无法进行选举，当第二台机器启动了就进入选举流程。==

当集群运行时，有新的节点加入，或者是非Leader节点移除（至少留下一台构成leader-follower关系），都不会影响集群的正常对外服务，但是当集群中Leader节点挂掉之后，会出现以下流程：
1. 变更节点状态。当Leader节点挂掉，余下的follower节点捕捉到leader丢失，会将自己的节点置为LOOKING，然后进入Leader选举流程。
2. 每个节点Server都会发出一个投票。首先生成对应的投票信息（myid,ZXID），ZXID是一个事务ID，在初始的时候所有的服务器都是一致的，但在运行期间可能会有不同，而第一轮投票所有的服务器都会投给自己，所以服务器投票信息为（自身myid，自身ZXID）。
3. 接收集群中各个服务器的投票。每个服务器都会接收来自其它服务器的投票，每个服务器在接收到投票信息后，会先判断投票信息的有效性：是否是本轮投票、是否来自状态LOOKING的服务器。
4. 处理投票。针对每一个投票信息，服务器都需要将别人的投票和自己的投票PK，具体规则有：==1、优先检查ZXID，ZXID较大的服务器优先成为Leader；2、如果ZXID一致，那么比对myid，myid较大的优先成为Leader。== 如上面的选举，zk1的投票信息为(1,0x10000002b),zk3的投票信息为(3,0x10000002b)，当zk1收到zk2的投票后跟自己的投票比对，因为ZXID一致，所以比较myid，因为zk3的myid较大，所以优先成为Leader，因此zk1重新发送一个(3,0x10000002b)的投票。zk3收到第一次zk1发送的投票后，跟自己的投票比较，发现自己投票更优先，所以不需更改投票。
5. 统计投票。每一次投票后，服务器都会统计所有投票，判断集群中是否已经有超过一半（包括等于）的服务器接受了同一个投票信息。对于上面的zk1和zk3，都接受(3,0x10000002b)这个投票信息，并统计出有2台服务器接受了这个投票，数量过半，即认为投票结束，已经选出了对应的Leader。
6. 改变服务器状态。根据投票统计结果，每个服务器更新自己的状态：如果是Follower，那么就变更状态为FOLLOWING；如果是Leader，那么就变更为LEADING。

### 选举算法分析
使用TCP版本的FastLeaderElection算法。（之前还有纯UDP实现的LeaderElection、UDP版本的FastLeaderElection算法（非授权模式），但都已经被废弃了）

#### 术语解释
##### SID：服务器ID
SID是用来唯一标识集群中服务器的ID，是一个数字，在集群中不能重复，与集群配置中的myid保持一致。

##### ZXID：事务ID
ZXID是一个事务ID，用来唯一标识一次服务器状态的变更。在某一时刻，集群中每一台机器的ZXID不一定都一致，这与Zookeeper服务器与客户端的“更新请求”处理逻辑有关。

##### Vote：投票
Leader选举的投票术语

##### Quorum：过半机器数
可以理解为一个量词，假设集群中机器数为n，那么quorum=n/2+1。

#### 算法分析

##### Leader选举触发
集群中一台服务器进入Leader选举流程一般有两个触发点：
- 服务器初始化启动的时候
- 服务器运行期间无法与运行中的Leader保持连接

那么服务器进入Leader流程的时候也会碰到两种情况：
- 被告知集群中已经存在一个Leader了。通过被告知的Leader信息，与Leader建立连接，并进行状态同步。
- 集群中确实不存在Leader了，开始进行Leader选举投票。

##### 第一次投票
当集群进入Leader选举状态，每一台服务器先都变更自身的状态为“==LOOKING==”，译为正在寻找Leader，然后开始组装投票信息(SID,ZXID)即为上面分析的(myid,ZXID)。这个时候一般第一次投票信息是自身服务器，投票信息即为：(服务自身SID,服务器自身ZXID)。

举例说明，假设一个集群由五台服务器组成，SID分别为：1,2,3,4,5，ZXID分别为：9,9,9,8,8，其中SID=2为目前运行中集群的Leader。在某个时刻发生网络分区，SID=1、SID=2两台服务器无法连接脱离集群，剩余的3/4/5服务器开始进入选举模式。他们先将自身的状态变更为“LOOKING”，然后由于集群状态不明朗所以都构造选举自己的投票信息：(3,9)、(4,8)、(5,8)，向集群他的服务器发送。

##### 变更投票
服务器在发送了自己的投票信息后，也会同步的接受来自其它服务器的投票信息，服务器在接收了来自其它服务器的投票信息后，就要开始评估是否需要变更自己的投票信息。

收到投票消息示例：

```
Notification: 2 (message format version), 1 (n.leader), 0x10000002b (n.zxid), 0x2 (n.round), LOOKING (n.state), 1 (n.sid), 0x1 (n.peerEPoch), LOOKING (my state),0 (n.config version)

//通知:2(投票信息轮次)，1（被推荐的Leader的SID），0x10000002b（被推荐的Leader的事务ID），0x2 (n.round)，LOOKING（选举的服务器状态）, 0x1 (被推荐的Leader的EPoch), LOOKING (发起投票服务器的状态),0 (n.config version)
```
判断是否变更投票依据有：是一个接收投票(vote_sid,vote_zxid)和自己投票(self_sid,self_zxid)的对比过程
1. 如果vote_zxid > self_zxid，认可当前接收到的投票，然后变更自己投票为接收的投票发出去；
2. 如果vote_zxid < self_zxid，坚持自己的投票，不做变更；
3. 如果vote_zxid = self_zxid 且 vote_sid > self_sid，那么就认可当前接收到的投票，然后变更自己投票为接收的投票发出去；
4. 如果vote_zxid = self_zxid 且 vote_sid < self_sid，坚持自己的投票，不做变更；

==总结就是优先比对ZXID（更大的ZXID代表其服务器数据状态是最新的，有助于维护数据的一致性），其次比较SID。==

如图：

![image](https://note.youdao.com/yws/res/11875/EC347E194B354A3DBCE3C1BC14D448F0)

##### 确定Leader
经过服务器第一次投票，然后接受其他服务的投票视情况变更投票，服务器会统计当前的投票情况，如果超过半数的服务器接受到相同的投票信息，那么对应的SID即成为新的Leader。

#### 选举算法实现细节
##### 服务器状态
Zookeeper定义了四种服务器状态：（org.apache.zookeeper.server.quorum.QuorumPeer.ServerState）
- LOOKING：选举Leader状态。
- FOLLOWING：跟随者状态，当前服务器角色为Follower。
- LEADING：领导者状态，当前服务器角色为Leader。
- OBSERVING：观察者状态，当前服务器角色为Observer。

##### 投票消息结构
org.apache.zookeeper.server.quorum.Vote
```
public class Vote {
    private final int version;
    //被推举的Leader的SID值
    private final long id;
    //被推举的Leader的事务ID
    private final long zxid;
    //逻辑时钟（即选举周期）,该值是一个自增整数，每开始新一轮投票都会对其加1
    private final long electionEpoch;
    //被推举的Leader的逻辑轮次（即选举周期）
    private final long peerEpoch;
    //当前服务器的状态
    private final ServerState state;
    ...
}
```


# Java原生客户端

## 建立会话
下面有一个简单的客户端与上面的zookeeper集群建立会话例子：实现zookeeper事件监听接口，新建会话->线程阻塞等待服务端传来会话成功建立事件->线程恢复

```
public class ZookeeperNativeClientImpl implements Watcher {
    private static CountDownLatch countDownLatch = new CountDownLatch(1);
    @Override
    public void process(WatchedEvent watchedEvent) {
        System.out.println("Receive the Eavent Masage:" + watchedEvent.toString());
        if(Event.KeeperState.SyncConnected == watchedEvent.getState()){
            //监控到连接事件返回，倒计时结束，线程继续
            System.out.println("zookeeper确认连接："+ watchedEvent.getPath());
            countDownLatch.countDown();
        }
    }
    public static void main(String[] args) {
        String connectString = "192.168.100.236:3148,192.168.100.236:3248,192.168.100.236:3348";
        int sessionTimeout = 5000;
        try {
            //创建一个会话
            try (ZooKeeper zooKeeper = new ZooKeeper(connectString, sessionTimeout, new ZookeeperNativeClientImpl());) {
                //zookeeper客户端和服务端（TCP连接）会话的建立是异步的，上面的构造方法会在客户端初始化完后立即返回了，这里其实还没有真正建立好可用的会话
                // ，所以这里会输出sessionId：0，并且连接状态处于state:CONNECTING
                System.out.println("sessionId:"+ zooKeeper.getSessionId());
                System.out.println("state:"+ zooKeeper.getState());
                //进程倒计时阻塞，等待会话建立完毕，zookeeper服务端发送一个连接事件返回这
                countDownLatch.await();
                //真正连接后zookeeper推送到客户端事件，这个时候真正连接，返回了正在的now sessionId:72059037908533255，now state:CONNECTED
                System.out.println("now sessionId:"+ zooKeeper.getSessionId());
                System.out.println("now state:"+ zooKeeper.getState());
                // 使用上一次的SessionId创建新的会话
                //ZooKeeper zooKeeper1 = new ZooKeeper(connectString, sessionTimeout, new ZookeeperNativeClientImpl(),
                 //       zooKeeper.getSessionId(), zooKeeper.getSessionPasswd());
               //System.out.println("state1:"+ zooKeeper1.getState());
            }
        }catch(Exception e){
            e.printStackTrace();
        }
    }
}
```
正常输出：

```
...
sessionId:0
state:CONNECTING
...
Receive the Eavent Masage:WatchedEvent state:SyncConnected type:None path:null
zookeeper确认连接：null
now sessionId:144121779447463936
now state:CONNECTED
...
```
当我们关闭集群中两个节点，只留下一个节点(关闭3148，3348服务，留下3248)，启动客户端会出现：

```
15:25:56.475 [main] INFO org.apache.zookeeper.ZooKeeper - Initiating client connection, connectString=192.168.100.236:3148,192.168.100.236:3248,192.168.100.236:3348 sessionTimeout=5000 watcher=com.eavenliu.zookeeper.nativeclient.service.impl.ZookeeperNativeClientImpl@4dcbadb4
15:25:56.502 [main] INFO org.apache.zookeeper.common.X509Util - Setting -D jdk.tls.rejectClientInitiatedRenegotiation=true to disable client-initiated TLS renegotiation
15:25:57.166 [main] INFO org.apache.zookeeper.ClientCnxnSocket - jute.maxbuffer value is 4194304 Bytes
15:25:57.174 [main] INFO org.apache.zookeeper.ClientCnxn - zookeeper.request.timeout value is 0. feature enabled=
sessionId:0
state:CONNECTING
...
15:26:15.295 [main-SendThread(192.168.100.236:3148)] INFO org.apache.zookeeper.ClientCnxn - Opening socket connection to server 192.168.100.236/192.168.100.236:3148. Will not attempt to authenticate using SASL (unknown error)
15:26:16.307 [main-SendThread(192.168.100.236:3148)] INFO org.apache.zookeeper.ClientCnxn - Socket error occurred: 192.168.100.236/192.168.100.236:3148: Connection refused: no further information
15:26:16.365 [main-SendThread(192.168.100.236:3148)] DEBUG org.apache.zookeeper.ClientCnxnSocketNIO - Ignoring exception during shutdown input
java.nio.channels.ClosedChannelException: null
...
15:26:34.497 [main-SendThread(192.168.100.236:3348)] INFO org.apache.zookeeper.ClientCnxn - Opening socket connection to server 192.168.100.236/192.168.100.236:3348. Will not attempt to authenticate using SASL (unknown error)
15:26:35.502 [main-SendThread(192.168.100.236:3348)] INFO org.apache.zookeeper.ClientCnxn - Socket error occurred: 192.168.100.236/192.168.100.236:3348: Connection refused: no further information
15:26:35.503 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxnSocketNIO - Ignoring exception during shutdown input
java.nio.channels.ClosedChannelException: null
...
15:26:53.627 [main-SendThread(192.168.100.236:3248)] INFO org.apache.zookeeper.ClientCnxn - Opening socket connection to server 192.168.100.236/192.168.100.236:3248. Will not attempt to authenticate using SASL (unknown error)
15:26:53.630 [main-SendThread(192.168.100.236:3248)] INFO org.apache.zookeeper.ClientCnxn - Socket connection established, initiating session, client: /192.168.100.94:63988, server: 192.168.100.236/192.168.100.236:3248
15:26:53.641 [main-SendThread(192.168.100.236:3248)] DEBUG org.apache.zookeeper.ClientCnxn - Session establishment request sent on 192.168.100.236/192.168.100.236:3248
15:26:53.650 [main-SendThread(192.168.100.236:3248)] INFO org.apache.zookeeper.ClientCnxn - Unable to read additional data from server sessionid 0x0, likely server has closed socket, closing socket connection and attempting reconnect
...
15:27:13.398 [main-SendThread(192.168.100.236:3148)] INFO org.apache.zookeeper.ClientCnxn - Opening socket connection to server 192.168.100.236/192.168.100.236:3148. Will not attempt to authenticate using SASL (unknown error)
15:27:14.402 [main-SendThread(192.168.100.236:3148)] INFO org.apache.zookeeper.ClientCnxn - Socket error occurred: 192.168.100.236/192.168.100.236:3148: Connection refused: no further information
15:27:14.402 [main-SendThread(192.168.100.236:3148)] DEBUG org.apache.zookeeper.ClientCnxnSocketNIO - Ignoring exception during shutdown input
java.nio.channels.ClosedChannelException: null
...一直循环
```
客户端会一直循环尝试与集群中的三个节点建立会话，两个已经关闭的节点是直接拒绝连不上，仅存的节点不对外提供服务，所以客户端一直尝试。

当我们启动3148节点，集群状态再次建立完毕，客户端下一次循环到正确节点的时候会话又可以成功建立。
## 创建节点
Zookeeper实例实际提供了同步和异步以及是否TTL节点接口来创建节点：

```
public String create(final String path, byte data[], List<ACL> acl,
            CreateMode createMode)
    {}
public void create(final String path, byte data[], List<ACL> acl,
            CreateMode createMode, StringCallback cb, Object ctx)
    {}
public String create(final String path, byte data[], List<ACL> acl,
            CreateMode createMode, Stat stat, long ttl)
            throws KeeperException, InterruptedException 
    {}
public void create(final String path, byte data[], List<ACL> acl,
            CreateMode createMode, Create2Callback cb, Object ctx, long ttl)
    {}
```

| 参数名     | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| path       | 需要创建的数据节点的节点路径，如/test/chain                  |
| data[]     | 字节数组，节点创建后的初始数据                               |
| acl        | 节点的ACL权限策略                                            |
| createMode | 节点类型，枚举：PERSISTENT-持久化节点、PERSISTENT_SEQUENTIAL-持久顺序节点、EPHEMERAL-短暂节点、EPHEMERAL_SEQUENTIAL-短暂顺序节点、CONTAINER-容器节点、PERSISTENT_WITH_TTL-持久TTL节点、PERSISTENT_SEQUENTIAL_WITH_TTL-持久顺序TTL节点 |
| cb         | 注册一个异步回调函数。实现AsyncCallback接口方法的逻辑实现，服务端创建完节点后客户端自动调用这个方法。 |
| ctx        | 传递一个对象，可以在回调方法执行的时候使用，一般是放一个上下文信息 |
| ttl        | 为znode设置TTL（Time To Life）（以毫秒为单位）如果未在TTL定义时间内修改znode并且没有子节点，则它将成为服务器在将来的某个时刻删除的候选者。 |

需要注意几点：
- path节点不支持递归创建，必须先有父节点再有子节点；
- 节点数据只支持字节数据，可以自行使用序列化工具序列化复杂对象

### Stat结构
ZooKeeper中每个znode的Stat结构由以下字段组成：
- czxid 导致创建此znode的更改的zxid。
- mzxid 最后修改此znode的更改的zxid。
- pzxid 最后修改此znode的子项的更改的zxid。
- ctime 创建此znode时从纪元开始的时间（以毫秒为单位）。
- mtime 上次修改此znode时的时间（以毫秒为单位）。
- version 对此znode数据的更改次数。
- cversion 此znode的子项的更改数。
- aversion 对此znode的ACL的更改次数。
- ephemeralOwner 如果znode是一个临时节点，则此znode的所有者的会话ID。如果它不是短暂的节点，则它将为零。
- dataLength 此znode的数据字段的长度。
- numChildren 此znode的子节点数。

### 代码示例
1、创建一般持久化节点：

```
//创建一般持久节点
...
String data = "createData";
List<ACL> aclList = new ArrayList<>(Collections.singletonList(new ACL(ZooDefs.Perms.ALL, new Id("world", "anyone"))));
String path = zooKeeper.create("/createP",data.getBytes(), aclList, CreateMode.PERSISTENT);
System.out.println("create path back:"+ path);

//创建一般持久顺序节点
String data1 = "createData";
String path1 = zooKeeper.create("/createP_S",data1.getBytes(), aclList, CreateMode.PERSISTENT_SEQUENTIAL);
System.out.println("create path1 back:"+ path1);

//创建一般持久顺序TTL节点
String data2 = "createData";
String path2 = zooKeeper.create("/createPST",data2.getBytes(), aclList, CreateMode.PERSISTENT_SEQUENTIAL_WITH_TTL, null,10000);
System.out.println("create path2 back:"+ path2);
...
```
输出：
```
create path back:/createP
create path1 back:/createP_S0000000012
org.apache.zookeeper.KeeperException$UnimplementedException: KeeperErrorCode = Unimplemented for /createPST
```
这里可以看到上面两个节点创建成功，但TTL节点创建失败，这是因为TTL节点默认情况下它们被禁用，必须通过System属性启用TTL节点。

服务器：
```
[zk: 127.0.0.1:3148(CONNECTED) 18] get /createP
createData
[zk: 127.0.0.1:3148(CONNECTED) 19] get /createP_S0000000012
createData

```
2、创建临时节点：

```
//创建一般临时节点
String data = "createData";
String path = zooKeeper.create("/createE",data.getBytes(), aclList, CreateMode.EPHEMERAL);
System.out.println("create path back:"+ path);

//创建一般临时顺序节点
String data1 = "createData";
String path1 = zooKeeper.create("/createE_S",data1.getBytes(), aclList, CreateMode.EPHEMERAL_SEQUENTIAL);
```
输出：

```
create path back:/createE
create path1 back:/createE_S0000000014
...
11:27:15.620 [main] DEBUG org.apache.zookeeper.ZooKeeper - Closing session: 0x3000006a0b50004
11:27:15.621 [main] DEBUG org.apache.zookeeper.ClientCnxn - Closing client for session: 0x3000006a0b50004
Receive the Eavent Masage:WatchedEvent state:Disconnected type:None path:null
11:27:15.726 [main] DEBUG org.apache.zookeeper.ClientCnxn - Disconnecting client for session: 0x3000006a0b50004
Receive the Eavent Masage:WatchedEvent state:Closed type:None path:null
11:27:15.836 [main] INFO org.apache.zookeeper.ZooKeeper - Session: 0x3000006a0b50004 closed
```
服务器：

```
[zk: 127.0.0.1:3148(CONNECTED) 26] ls /
[createE, createE_S0000000014, createP, createP_S0000000012, zk-test, zookeeper]
[zk: 127.0.0.1:3148(CONNECTED) 27] ls /
[createP, createP_S0000000012, zk-test, zookeeper]
```
可以看出，当客户端连接断开之后，服务器生成的临时节点立马被删。

3、异步创建节点：

```
 //创建一般持久异步节点
String dataS = "createData";
zooKeeper.create("/createAsync",dataS.getBytes(), aclList, CreateMode.PERSISTENT, new TestCreate2Callback(),"test context");

public class TestCreate2Callback implements AsyncCallback.Create2Callback {
    @Override
    public void processResult(int rc, String path, Object ctx, String name, Stat stat) {
        System.out.println("Create Node Result:[rc:"+rc+",path:"+path+",ctx:"+ctx.toString()+",name:"+name+",stat:"+stat.toString()+"]");
    }
}
```
输出：（rc：resultCode，创建是否成功的返回码，0=success）

```
Create Node Result:[rc:0,path:/createAsync,ctx:test context,name:/createAsync,stat:17179869249,17179869249,1563853808023,1563853808023,0,0,0,0,10,0,17179869249]
```
根据不同的需要实现不同的异步接口：

![image](https://note.youdao.com/yws/res/13024/0091F26CAA8C44BF960AA781C3E54B8C)

## 删除节点
删除节点有两个接口：同步删除，异步删除

```
public void delete(final String path, int version)
        throws InterruptedException, KeeperException
    {}
public void delete(final String path, int version, VoidCallback cb,
            Object ctx)
    {}
```
参数说明：

| 参数名  | 说明                     |
| ------- | ------------------------ |
| path    | 指定的数据节点路径       |
| version | 节点的数据版本           |
| cb      | 异步回调函数VoidCallback |
| ctx     | 传递上下文信息           |

## 获取数据
### getChildren获取节点的子节点
所有方法：

![image](https://note.youdao.com/yws/res/13050/75A2BCD536334EF8949A60729B068292)

参数说明：
| 参数名  | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| path    | 指定的数据节点路径                                           |
| watcher | 注册一个watcher。一旦在本次子节点获取之后，子节点列表发生变更的话，那么就会像客户端发送通知，允许为null |
| watch   | 是否需要一个watcher。当为true时，取默认的watcher，为false的话不使用watcher |
| cb      | 异步回调函数Children2Callback                                |
| ctx     | 传递上下文信息                                               |
| stat    | 指定数据节点的节点状态信息。。传入旧的stat，处理回调会返回将新的Stat信息放到Stat中 |

示例代码片段：同步查询子节点，并注册监视事件

```
List<String> aaa = zooKeeper.getChildren("/createAsync", new ZookeeperNativeClientImpl());
for(String a : aaa){
    System.out.println("child:"+ a);
}
Thread.sleep(20000);

@Override
public void process(WatchedEvent watchedEvent) {
    System.out.println("Receive the Eavent Masage:" + watchedEvent.toString());
    if(Event.EventType.NodeChildrenChanged == watchedEvent.getType()){
        System.out.println("节点路径："+watchedEvent.getPath()+",子节点发生变化，需要重新获取");
    }
    ...
}
```
主线程查询子节点，并注册监视事件，日志输出：

```
child:test13
child:test
child:test12
16:15:06.288 [main-SendThread(192.168.100.236:3148)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification sessionid:0x10000068b17000b
16:15:06.289 [main-SendThread(192.168.100.236:3148)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/createAsync for sessionid 0x10000068b17000b
Receive the Eavent Masage:WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/createAsync
节点路径：/createAsync,子节点发生变化，需要重新获取
```
在服务器端给/createAsync增加了五个子节点，但只收到一条变更消息，即监视器在触发变更事件之后就消失了。

异步查询方式是为了加载配置场景，防止子节点过多对于主流程的阻塞，可以实现Children2Callback接口实现异步查询回调。

### getData 获取一个节点的节点数据
获取节点数据有四个方法：

![image](https://note.youdao.com/yws/res/13095/CE79F6795A794BFE9D7BE56189A410E8)

参数说明：
| 参数名  | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| path    | 指定的数据节点路径                                           |
| watcher | 注册一个watcher。一旦在本次节点数据获取之后，节点数据（也包含节点数据版本version变化，不仅仅是存的具体数据变化）发生变更的话，那么就会像客户端发送通知，允许为null |
| watch   | 是否需要一个watcher。当为true时，取默认的watcher，为false的话不使用watcher |
| cb      | 异步回调函数接口DataCallback                                 |
| ctx     | 传递上下文信息                                               |
| stat    | 指定数据节点的节点状态信息。传入旧的stat，处理回调会返回将新的Stat信息放到Stat中 |

示例代码：同步获取一个节点数据，并注册监视器，然后更改数据后观察事件返回：

```
//查询节点数据
String path = "/createAsync";
//保存服务端的节点stat信息
Stat stat = new Stat();
byte[] aaa = zooKeeper.getData(path, new ZookeeperNativeClientImpl(), stat);
System.out.println("Path["+path+"] data:"+ new String(aaa));
System.out.println("Stat:"+ stat.getCzxid()+","+stat.getCversion());
//变更节点数据
zooKeeper.setData(path, "newdata".getBytes(), -1);
Thread.sleep(20000);

@Override
public void process(WatchedEvent watchedEvent) {
    System.out.println("Receive the Eavent Masage:" + watchedEvent.toString());
    if(Event.EventType.NodeChildrenChanged == watchedEvent.getType()){
        System.out.println("节点路径："+watchedEvent.getPath()+",子节点发生变化，需要重新获取");
    }else if(Event.EventType.NodeDataChanged == watchedEvent.getType()){
        System.out.println("节点路径："+watchedEvent.getPath()+",节点数据发生变化，需要重新获取");
    }
    ...
}
```
输出：

```
Path[/createAsync] data:createData
Stat:17179869249,6
16:48:45.166 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification sessionid:0x3000006a0b50005
16:48:45.168 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeDataChanged path:/createAsync for sessionid 0x3000006a0b50005
Receive the Eavent Masage:WatchedEvent state:SyncConnected type:NodeDataChanged path:/createAsync
节点路径：/createAsync,节点数据发生变化，需要重新获取
```
异步同理可得，不额外做示例。

### setData 更新数据
更新数据方法：

```
public Stat setData(final String path, byte data[], int version)
        throws KeeperException, InterruptedException
    {}
public void setData(final String path, byte data[], int version,
            StatCallback cb, Object ctx)
    {}
```
参数说明：
| 参数名  | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| path    | 指定的数据节点路径                                           |
| data[]  | 需要更新的数据，自行序列化                                   |
| cb      | 异步回调函数接口StatCallback                                 |
| ctx     | 传递上下文信息                                               |
| version | 指定数据节点的节点数据版本。表明本次更新是面对这个版本的节点数据。为-1表示基于当前版本更新，无原子性要求。 |

==更新数据接口很简单，与上面的接口类似，关键在于version参数。== 我们知道一般更新会先获取数据，然后更新数据，我们在getData接口中获取数据，stat中带有version信息，然后使用setData更新数据时指定是对目标version的节点数据更新，当服务器版本与请求更新的版本不一致就会更新失败，原理与==CAS原理==相通，==保证了数据的线程安全==。

### exists 检测节点是否存在
有以下方法：

![image](https://note.youdao.com/yws/res/13152/F1D90EAF959E473C8F91B5BC67B2C8B1)

参数说明：
| 参数名  | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| path    | 指定的节点路径                                               |
| watcher | 注册一个watcher。监听三个事件：==节点被创建、节点被更新、节点被删除== |
| watch   | 是否需要一个watcher。当为true时，取系统默认的watcher，为false的话不使用watcher |
| cb      | 异步回调函数接口StatCallback                                 |
| ctx     | 传递上下文信息                                               |
exists不管目标节点是否存在，都可以为其注册监听事件用来监听节点的创建、更新、删除（不涉及子节点），监听事件watcher触发了需要重新注册。

### ACL权限控制

# 开源客户端
## zkClient
## curator
Curator的机制与zkclient和原生的有一点差别，我们主要分析Curator的核心封装。

### 创建会话 CuratorFrameworkFactory
Curator使用CuratorFrameworkFactory这个工厂类来创建会话：

```
    public static CuratorFramework newClient(String connectString, RetryPolicy retryPolicy)
    {
        return newClient(connectString, DEFAULT_SESSION_TIMEOUT_MS, DEFAULT_CONNECTION_TIMEOUT_MS, retryPolicy);
    }

    /**
     * Create a new client
     *
     * @param connectString       zookeeper服务器列表：ip:host,ip:host,...
     * @param sessionTimeoutMs    session过期时间（毫秒）
     * @param connectionTimeoutMs 连接超时时间（毫秒）
     * @param retryPolicy         重连策略
     * @return client
     */
    public static CuratorFramework newClient(String connectString, int sessionTimeoutMs, int connectionTimeoutMs, RetryPolicy retryPolicy)
    {
        return builder().
            connectString(connectString).
            sessionTimeoutMs(sessionTimeoutMs).
            connectionTimeoutMs(connectionTimeoutMs).
            retryPolicy(retryPolicy).
            build();
    }
```
参数说明：

| 参数名              | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| connectString       | zookeeper服务器列表：ip:host,ip:host,...                     |
| sessionTimeoutMs    | 会话超时时间（毫秒），默认是Integer.getInteger("curator-default-session-timeout", 60 * 1000); |
| connectionTimeoutMs | 连接创建超时时间（毫秒），Integer.getInteger("curator-default-connection-timeout", 15 * 1000); |
| retryPolicy         | 重试策略。可以自定义，内置主要有几种策略：ExponentialBackoffRetry、RetryNTimes、RetryOneTime、RetryUntilElapsed、RetryForever、BoundedExponentialBackoffRetry |

自定义重试策略继承自接口RetryPolicy，实现下面的方法：
```
/**
 * 
 *由于某种原因操作失败时调用。 此方法应返回true以进行另一次尝试。
 *
 * @param retryCount 到目前为止重试的次数（第一次为0）
 * @param elapsedTimeMs 重试操作后经过的时间（以毫秒为单位）
 * @param sleeper 用它来挂起线程 - 不要调用Thread.sleep
 * @return true/false
 */
public boolean  allowRetry(int retryCount, long elapsedTimeMs, RetrySleeper sleeper);
```
#### 示例代码

```
public class ZookeeperCuratorClient {

    public static void main(String[] args) {
        try {
            String connectString = "192.168.100.236:3148,192.168.100.236:3248,192.168.100.236:3348";
            RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
            //CuratorFramework curatorClient = CuratorFrameworkFactory.newClient(connectString, 5000, 3000, retryPolicy);
            //Fluent形式创建会话实例
            CuratorFramework curatorClient = CuratorFrameworkFactory.builder()
                    .connectString(connectString)
                    .sessionTimeoutMs(5000)
                    .connectionTimeoutMs(3000)
                    .namespace("test") //实现不同的业务隔离，指定的命名空间，即指定一个根路径,后续会话的节点操作都是相对/test这个根节点的
                    .retryPolicy(retryPolicy)
                    .build();
            curatorClient.start();
            Thread.sleep(Integer.MAX_VALUE);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```
### 创建节点
代码示例：
```
...
curatorClient.start();
Thread.sleep(40000);//确保已经连接状态为CONNECTED
//创建一个节点
curatorClient.create().forPath("/nodata");
//创建一个节点并附带初始值
curatorClient.create().forPath("/hasData", "hello".getBytes());
//创建一个临时节点，初始内容为空
curatorClient.create().withMode(CreateMode.EPHEMERAL).forPath("/tmp");
//创建一个临时节点，并递归创建父节点,父节点会是持久节点
curatorClient.create().creatingParentContainersIfNeeded().withMode(CreateMode.EPHEMERAL).forPath("/hasData/child/test");
```

### 删除节点
代码示例：
```
//删除一个节点
curatorClient.delete().forPath("/nodata");
//删除节点并且级联删除子节点
curatorClient.delete().deletingChildrenIfNeeded().forPath("/hasData");
```
### 获取数据
代码示例：
```
//查询数据
byte[] result = curatorClient.getData().forPath("/hello");
System.out.println("getData path[/test/hello]:" + new String(result));
```
### 更新数据
代码示例：

```
//更新指定节点的指定版本数据
Stat stat1 = curatorClient.setData().withVersion(-1).forPath("/hello","newdata".getBytes());

//更新指定节点的指定版本数据,当forPath没有输入新数据时，去默认数据，默认数据可以在创建会话时使用defaultData(byte[])设置，
//没显式设置defaultData,系统默认取客户端IP地址（InetAddress.getLocalHost().getHostAddress().getBytes()）
Stat stat1 = curatorClient.setData().forPath("/hello");
```

### 异步接口
使用类似curatorClient.getData().inBackground()来调用异步接口。

```
public T inBackground();
public T inBackground(Object context);
public T inBackground(BackgroundCallback callback);
public T inBackground(BackgroundCallback callback, Object context);
public T inBackground(BackgroundCallback callback, Executor executor);
public T inBackground(BackgroundCallback callback, Object context, Executor executor);
```
主要讨论BackgroundCallback和Executor参数：

==Executor参数==：在Zookeeper中，所有的异步通知事件处理都是由ClientCnxn.EventThread这个线程处理,这个线程串行处理所有的通知事件。虽然这样可以很好的保证事件通知的顺序性，但是，如果处理逻辑单元比较复杂就会影响对其他事件的处理。所以允许传入一个Executor，用于将复杂的事件处理放到一个专门的线程池中执行。

异步接口的核心是需要实现BackgroundCallback接口：
```
class BackgroundTest implements BackgroundCallback{
    /**
     *
     * @param client 当前客户端实例
     * @param event 当前事件,可以获得当前事件类型和返回码
     * @throws Exception
     */
    @Override
    public void processResult(CuratorFramework client, CuratorEvent event) throws Exception {

    }
}
```

示例代码：

```
//使用自定义Executor线程来异步增加节点路径
static ExecutorService executor = new ScheduledThreadPoolExecutor(1, new ThreadFactoryBuilder().setNameFormat("extend-schedule-pool-%d").setDaemon(true).build());
static CountDownLatch countDownLatch = new CountDownLatch(2);
...
curatorClient.create().creatingParentContainersIfNeeded().withMode(CreateMode.PERSISTENT).inBackground((client, event) -> {
    System.out.println("C-Thread["+Thread.currentThread().getName()+"],ResultCode["+event.getResultCode()+"],Event["+event.getType()+"],Path["+event.getPath()+"]");
    countDownLatch.countDown();
},executor).forPath("/hasData/child/test/hhhhh");

//异步增加节点路径
curatorClient.create().creatingParentContainersIfNeeded().withMode(CreateMode.PERSISTENT).inBackground((client, event) -> {
    System.out.println("Thread["+Thread.currentThread().getName()+"],ResultCode["+event.getResultCode()+"],Event["+event.getType()+"],Path["+event.getPath()+"]");
    countDownLatch.countDown();
}).forPath("/hasData/child2/test/hhhhh");
countDownLatch.await();
executor.shutdown()
```
输出：

```
C-Thread[extend-schedule-pool-0],ResultCode[0],Event[CREATE],Path[/hasData/child/test/hhhhh]
Thread[main-EventThread],ResultCode[0],Event[CREATE],Path[/hasData/child2/test/hhhhh]
```
### 常用场景
curator提供一些便捷的场景封装，依赖curator-recipes
```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>${curator.curator}</version>
</dependency>
```


#### 事件监听（重要）
Zookeeper的原生使用Watcher来进行事件监听，使用很繁琐，需要反复重新注册Watcher，Curator使用Cache实现对Zookeeper的服务器事件进行监听。Cache是Curator对于事件监听的封装，其对事件的监听其实可以近似看做是一个本地缓存视图和远程Zookeeper视图的对比过程。

Curator能够自动反复注册监听，减轻了基于原生开发的复杂度。Cache监听类型主要分为：==节点监听和子节点监听==。

##### 节点数据监听：NodeCache
NodeCache封装了对于数据节点本身的一个变化监听，NodeCache实例构造：
```
/**
 * @param curator客户端会话
 * @param path 需要缓存监听的数据节点路径
 * @param dataIsCompressed 是否进行数据压缩，有默认不压缩的实例构造方法
 */
public NodeCache(CuratorFramework client, String path, boolean dataIsCompressed)
{
    this.client = client.newWatcherRemoveCuratorFramework();
    //路径格式校验
    this.path = PathUtils.validatePath(path);
    this.dataIsCompressed = dataIsCompressed;
}
```
示例例子：监听“/hello”节点的数据变化

```
NodeCache nodeCache = new NodeCache(curatorClient, "/hello", false);
//start中设置为true时，第一次启动时就会去读取对应节点的数据并保存在Cache中。
nodeCache.start(false);
nodeCache.getListenable().addListener(new NodeCacheListener() {
    @Override
    public void nodeChanged() throws Exception {
        System.out.println("Node Change,new data:" + nodeCache.getCurrentData().getData().toString());
    }
});
Thread.sleep(Integer.MAX_VALUE);
```
启动后，当目标节点发生变化时：

```
15:37:21.399 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification sessionid:0x3000009d2b60001
15:37:21.399 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeDataChanged path:/test/hello for sessionid 0x3000009d2b60001
15:37:21.405 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60001, packet:: clientPath:null serverPath:null finished:false header:: 9,3  replyHeader:: 9,21474836497,0  request:: '/test,F  response:: s{17179869308,17179869308,1563951182218,1563951182218,0,3,0,0,0,3,17179869326} 
15:37:21.409 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60001, packet:: clientPath:/test/hello serverPath:/test/hello finished:false header:: 10,3  replyHeader:: 10,21474836497,0  request:: '/test/hello,T  response:: s{17179869314,21474836497,1563951281132,1564558637048,8,0,0,0,6,0,17179869314} 
15:37:21.413 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60001, packet:: clientPath:/test/hello serverPath:/test/hello finished:false header:: 11,4  replyHeader:: 11,21474836497,0  request:: '/test/hello,T  response:: #313233343131,s{17179869314,21474836497,1563951281132,1564558637048,8,0,0,0,6,0,17179869314} 
Node Change,new data:[B@14a343e
15:37:22.902 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Got notification sessionid:0x3000009d2b60001
15:37:22.903 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeDataChanged path:/test/hello for sessionid 0x3000009d2b60001
15:37:22.906 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Got ping response for sessionid: 0x3000009d2b60001 after 3ms
15:37:22.907 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60001, packet:: clientPath:null serverPath:null finished:false header:: 12,3  replyHeader:: 12,21474836498,0  request:: '/test,F  response:: s{17179869308,17179869308,1563951182218,1563951182218,0,3,0,0,0,3,17179869326} 
15:37:22.913 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60001, packet:: clientPath:/test/hello serverPath:/test/hello finished:false header:: 13,3  replyHeader:: 13,21474836498,0  request:: '/test/hello,T  response:: s{17179869314,21474836498,1563951281132,1564558638554,9,0,0,0,6,0,17179869314} 
15:37:22.918 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60001, packet:: clientPath:/test/hello serverPath:/test/hello finished:false header:: 14,4  replyHeader:: 14,21474836498,0  request:: '/test/hello,T  response:: #313233343233,s{17179869314,21474836498,1563951281132,1564558638554,9,0,0,0,6,0,17179869314} 
Node Change,new data:[B@44e1386a
15:37:24.585 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Got ping response for sessionid: 0x3000009d2b60001 after 2ms
```
#### 子节点监听：PathChildrenCache
PathChildrenCache用来监听目标节点下面的直接子节点变化，间接子节点的变化是无法感知的，简单的示例：

```
PathChildrenCache pathChildrenCache = new PathChildrenCache(curatorClient, "/hello", false);
pathChildrenCache.start();
pathChildrenCache.getListenable().addListener(new PathChildrenCacheListener() {
    @Override
    public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
        if(event.getType() == PathChildrenCacheEvent.Type.CHILD_ADDED){
            System.out.println("CHILD_ADDED:"+ event.getData().getPath());
        }else if(event.getType() == PathChildrenCacheEvent.Type.CHILD_REMOVED){
            System.out.println("CHILD_REMOVED:"+ event.getData().getPath());
        }else if(event.getType() == PathChildrenCacheEvent.Type.CHILD_UPDATED){
            System.out.println("CHILD_UPDATED:"+ event.getData().getPath());
        }
    }
});
Thread.sleep(Integer.MAX_VALUE);
```
输出：

```
16:14:10.060 [main-SendThread(192.168.100.236:3248)] DEBUG org.apache.zookeeper.ClientCnxn - Got WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/test/hello for sessionid 0x2000009be940002
16:14:10.065 [main-SendThread(192.168.100.236:3248)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x2000009be940002, packet:: clientPath:/test/hello serverPath:/test/hello finished:false header:: 13,12  replyHeader:: 13,21474836502,0  request:: '/test/hello,T  response:: v{'john},s{17179869314,21474836501,1563951281132,1564560820102,10,1,0,0,6,1,21474836502} 
16:14:10.084 [main-SendThread(192.168.100.236:3248)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x2000009be940002, packet:: clientPath:/test/hello/john serverPath:/test/hello/john finished:false header:: 14,4  replyHeader:: 14,21474836502,0  request:: '/test/hello/john,T  response:: ,s{21474836502,21474836502,1564560845719,1564560845719,0,0,0,0,0,0,21474836502} 
CHILD_ADDED:/hello/john
...
CHILD_ADDED:/hello/hanna
...
CHILD_UPDATED:/hello/hanna
...
CHILD_REMOVED:/hello/john
```
### Master选举
Master选举不做过多的分析，客户端主要的实现思路是：选择一个根节点，例如：/master_select，多台机器同时向该节点创建一个子节点：/master_select/lock，根据zookeeper的机制，只有一台机器可以创建成功，成功的那台机器就是master。

### 分布式锁：InterProcessLock
Curator封装了接口InterProcessLock来实现分布式的互斥锁。

```
/**
 * 获取互斥锁 - 阻塞直到它可用。 每次获得的呼叫必须通过呼叫进行平衡
 * @throws Exception ZK errors, connection interruptions
 */
public void acquire() throws Exception;

/**
 * 获取互斥锁 - 阻塞直到它可用或给定时间到期
 * to {@link #release()}
 * @param time time to wait
 * @param unit time unit
 * @return true if the mutex was acquired, false if not
 * @throws Exception ZK errors, connection interruptions
 */
public boolean acquire(long time, TimeUnit unit) throws Exception;

/**
 * 释放一个互斥锁
 */
public void release() throws Exception;

/**
 * 如果此JVM中的线程获取了互斥锁，则返回true
 */
boolean isAcquiredInThisProcess();
```
#### 具体实现的锁

![image](https://note.youdao.com/yws/res/15757/5316630F46274775ACDE0918892C5F76)

- InterProcessMutex：分布式可重入排它锁
- InterProcessSemaphoreMutex：分布式排它锁
- InterProcessMultiLock：将多个锁作为单个实体管理的容器

这里有一个InterProcessMutex简单的示例：

```
InterProcessLock interProcessLock = new InterProcessMutex(curatorClient,"/hello");
//十个线程同时生成订单号
for (int i = 0; i < 10; i++){
    Thread thread = new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                //线程等待开始
                countDownLatch.await();
                interProcessLock.acquire();
            }catch(Exception e){
                e.printStackTrace();
            }
            SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss|SSS");
            System.out.println("thread["+Thread.currentThread().getName()+"]生成订单号："+ sdf.format(new Date()));
            try {
                interProcessLock.release();
            }catch (Exception e){

            }
        }
    });
    thread.start();
}
//开始线程并发生成订单号
countDownLatch.countDown();
```
会先输出在对应锁生成的节点，然后可以看出订单号无重复：

```
 header:: 12,15  replyHeader:: 12,21474836511,0  request:: '/test/hello/_c_5631ce5a-2104-4f2c-83d4-84cb97a96a1f-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_5631ce5a-2104-4f2c-83d4-84cb97a96a1f-lock-0000000002,s{21474836511,21474836511,1564565673968,1564565673968,0,0,0,72057635889479682,9,0,21474836511} 
 header:: 13,15  replyHeader:: 13,21474836512,0  request:: '/test/hello/_c_b618f548-0635-42f1-8140-2534d3d7e36b-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_b618f548-0635-42f1-8140-2534d3d7e36b-lock-0000000003,s{21474836512,21474836512,1564565674019,1564565674019,0,0,0,72057635889479682,9,0,21474836512} 
 header:: 14,15  replyHeader:: 14,21474836513,0  request:: '/test/hello/_c_68d84f47-2c43-4cb3-8f61-a0d094e6ce85-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_68d84f47-2c43-4cb3-8f61-a0d094e6ce85-lock-0000000004,s{21474836513,21474836513,1564565674020,1564565674020,0,0,0,72057635889479682,9,0,21474836513} 
 header:: 15,15  replyHeader:: 15,21474836514,0  request:: '/test/hello/_c_85c882d4-b70d-4718-9f0e-196bf98474ca-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_85c882d4-b70d-4718-9f0e-196bf98474ca-lock-0000000005,s{21474836514,21474836514,1564565674040,1564565674040,0,0,0,72057635889479682,9,0,21474836514} 
 header:: 16,15  replyHeader:: 16,21474836515,0  request:: '/test/hello/_c_649fcae7-4975-4696-a88c-86e4551cfb7f-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_649fcae7-4975-4696-a88c-86e4551cfb7f-lock-0000000006,s{21474836515,21474836515,1564565674043,1564565674043,0,0,0,72057635889479682,9,0,21474836515} 
 header:: 17,15  replyHeader:: 17,21474836516,0  request:: '/test/hello/_c_425d7ed1-b6d2-4613-8fd4-83ba1f436ab7-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_425d7ed1-b6d2-4613-8fd4-83ba1f436ab7-lock-0000000007,s{21474836516,21474836516,1564565674048,1564565674048,0,0,0,72057635889479682,9,0,21474836516} 
 header:: 18,15  replyHeader:: 18,21474836517,0  request:: '/test/hello/_c_463294fe-58dd-456e-85e3-7740a2ece16e-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_463294fe-58dd-456e-85e3-7740a2ece16e-lock-0000000008,s{21474836517,21474836517,1564565674052,1564565674052,0,0,0,72057635889479682,9,0,21474836517} 
 header:: 19,15  replyHeader:: 19,21474836518,0  request:: '/test/hello/_c_fbb2569f-97eb-4a35-aa85-a3b3e81a0d58-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_fbb2569f-97eb-4a35-aa85-a3b3e81a0d58-lock-0000000009,s{21474836518,21474836518,1564565674056,1564565674056,0,0,0,72057635889479682,9,0,21474836518} 
 header:: 20,15  replyHeader:: 20,21474836519,0  request:: '/test/hello/_c_99dbb726-ee44-429f-a621-ade3ace036c1-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_99dbb726-ee44-429f-a621-ade3ace036c1-lock-0000000010,s{21474836519,21474836519,1564565674060,1564565674060,0,0,0,72057635889479682,9,0,21474836519} 
 header:: 21,15  replyHeader:: 21,21474836520,0  request:: '/test/hello/_c_03354751-3df4-49d5-ac55-153de1cb73a0-lock-,#31302e302e37352e31,v{s{31,s{'world,'anyone}}},3  response:: '/test/hello/_c_03354751-3df4-49d5-ac55-153de1cb73a0-lock-0000000011,s{21474836520,21474836520,1564565674067,1564565674067,0,0,0,72057635889479682,9,0,21474836520}
```
##### 分布式可重入锁：InterProcessMutex源码分析
###### 1、获取锁实例
提供了两个构造方法：

```
public InterProcessMutex(CuratorFramework client, String path)
{
    this(client, path, new StandardLockInternalsDriver());
}
public InterProcessMutex(CuratorFramework client, String path, LockInternalsDriver driver)
{
    this(client, path, LOCK_NAME, 1, driver);
}
```
这里有一个锁驱动类：StandardLockInternalsDriver，实现了LockInternalsDriver，我们可以自定义来实现加锁：

```
//判断是够获取到了锁
public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception;
//在zookeeper的指定路径上，创建一个临时序列节点。
public String createsTheLock(CuratorFramework client,  String path, byte[] lockNodeBytes) throws Exception;
//修复排序，在StandardLockInternalsDriver的实现中，即获取到临时节点的最后序列数，进行排序
public String           fixForSorting(String str, String lockName);
```


真正调用生成：

```
/**
 * 获取锁实例
 * @param client    zookeeper客户端实例
 * @param path  目标锁路径
 * @param lockName  锁名称前缀，默认"lock-"
 * @param maxLeases 可对外的最大锁数量
 * @param driver    锁驱动类
 */
InterProcessMutex(CuratorFramework client, String path, String lockName, int maxLeases, LockInternalsDriver driver)
{
    //校验目标锁路径是否合法，并将锁路径缓存至basePath
    basePath = PathUtils.validatePath(path);
    //请求锁接口实体实例化
    internals = new LockInternals(client, driver, path, lockName, maxLeases);
}
```
关键在于对请求锁LockInternals实体的实例化，后续也是通过这个实体来发起锁请求：

```
 LockInternals(CuratorFramework client, LockInternalsDriver driver, String path, String lockName, int maxLeases)
{
    this.driver = driver;
    this.lockName = lockName;
    this.maxLeases = maxLeases;
    //为了使得监听更加精确，创建允许一次性删除所有监听器的客户端实例（便于释放锁的时候一次性删除watcher）
    this.client = client.newWatcherRemoveCuratorFramework();
    this.basePath = PathUtils.validatePath(path);
    //将path和lockName组合成一个合规的path，用于后面新增临时序列化节点
    this.path = ZKPaths.makePath(path, lockName);
}
```
###### 2、使用InterProcessMutex中的acquire()请求持有锁

```
/**
 * 获取互斥锁 - 阻塞直到它获取成功。 注意：同一个线程可重入。 Each call to acquire that returns true must be balanced by a call
 * @throws Exception ZK errors, connection interruptions
 */
@Override
public void acquire() throws Exception
{
    if ( !internalLock(-1, null) )
    {
        throw new IOException("Lost connection while trying to acquire lock: " + basePath);
    }
}
/**
 * 获取互斥锁 - 阻塞直到它可用或给定时间到期. 注意：同一个线程可重入 Each call to acquire that returns true must be balanced by a call
 * @param time 阻塞等待时间
 * @param unit 时间单位
 * @return true if the mutex was acquired, false if not
 * @throws Exception ZK errors, connection interruptions
 */
@Override
public boolean acquire(long time, TimeUnit unit) throws Exception
{
    return internalLock(time, unit);
}
```
- acquire() :入参为空，调用该方法后，会一直堵塞，直到抢夺到锁资源，或者zookeeper连接中断后，抛出异常。
- acquire(long time, TimeUnit unit)：入参传入超时时间以及单位，抢夺时，如果出现堵塞，会在超过该时间后，返回false。请默认使用这一种。

###### 3、重入锁处理

```
 private boolean internalLock(long time, TimeUnit unit) throws Exception
{
    //关于并发的注意事项：获取当前请求锁的线程，后续缓存线程获取的锁，也可以做可重入处理
    Thread currentThread = Thread.currentThread();
    LockData lockData = threadData.get(currentThread);
    if ( lockData != null )
    {
        lockData.lockCount.incrementAndGet();
        return true;
    }
    //争夺锁
    String lockPath = internals.attemptLock(time, unit, getLockNodeBytes());
    if ( lockPath != null )
    {
        LockData newLockData = new LockData(currentThread, lockPath);
        threadData.put(currentThread, newLockData);
        return true;
    }
    return false;
}
```
这段代码里面，实现了锁的可重入。每个InterProcessMutex实例，都会持有一个ConcurrentMap类型的threadData对象，以线程对象作为Key，以LockData作为Value值。通过判断当前线程threadData是否有值，如果有，则表示线程可以重入该锁，于是将lockData的lockCount进行累加；如果没有，则进行锁的抢夺。
internals.attemptLock方法返回lockPath!=null时，表明了该线程已经成功持有了这把锁，于是乎LockData对象被new了出来，并存放到threadData中。

###### 4、锁争夺
锁的争夺是这里的核心：

```
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception
{
    final long      startMillis = System.currentTimeMillis();
    final Long      millisToWait = (unit != null) ? unit.toMillis(time) : null;
    final byte[]    localLockNodeBytes = (revocable.get() != null) ? new byte[0] : lockNodeBytes;
    int             retryCount = 0;
    String          ourPath = null;
    boolean         hasTheLock = false;
    boolean         isDone = false;
    while ( !isDone )
    {
        isDone = true;
        try
        {
            //在目标锁路径下新增当前线程的临时序列节点
            ourPath = driver.createsTheLock(client, path, localLockNodeBytes);
            //根据创建的节点判断是否可以持有锁，阻塞等待（有超时限制的话会返回false'）持有锁
            hasTheLock = internalLockLoop(startMillis, millisToWait, ourPath);
        }
        catch ( KeeperException.NoNodeException e )
        {
            // 当StandardLockInternalsDriver无法找到锁定节点时，它会在会话到期时发生，等等。因此，如果重试允许，只需再试一次
            if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
            {
                isDone = false;
            }
            else
            {
                throw e;
            }
        }
    }
    //成功持有锁返回锁节点路径
    if ( hasTheLock )
    {
        return ourPath;
    }
    return null;
}
```
这里有三个关键点：
- while循环：正常情况下，这个循环会在下一次结束。但是当出现NoNodeException异常时，会根据zookeeper客户端的重试策略，进行有限次数的重新获取锁。
- driver.createsTheLock：这个driver的createsTheLock方法就是在创建这个锁，即在zookeeper的指定路径上，创建一个临时序列节点。注意：此时只是纯粹的创建了一个节点，不是说线程已经持有了锁
- internalLockLoop方法：判断自身是否能够持有锁。如果不能，进入wait，等待被唤醒

前面两点不做过多分析，分析一下internalLockLoop方法：

```
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception
{
    boolean     haveTheLock = false;
    boolean     doDelete = false;
    try
    {
        if ( revocable.get() != null )
        {
            client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
        }
        //判断客户端实例是否启动，同时本线程是否已经持有了锁，否则一直循环（当请求锁时没有设置超时）
        while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
        {
            //获取目标锁节点下的所有子节点，并且根据所有子节点后面的10位序列值从小到大排序（上面创建的节点是一个临时序列节点）
            List<String>        children = getSortedChildren();
            //获取当前线程创建的节点的名称
            String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash
            //判断当前线程是否可以持有锁
            PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
            //predicateResults.getsTheLock() == true 表示当前线程可以持有锁，否则不能持有锁需要继续等待
            if ( predicateResults.getsTheLock() )
            {
                haveTheLock = true;
            }
            else
            {
                //获取当前线程最近的上一个等待持有锁的临时节点（或者已经持有锁）
                String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();
                synchronized(this)
                {
                    try
                    {
                        // 监听临近节点的节点变化
                        // 使用getData（）而不是exists（）来避免留下不必要的观察者，这是一种资源泄漏
                        client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                        if ( millisToWait != null )
                        {
                            millisToWait -= (System.currentTimeMillis() - startMillis);
                            startMillis = System.currentTimeMillis();
                            if ( millisToWait <= 0 )
                            {
                                doDelete = true;    // timed out - delete our node
                                break;
                            }
                            //有超时的等待监听的节点发生变化
                            wait(millisToWait);
                        }
                        else
                        {
                            //线程一直挂起，等待监听的节点发生变化然后被唤醒
                            wait();
                        }
                    }
                    catch ( KeeperException.NoNodeException e )
                    {...}
                }
            }
        }
    }
    catch ( Exception e )
    {...}
    finally
    {...}
    return haveTheLock;
}
```
上面的代码也有几个关键点：
- while循环：如果你一开始使用无参的acquire方法，那么此处的循环可能就是一个死循环。当zookeeper客户端启动时，并且当前线程还没有成功获取到锁时，就会开始新的一轮循环
- getSortedChildren：这个方法比较简单，就是获取到所有子节点列表，并且从小到大根据节点名称后10位数字进行排序。在上面提到了，创建的是序列节点
- driver.getsTheLock：判断是否可以持有锁，判断规则：当前创建的节点是否在上一步获取到的子节点列表的首位。
driver.getsTheLock的逻辑是：

```
public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
{
    //获取当前线程抽奖的节点位于目标锁节点的子节点列表中的位置
    int             ourIndex = children.indexOf(sequenceNodeName);
    //锁子节点列表中没有当前线程创建的节点，抛出异常
    validateOurIndex(sequenceNodeName, ourIndex);
    //判断当前线程是否可以持有锁：创建节点的位置是否小于可获取锁数量（默认都是1个，即节点需要排在所有节点的首位）
    boolean         getsTheLock = ourIndex < maxLeases;
    //getsTheLock = true说明当前线程可以持有锁，如果不是说明其他线程持有锁，那么getsTheLock = false，此处还需要获取到自己前一个等待持有锁的临时节点的名称pathToWatch。
    // （注意这个pathToWatch后面有比较关键的作用）
    String          pathToWatch = getsTheLock ? null : children.get(ourIndex - maxLeases);
    return new PredicateResults(pathToWatch, getsTheLock);
}
```
- synchronized(this)：监听临近节点的节点变化，线程交出cpu的占用，进入等待状态，等到被唤醒。

##### 释放锁
1. 减少重入锁的计数，直到变成0。
2. 释放锁，即移除移除Watchers & 删除创建的节点
3. 从threadData中，删除自己线程的缓存

#### 分布式可重入锁原理总结
1. 某个线程需要持有某个指定路径下的锁时，先需要在该路径下创建一个==临时序列节点==，节点名字由uuid + 递增序列组成；
2. 通过对比自身的序列数是否在所有子节点的第一位，来判断是否成功获取到了锁；
3. 当获取锁失败时，它会添加watcher来监听最近一个等待节点的变动情况；
4. 直到watcher的事件生效将自己唤醒，或者超时时间异常返回。


### 分布式计数器
curator封装了一套对应的分布式计数器，用于统计在线人数等业务场景。

==基本设计原理==是：指定一个zookeeper数据节点作为计数器，多个实例应用在分布式锁的控制下，通过更新该数据节点的内容来实现计数功能。

计数器的接口定义是：DistributedAtomicNumber，通过DistributedAtomicInteger简单示例：

```
/**
 * 仅以乐观模式创建 - 即未完成对互斥锁的升级
 * @param client the client
 * @param counterPath path to hold the value
 * @param retryPolicy the retry policy to use
 */
public DistributedAtomicInteger(CuratorFramework client, String counterPath, RetryPolicy retryPolicy)
{
    this(client, counterPath, retryPolicy, null);
}
/**
 * 以互斥模式创建。 首先使用给定的重试策略进行乐观锁定 ，如果操作不成功，将尝试使用分布式锁来重试{@link InterProcessMutex}
 * @param client 客户端会话
 * @param counterPath 目标节点
 * @param retryPolicy 重试策略
 * @param promotedToLock 为互斥锁提供支持
 */
public DistributedAtomicInteger(CuratorFramework client, String counterPath, RetryPolicy retryPolicy, PromotedToLock promotedToLock)
{
    value = new DistributedAtomicValue(client, counterPath, retryPolicy, promotedToLock);
}
```
示例代码：

```
RetryPolicy retryPolicy1 = new RetryNTimes(3, 1000);
//分布式计数器
DistributedAtomicInteger atomicInteger = new DistributedAtomicInteger(curatorClient, "/hello/num", retryPolicy1);
AtomicValue<Integer> rc = atomicInteger.add(2);
//一定要检查是否操作成功
if(rc.succeeded()){
    System.out.println("计数成功");
}else{
    System.out.println("计数失败");
}
```

```
17:22:14.831 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60002, packet:: clientPath:null serverPath:null finished:false header:: 2,3  replyHeader:: 2,21474836533,0  request:: '/test,F  response:: s{17179869308,17179869308,1563951182218,1563951182218,0,3,0,0,0,3,17179869326} 
17:22:14.847 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60002, packet:: clientPath:null serverPath:null finished:false header:: 3,4  replyHeader:: 3,21474836533,-101  request:: '/test/hello/num,F  response::  
17:22:14.930 [main-SendThread(192.168.100.236:3348)] DEBUG org.apache.zookeeper.ClientCnxn - Reading reply sessionid:0x3000009d2b60002, packet:: clientPath:null serverPath:null finished:false header:: 4,15  replyHeader:: 4,21474836534,0  request:: '/test/hello/num,#0002,v{s{31,s{'world,'anyone}}},0  response:: '/test/hello/num,s{21474836534,21474836534,1564737729643,1564737729643,0,0,0,0,4,0,21474836534} 
计数成功
```

几点注意事项：
- 一定要进行rc.succeeded()判断本次操作是否成功
- 不存在目标节点时新建，但存在节点时，需要根据版本号更新:client.setData().withVersion(stat.getVersion()).forPath(path, newValue);
- 当需要操作能成功时可以使用互斥模式创建，在rc.succeeded()失败后会进入分布式锁阻塞继续等待操作（超时时间可以设置）

### 分布式线程同步执行：Barrier
==Barrier是一种多线程间控制同步的一种经典的方式==，在jdk中也有自带的实现：CyclicBarrier

#### JDK中的线程同步执行工具：CyclicBarrier
我们先分析一下jdk的实现：CyclicBarrier

```
public static void main(String[] args) {
    try {
        ExecutorService executorService = new ScheduledThreadPoolExecutor(4, new ThreadFactoryBuilder().setNameFormat("extend-schedule-pool-%d").build());
        executorService.submit(new Thread(new Runner("1号选手")));
        executorService.submit(new Thread(new Runner("2号选手")));
        executorService.submit(new Thread(new Runner("3号选手")));
        executorService.submit(new Thread(new Runner("4号选手")));
        executorService.shutdown();
    }catch (Exception e){
        e.printStackTrace();
    }
}

public class Runner implements Runnable{
    public static CyclicBarrier barrier = new CyclicBarrier(4);
    private String name;
    public Runner(String name){
        this.name = name;
    }
    @Override
     public void run() {
        System.out.println(Thread.currentThread().getName() + ":"+System.currentTimeMillis()+":" +name + " 准备好了");
        try {
            barrier.await();
        }catch (Exception e){
        }
        System.out.println(Thread.currentThread().getName()+ ":"+System.currentTimeMillis()+":" +name + " 开始！！！");
    }
}
```
输出：

```
extend-schedule-pool-0:1564740787911:1号选手 准备好了
extend-schedule-pool-1:1564740787912:2号选手 准备好了
extend-schedule-pool-2:1564740787915:3号选手 准备好了
extend-schedule-pool-3:1564740787916:4号选手 准备好了
extend-schedule-pool-3:1564740787916:4号选手 开始！！！
extend-schedule-pool-0:1564740787916:1号选手 开始！！！
extend-schedule-pool-1:1564740787916:2号选手 开始！！！
extend-schedule-pool-2:1564740787916:3号选手 开始！！！
```
可以看到，四个线程是同一时间执行的。但是这只能保证在同一JVM进程中，在分布式架构下是没办法保证的。所以需要引入分布式的线程同步执行控制工具

#### curator封装的分布式线程同步barrier
DistributedBarrier

### 工具类：ZKPaths、EnsurePath

#### ZKPaths
ZKPaths构建一些简单的API来ZNode路径，递归创建和删除节点。

# zookeeper的典型应用场景
## 数据发布/订阅
## 负载均衡
## 命名服务
## 分布式协调/通知
## 集群管理
## Master选举
## 分布式锁
## 分布式队列

#### EnsurePath
EnsurePath提供了一种确保数据节点存在的机制

# Specification
## 系统基础设计模型
- 数据结构：ZooKeeper提供的名称空间非常类似于标准Unix文件系统。名称是由斜杠（/）分隔的路径元素序列。ZooKeeper名称空间中的每个节点都由路径标识。
- Znodes：持久化和临时节点（还细分了：序列节点（单调递增）、TTL节点、容器节点）。
- Version：保证分布式数据的原子性操作（类似CAS思想）。
- Watcher：节点变更监视，主要特性有：单次性、客户端串行执行（客户端的回调是一个串行同步的过程，保证了顺序性）、轻量（通知数据结构只包含：通知状态、事件类型和节点路径）。
- ACL：保障数据的安全。

## 数据序列化与通信协议
Zookeeper客户端和服务端会进行一系列的网络通信以实现数据的传输。对于一个高性能的网络通信通道，需要考虑两点：数据的序列化和反序列化、合适高效的通讯协议。

### Jute序列化组件
Jute是Zookeeper使用的序列化组件，虽然Jute不能跟现在主流的跨语言序列化框架相提并论，如Apache Avro、Thrift和Protobuf等，但Jute目前并没有成为Zookeeper的性能瓶颈。

### 通信协议
==Zookeeper基于TCP/IP协议实现了自己的通信协议来完成客户端和服务端、服务端与服务端之间的网络通信。==

Zookeeper通信协议整体设计上较简单，对于请求，主要包含：请求头和请求体，对于响应：主要包含响应头和响应体。

![image](https://note.youdao.com/yws/res/139899/77F3FA1BDEBE461382628A789E22EB9F)

#### 协议解析：请求部分
现在详细分析请求协议的详细设计。

Zookeeper的通信协议封装在包：org.apache.zookeeper.proto 下，先关注请求协议的设计：下图是一个“获取节点数据-GetDataRequest”的请求的完整协议定义

![image](https://note.youdao.com/yws/res/139910/D834F126025844278E84F7525D105F3B)

##### 请求头-RequestHeader
请求头结构定义：org.apache.zookeeper.proto.RequestHeader

```
@Public
public class RequestHeader implements Record {
    private int xid;
    private int type;
    public RequestHeader(int xid, int type) {
        this.xid = xid;
        this.type = type;
    }
    //多余代码忽略
}
```
由源代码可以看出，请求头主要包括两个部分：
- xid：用于记录客户端请求的发起序号，用来确保对单个客户端请求的顺序响应；
- type：代表请求的操作类型，常见的有：创建节点-1，删除节点-2，获取节点数据-4等，详细见：org.apache.zookeeper.ZooDefs.OpCode

协议规定，除了会话建立的请求，其他请求都应该带上请求头。

##### 请求体-RequestBody
==协议的请求体部分指的是请求的主体内容部分，包含了请求的所有操作内容。根据不同的请求类型，其请求体的构造是不一样的==，这里我们以会话创建、获取节点数据和更新节点数据这三个典型的请求体为例来对请求体进行分析。

###### ConnectRequest 会话创建
查看其请求体数据结构：org.apache.zookeeper.proto.ConnectRequest
```
@InterfaceAudience.Public
public class ConnectRequest implements Record {
  //协议的版本号
  private int protocolVersion;
  //最近一次接收到的服务器ZXID 
  private long lastZxidSeen;
  //会话超时时间
  private int timeOut;
  //会话标识
  private long sessionId;
  //会话密码
  private byte[] passwd;
  
  //多余代码省略...
}
```
###### GetDataRequest 获取节点数据
查看其请求体数据结构：org.apache.zookeeper.proto.GetDataRequest
```
@InterfaceAudience.Public
public class GetDataRequest implements Record {
  //数据节点的节点路径
  private String path;
  //是否注册Watcher的标识
  private boolean watch;
  
  //多余代码省略...
}
```
###### SetDataRequest 更新节点数据
查看其请求体数据结构：org.apache.zookeeper.proto.SetDataRequest
```
@InterfaceAudience.Public
public class SetDataRequest implements Record {
  //数据节点的节点路径
  private String path;
  //更新数据内容字节
  private byte[] data;
  //节点数据的期望版本号
  private int version;
  
  //多余代码省略...
}
```

#### 协议解析：响应部分
根据上面的请求协议，我们来看看响应协议是怎么设计的。

举例，下面是一个获取节点数据的完整响应协议的结构：

![image](https://note.youdao.com/yws/res/140025/C6182E2D96514B18ADA14709DCD056B3)

##### ReplyHeader 响应头
查看响应头数据结构：org.apache.zookeeper.proto.ReplyHeader
```
@InterfaceAudience.Public
public class ReplyHeader implements Record {
  //xid对应请求头中的xid的值，原值返回
  private int xid;
  //代表Zookeeper服务器上当前最新的事务ID
  private long zxid;
  //错误码（Code.OK:0,Code.NONODE:101,...）
  private int err;
  
  //多余代码省略...
 }
```
##### ResponseBody 响应体
协议的响应体是指响应的主体内容部分，包含了响应的所有返回数据。不同的响应类型，其响应体结构是不一样的。

###### ConnectResponse 会话创建
org.apache.zookeeper.proto.ConnectResponse
```
@InterfaceAudience.Public
public class ConnectResponse implements Record {
  private int protocolVersion;
  private int timeOut;
  private long sessionId;
  private byte[] passwd;
 }
```

###### GetDataResponse 获取节点数据
org.apache.zookeeper.proto.GetDataResponse
```
public class GetDataResponse implements Record {
  //节点的数据内容
  private byte[] data;
  //节点状态
  private org.apache.zookeeper.data.Stat stat;
}
```

###### SetDataResponse 更新节点数据
org.apache.zookeeper.proto.SetDataResponse
```
@InterfaceAudience.Public
public class SetDataResponse implements Record {
  //节点状态
  private org.apache.zookeeper.data.Stat stat;
}
```

```
@InterfaceAudience.Public
public class Stat implements Record {
  private long czxid;
  private long mzxid;
  private long ctime;
  private long mtime;
  private int version;
  private int cversion;
  private int aversion;
  private long ephemeralOwner;
  private int dataLength;
  private int numChildren;
  private long pzxid;
}
```

## 客户端核心抽象
Zookeeper的客户端主要是由以下的几个核心组件构成：
- Zookeeper 实例：创建会话
- ClientWatchManager：客户端Watcher管理器
- HostProvider：客户端地址列表管理器
- ClientCnxn：客户端核心线程，其内部又包含两个线程，即SendThread和EventThread。

SendThread和EventThread：
- SendThread：是一个I/O线程，主要负责Zookeeper客户端和服务端之间的网络I/O通信；
- EventThread：是一个事件线程，主要负责对服务器的事件进行处理。

![image](https://note.youdao.com/yws/res/140136/4FD535E8429C41E2B6CF590D68532652)

### 一次会话的创建过程
==会话的创建大致分为三个阶段：初始化阶段、会话创建阶段、响应处理阶段==。下面是一个整体的流程图：

![image](https://note.youdao.com/yws/res/140160/ECE19F40E21F40E5B7AE614E46598807)

#### 初始化阶段
1. ==初始化Zookeeper对象==。通过调用Zookeeper的构造方法来实例化一个Zookeeper对象，在初始化过程中，会创建一个客户端的Watcher管理器：ClientWatchManager。
2. ==设置会话默认的Watcher==。如果在构造方法中传入了一个构造好的Watcher对象，那么客户端就会将这个对象作为默认Watcher保存在ClientWatchManager中。
3. ==构造Zookeeper服务器地址列表管理器：HostProvider==。对于构造方法传入的服务器地址，客户端会将其存放到服务器地址列表管理器HostProvider中。
4. ==创建并初始化客户端网络连接器：ClientCnxn==。客户端会先创建一个网络连接器ClientCnxn，用来统一管理客户端和服务端之间的交互。另外，客户端在创建ClientCnxn的同时，还会初始化客户端的两个核心队列：outgoingQueue和pendingQueue，分别作为客户端的请求发送队列和服务端响应的等待队列。（ClientCnxn连接器的底层I/O处理器是ClientCnxnSocket，所以客户端也会创建ClientCnxnSocket）
5. ==初始化SendThread和EventThread==。客户端会创建两个核心网络线程SendThread和EventThread，前者负责客户端到服务端的网络I/O，后者则用于进行客户端的事件处理。同时，客户端还会将ClientCnxnSocket分配给SendThread作为底层网络I/O处理器，并初始化EventThread的待处理事件队列waitingEvents。

#### 会话创建阶段
6. ==启动SendThread和EventThread==。SendThread首先会判断当前客户端的状态，进行一系列请理性的工作，为客户端发送”会话创建“请求做准备。
7. ==获取一个服务器地址==。在进行TCP连接前，SendThread从HostProvider中随机获取出一个地址，然后委托给CLientCnxnSocket去创建与Zookeeper服务器之间的TCP连接。
8. ==创建TCP连接==。CLientCnxnSocket负责和服务器之间创建一个TCP长连接。
9. ==构造ConnectRequest请求==。TCP连接创建后，开始进行Zookeeper的客户端会话创建，SendThread会负责根据当前客户端的实际设置，构造出一个ConnectRequest请求，该请求代表了客户端试图与服务端创建一个会话。同时，Zookeeper客户端还会进一步将该请求包装成网络I/O层的Packet对象，放入请求发送队列outgoingQueue中去。
10. ==客户端发送请求==。客户端请求准备完毕开始向服务器发送请求。ClientCnxnSocket负责从outgoingQueue中取出一个待发送的Packet对象，将其序列化为ByteBuffer后，向服务端发送。

#### 响应处理阶段
11. 接收服务器响应。ClientCnxnSocket接收到的服务端响应后会先判断客户端状态是否是”已初始化“，如果尚未完成初始化，那么就认为该响应一定是会话创建的请求响应，直接交由readConnectResult方法来处理。
12. 处理Response。ClientCnxnSocket会对接收到的数据进行反序列化，得到ConnectResponse，并从中获取到Zookeeper服务器分配的sessionId。
13. 连接成功。连接成功后，一方面要通知SendThread线程，进一步对客户端会话参数进行设置，包括readTimeout和connectTimeout等，并更新客户端状态；另一方面需要通知地址管理器HostProvider当前成功连接的服务器地址。
14. 生成事件：SyncConnected-None。为了让上层应用感知到会话的成功创建，SendThread会生成一个事件SyncConnected-None，代表着会话创建成功，并将该事件传递给EventThread线程。
15. 查询Watcher。EventThread捕获到事件，会从ClientWatchManager管理器中查询出对应的Watcher，针对SyncConnected-None事件，直接使用默认的Watcher，然后将其放到EventThread的waitingEvents队列中去。
16. 处理事件。EventThread不断地从waitingEvents队列中取出待处理的Watcher对象，然后直接调用该对象的process接口方法以达到触发Watcher的目的。

至此，Zookeeper的客户端完整的会话创建流程结束。下面会对一些关键的源码进行解读：

### 服务器地址的选择与使用
一般我们构造会话时，对于服务器地址的格式如下：

```
"127.0.0.1:2181,127.0.0.1:2281,127.0.0.1:2381"
```
如上，我们可以将集群的所有服务器地址都配置进去，那么具体在创建客户端会话的时候是跟哪个服务器进行TCP连接，还是全部连接？单个连接是顺序的还是随机的？相关的服务器地址处理源码如下：

```
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
        boolean canBeReadOnly) throws IOException {
    this(connectString, sessionTimeout, watcher, canBeReadOnly,
            createDefaultHostProvider(connectString));
}
// default hostprovider
private static HostProvider createDefaultHostProvider(String connectString) {
    return new StaticHostProvider(
            new ConnectStringParser(connectString).getServerAddresses());
}

public final class ConnectStringParser {
    private static final int DEFAULT_PORT = 2181;

    private final String chrootPath;

    private final ArrayList<InetSocketAddress> serverAddresses = new ArrayList<InetSocketAddress>();

    /**
     * 
     * @throws IllegalArgumentException
     *             for an invalid chroot path.
     */
    public ConnectStringParser(String connectString) {
        // parse out chroot, if any
        int off = connectString.indexOf('/');
        if (off >= 0) {
            String chrootPath = connectString.substring(off);
            // ignore "/" chroot spec, same as null
            if (chrootPath.length() == 1) {
                this.chrootPath = null;
            } else {
                PathUtils.validatePath(chrootPath);
                this.chrootPath = chrootPath;
            }
            connectString = connectString.substring(0, off);
        } else {
            this.chrootPath = null;
        }

        List<String> hostsList = split(connectString,",");
        for (String host : hostsList) {
            int port = DEFAULT_PORT;
            int pidx = host.lastIndexOf(':');
            if (pidx >= 0) {
                // otherwise : is at the end of the string, ignore
                if (pidx < host.length() - 1) {
                    port = Integer.parseInt(host.substring(pidx + 1));
                }
                host = host.substring(0, pidx);
            }
            serverAddresses.add(InetSocketAddress.createUnresolved(host, port));
        }
    }

    public String getChrootPath() {
        return chrootPath;
    }

    public ArrayList<InetSocketAddress> getServerAddresses() {
        return serverAddresses;
    }
}
```
根据源码，服务端地址被第一次处理之后会存入ConnectStringParser这个全局对象中封存起来。而ConnectStringParser会对地址做两个处理：
- 存在chroot的情况下，解析保存至String chrootPath；
- 保存服务器地址列表至ArrayList<InetSocketAddress> serverAddresses。

#### chrootPath：客户端命名隔离空间
客户端命名隔离空间有助于实现不同应用之间的相互隔离。如我们希望当前应用X创建和使用的所有结点都处于/apps/X/结点路径下，我们可以配置：

```
"127.0.0.1:2181,127.0.0.1:2281,127.0.0.1:2381/apps/X"
```
ConnectStringParser在解析服务器地址的时候就会解析出chrootPath=/apps/X，后续所有的节点操作都会相对于这个默认的节点路径前缀。

#### HostProvider 地址列表管理器
ConnectStringParser解析完服务器地址之后，需要将解析出来的服务器地址保存至HostProvider进行管理。
```
// 默认使用的地址列表管理器
private static HostProvider createDefaultHostProvider(String connectString) {
    return new StaticHostProvider(
            new ConnectStringParser(connectString).getServerAddresses());
}

```
HostProvider提供的管理接口有：

```
@InterfaceAudience.Public
public interface HostProvider {
    public int size();
    /**
     * 尝试连接的下一个主机。
     * 对于spinDelay为0，应该没有等待时间。
     * @param spinDelay 如果所有主机都尝试过一次，则等待spinDelay毫秒。
     */
    public InetSocketAddress next(long spinDelay);

    /**
     * 回调方法。成功连接后回调通知HostProvider成功连接。HostProvider可以使用此通知来重置其内部状态。
     */
    public void onConnected();

    /**
     * 更新服务器列表。 
     * @param serverAddresses 新的服务器列表
     * @param currentHost 此客户端当前连接的主机
     * @return 如果更改连接是负载平衡所必需的，则返回true，否则返回false。  
     */
    boolean updateServerList(Collection<InetSocketAddress> serverAddresses,
        InetSocketAddress currentHost);
}
```
客户端可以实现HostProvider构建自定义的服务器地址管理功能，默认使用的实现StaticHostProvider定义是：

![image](https://note.youdao.com/yws/res/140570/2E95140BFA464CD49C3397686A1B5C74)

StaticHostProvider是对于HostProvider一个很简单通用的实现，实现具体有几处细节：
1. 初始化对象。首先根据传入的服务器地址列表，通过Collections.shuffle()方法对传入的服务器进行随机排序（仅随机排序一次）；
2. 定义了两个指针分别是：lastIndex、currentIndex（初始化的时候值都是-1），通过这两个指针可以将上一步打乱的服务器地址构建成一个环形链表，在每次尝试获取一个服务器地址的时候，都会首先将currentIndex向前移动一位，当游标移动超过链表长度时就重置为0继续开始，保证了next()始终会返回服务器地址。lastIndex保存的是上一次成功连接的服务器位置，当currentIndex==lastIndex的时候，就进行spinDelay毫秒等待；

下面是对关键部分代码截取：（更新配置部分逻辑较多暂时不做分析）
```
private void init(Collection<InetSocketAddress> serverAddresses, long randomnessSeed, Resolver resolver) {
    this.sourceOfRandomness = new Random(randomnessSeed);
    this.resolver = resolver;
    if (serverAddresses.isEmpty()) {
        throw new IllegalArgumentException(
                "A HostProvider may not be empty!");
    }
    this.serverAddresses = shuffle(serverAddresses);
    currentIndex = -1;
    lastIndex = -1;
}
 public InetSocketAddress next(long spinDelay) {
    boolean needToSleep = false;
    InetSocketAddress addr;
    synchronized(this) {
        if (reconfigMode) {
            addr = nextHostInReconfigMode();
            if (addr != null) {
            	currentIndex = serverAddresses.indexOf(addr);
            	return resolve(addr);
            }
            //tried all servers and couldn't connect
            reconfigMode = false;
            needToSleep = (spinDelay > 0);
        }        
        ++currentIndex;
        if (currentIndex == serverAddresses.size()) {
            currentIndex = 0;
        }            
        addr = serverAddresses.get(currentIndex);
        needToSleep = needToSleep || (currentIndex == lastIndex && spinDelay > 0);
        if (lastIndex == -1) { 
            // We don't want to sleep on the first ever connect attempt.
            lastIndex = 0;
        }
    }
    if (needToSleep) {
        try {
            Thread.sleep(spinDelay);
        } catch (InterruptedException e) {
            LOG.warn("Unexpected exception", e);
        }
    }

    return resolve(addr);
}
public synchronized void onConnected() {
    lastIndex = currentIndex;
    reconfigMode = false;
}
```
基于StaticHostProvider的实现，我们在实际使用过程中，可以针对实际生产的服务器情况对于Zookeeper集群客户端管理优化：
- 配置文件方式。客户端启动的时候读取配置文件；
- 动态变更的服务器地址列表。对于那种可能变动频繁的Zookeeper集群环境，自定义实现HostProvider的地址管理器，地址管理器定时从DNS解析或者统一配置中心获取对应的服务器配置，如果发现有更新变动，则将最新的服务器地址列表更新至客户端，客户端下一次调用next()方法的时候就使用新的服务器地址列表。
- 实现同机房优先策略。网络时延在跨很多机房连接时会带来不必要的损耗，而且也会有网络分区的危险，所以在实际生产实施中，我们可以选择优先连接同机房的服务器。

### ClientCnxn 网络I/O
ClientCnxn是客户端的核心工作类，它负责维护客户端和服务器之间的网络连接和网路通信。

下面的客户端会话实例执行到初始化ClientCnxn：
```
protected final ClientCnxn cnxn;
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
        boolean canBeReadOnly, HostProvider aHostProvider,
        ZKClientConfig clientConfig) throws IOException {
    LOG.info("Initiating client connection, connectString=" + connectString
            + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);

    if (clientConfig == null) {
        clientConfig = new ZKClientConfig();
    }
    this.clientConfig = clientConfig;
    watchManager = defaultWatchManager();
    watchManager.defaultWatcher = watcher;
    ConnectStringParser connectStringParser = new ConnectStringParser(
            connectString);
    hostProvider = aHostProvider;
    // 初始化ClientCnxn
    cnxn = createConnection(connectStringParser.getChrootPath(),
            hostProvider, sessionTimeout, this, watchManager,
            getClientCnxnSocket(), canBeReadOnly);
    cnxn.start();
}

public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
        ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
        long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
    this.zooKeeper = zooKeeper;
    this.watcher = watcher;
    this.sessionId = sessionId;
    this.sessionPasswd = sessionPasswd;
    this.sessionTimeout = sessionTimeout;
    this.hostProvider = hostProvider;
    this.chrootPath = chrootPath;

    connectTimeout = sessionTimeout / hostProvider.size();
    readTimeout = sessionTimeout * 2 / 3;
    readOnly = canBeReadOnly;

    sendThread = new SendThread(clientCnxnSocket);
    eventThread = new EventThread();
    this.clientConfig=zooKeeper.getClientConfig();
    initRequestTimeout();
}
```
![image](https://note.youdao.com/yws/res/140678/9BA5B50FE3F9490A91D108040B890709)

#### Packet
Packet是ClientCnxn内部定义的一个对协议层的封装，作为Zookeeper请求与响应的载体，其数据结构如下：
```
static class Packet {
    //请求头
    RequestHeader requestHeader;
    //响应头
    ReplyHeader replyHeader;
    //请求体
    Record request;
    //响应体
    Record response;
    //底层网络传输对象，调用createBB()方法对requestHeader、request、readOnly这三个属性进行序列化，也就是说只会序列化这三个属性进行传输，其他属性都保存在客户端的上下文。
    ByteBuffer bb;

    /** 客户端的路径视图(可能会根据chroot有所不同) **/
    String clientPath;
    /** 服务端的路径视图(可能会根据chroot有所不同) **/
    String serverPath;
    //
    boolean finished;
    //
    AsyncCallback cb;
    //
    Object ctx;
    //注册的Watcher
    WatchRegistration watchRegistration;
    //是否只读
    public boolean readOnly;
    //
    WatchDeregistration watchDeregistration;

    /** Convenience ctor */
    Packet(RequestHeader requestHeader, ReplyHeader replyHeader,
           Record request, Record response,
           WatchRegistration watchRegistration) {
        this(requestHeader, replyHeader, request, response,
             watchRegistration, false);
    }
    Packet(RequestHeader requestHeader, ReplyHeader replyHeader,
           Record request, Record response,
           WatchRegistration watchRegistration, boolean readOnly) {

        this.requestHeader = requestHeader;
        this.replyHeader = replyHeader;
        this.request = request;
        this.response = response;
        this.readOnly = readOnly;
        this.watchRegistration = watchRegistration;
    }

    public void createBB() {
        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            BinaryOutputArchive boa = BinaryOutputArchive.getArchive(baos);
            boa.writeInt(-1, "len"); // We'll fill this in later
            if (requestHeader != null) {
                requestHeader.serialize(boa, "header");
            }
            if (request instanceof ConnectRequest) {
                request.serialize(boa, "connect");
                // append "am-I-allowed-to-be-readonly" flag
                boa.writeBool(readOnly, "readOnly");
            } else if (request != null) {
                request.serialize(boa, "request");
            }
            baos.close();
            this.bb = ByteBuffer.wrap(baos.toByteArray());
            this.bb.putInt(this.bb.capacity() - 4);
            this.bb.rewind();
        } catch (IOException e) {
            LOG.warn("Ignoring unexpected exception", e);
        }
    }
}
```
#### outgoingQueue和pendingQueue
ClientCnxn中定义了两个核心队列：outgoingQueue和pendingQueue
```
/**
 * 存储那些已经从客户端发送到服务端，但需要等待服务端响应额Packet集合
 */
private final LinkedList<Packet> pendingQueue = new LinkedList<Packet>();
/**
 * 存储那些需要被发送到服务器的Packet集合
 */
private final LinkedBlockingDeque<Packet> outgoingQueue = new LinkedBlockingDeque<Packet>();
```

#### ClientCnxnSocket 底层Socket通信层
ClientCnxnSocket会在Zookeeper会话创建时通过反射的方式实例化：

```
#ZooKeeper.java

private ClientCnxnSocket getClientCnxnSocket() throws IOException {
    String clientCnxnSocketName = getClientConfig().getProperty(
            ZKClientConfig.ZOOKEEPER_CLIENT_CNXN_SOCKET);
    if (clientCnxnSocketName == null) {
        clientCnxnSocketName = ClientCnxnSocketNIO.class.getName();
    }
    try {
        Constructor<?> clientCxnConstructor = Class.forName(clientCnxnSocketName).getDeclaredConstructor(ZKClientConfig.class);
        ClientCnxnSocket clientCxnSocket = (ClientCnxnSocket) clientCxnConstructor.newInstance(getClientConfig());
        return clientCxnSocket;
    } catch (Exception e) {
        IOException ioe = new IOException("Couldn't instantiate "
                + clientCnxnSocketName);
        ioe.initCause(e);
        throw ioe;
    }
}
```
由上可知，ClientCnxnSocket可以自定义然后配置使用，zookeeper3.5.5默认使用ClientCnxnSocketNIO.java作为连接实现（使用了Java的原生NIO接口，其核心是doIO逻辑）。
如：-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNIO

如使用Netty框架进行替换。

##### 请求发送
在TCP连接正常且会话有效的情况下，客户端的请求发送流程为：
1. 从outgoingQueue队列中提取出一个可发送的Packet对象；
2. 同时生成一个客户端请求序号XID并将其设置到Packet请求头中，然后序列化后将其发送；
3. 请求发送完毕后，会立即将该Packet保存到pendingQueue队列中，以便等待服务端响应返回后进行相应的处理。

> 可发送的Packet：在outgoingQueue队列中的Packet整体上是按照先进先出（FIFO）被发送的，但是如果检测到客户端和服务端之间正在处理SASL取限的话，那么那些不含请求头的（requestHeader）的Packet（例如会话创建请求）是可以被发送的，其余的都无法被发送。

![image](https://note.youdao.com/yws/res/140804/8F870C9B1FB1402FA71C9D2717DE92F4)

##### 响应接收
客户端获取来自服务端的完整响应后，根据不同的客户端请求类型，进行不同的逻辑处理：
- ==检测到客户端还未初始化==，说明当前客户端和服务器之间正在进行会话创建，那么就直接将接收到的ByteBuffer（incomingBuffer）序列化成ConnectResponse；
- ==当会话处于正常周期，并且接收到服务器响应是一个事件==，那么Zookeeper客户端会将接收到的ByteBuffer（incomingBuffer）序列化成WatcherEvent对象，并将事件放入待处理队列中。
- ==如果是一个常规的请求响应（指的是Create、GetData和Exist等操作请求）== ，那么会从pendingQueue队列中取出一个Packet来进行相应的处理。Zookeeper客户端会首先通过检验服务端响应中包含的XID值来确保请求处理的顺序性，然后再将接收到的ByteBuffer（incomingBuffer）序列化成相应的Response对象。

最后，会在finishPacket方法中处理Watcher注册等逻辑。

#### SendThread
SendThread是客户端ClientCnxn内部一个核心的I/O调度线程，用于管理客户端与服务端之间的所有网络I/O操作。在客户端实际运行过程中：
- 一方面SendThread维护了客户端与服务端之间的会话生命周期，其通过在一定的周期频率内向服务端发送一个PING包来实现心跳检测。同时，在会话生命周期内，如果客户端与服务端之间出现TCP连接断开的情况，那么就会自动且透明地完成重连操作。
- 另一方面SendThread管理了客户端所有的请求发送和响应接收操作，其将上层客户端API操作转换成相应的请求协议并发送到服务端，并完成对同步调用的返回和异步调用的回调。同时，SendThread还负责将来自服务端的事件传递给EventThread去处理。

#### EventThread
EventThread是客户端ClientCnxn内部的另一个核心线程，负责客户端的事件处理，并触发客户端注册的Watcher监听。
- EventThread中有一个waitingEvents队列，用于临时存放那些需要被触发的Object，包括那些客户端注册的Watcher和异步接口中注册的回调器AsyncCallback。
- 同时，EventThread会不断的从waitingEvents这个队列中取出Object，识别出其具体类型（Watcher或AsyncCallback），并分别调用process和processResult接口方法来实现对事件的触发和回调。

### 会话

#### 会话状态
==在Zookeeper的客户端和服务端成功完成连接后就进入了一个会话生命周期，整个会话生命周期存在不同的状态进行切换：CONNECTING、CONNECTED、RECONNECTING和CLOSE等。==

![image](https://note.youdao.com/yws/res/12757/71D35F4E5C3545DEBD4A6C7B78364A4C)

#### 会话创建
##### Session
Session是Zookeeper中的会话实体，代表了一个客户端会话。其包含以下4个基本属性：
- SessionID：会话ID，全局唯一。
- TimeOut：会话超时时间。服务器根据超时时间断开连接。
- TickTime：下次会话超时时间。为了便于Zookeeper对会话进行“分桶策略”管理，同时也是为了高效低耗地实现会话的超时检查和清理，Zookeeper会为每个会话标记一个下次会话超时时间点。其值接近于当前时间加上TimeOut，但不完全相等。
- isClosing：标记会话是否关闭。服务端检测超时会将该标识设为“已关闭”，确保不再处理该会话请求。

##### SessionID
会话创建时，服务端为客户端分配一个sessionID，标识连接的唯一性。

##### SessionTracker
SessionTracker是Zookeeper服务端的会话管理器，负责会话的创建、管理和清理等工作。会话的整个生命周期都在SessionTracker的管理下。每一个会话在SessionTracker内部保留了三份：
- sessionsById：数据结构为为ConcurrentHashMap<Long, SessionImpl>，根据sessionID管理Sesiion实体；
- sessionsWithTimeout：数据结构为ConcurrentMap<Long, Integer>，用于根据sessionId管理Session的过期时间，维护有效性，该数据结构与Zookeeper内存数据库相通，会被定期持久化到快照文件中去；
- sessionsSets：数据结构为Set，维护所有的Session池。

##### 创建连接
服务器对于客户端的会话创建响应可以分为四个步骤：
1. 处理ConnectRequest请求：使用NIOServerCnxn来负责接收来自客户端的“会话创建”请求，并反序列化出ConnectRequest请求，然后根据Zookeeper服务端的配置完成会话超时时间的协商。
2. 会话创建：SessionTracker将会为该会话分配一个SessionID，并将其注册到sessionsById和sessionsWithTimeout，同时进行会话的激活。
3. 处理器链路处理：该“会话请求”还会在Zookeeper服务端的各个请求处理器之间进行顺序流转，最终完成创建。
4. 会话响应：完成创建响应客户端。

#### 会话管理
Zookeeper服务端上面讲述了会话创建的过程，这里叙述服务端进行会话管理。

##### 分桶策略
Zookeeper的会话管理主要是由SessionTracker负责的，其采用了一种特殊的会话管理方式，我们称之为“分桶策略”。

==分桶策略：== 指将类似的会话放在同一区块中进行管理，以便于Zookeeper对会话进行不同区块的隔离处理以及同一区块的同一管理。

![image](https://note.youdao.com/yws/res/141301/BDF047F8E9A24EF1A8079945FE8AA354)

//未完待续。。。