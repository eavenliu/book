# 分布式协议汇总

# 分布式协议
## CAP和BASE理论

### CAP理论
CAP理论告诉我们，一个分布式系统不可能同时满足一致性（C:Consistency）、可用性（A:Availability）和分区容错性（P：Partition tolerance）这三个基本需求，最多只能满足其中两项，但分区容错性是分布式系统一定要保证的，所以现在分布式一般分为CA和CP两种偏向。

### BASE理论
BASE是Basically Available（基本可用）、Soft state（软状态）和Eventually consistent（最终一致性）三个短语的简写。

BASE理论是对CAP中一致性和可用性权衡的结果，其来源于对大规模互联网系统分布式实践的总结，是基于CAP定理逐步演化而来的，其核心思想是即使无法做到强一致性，但每个应用都可以根据自身应用的特点，采用适当的方式来使系统达到最终一致性。

#### 基本可用（Basically Available）
**基本可用指的是分布式系统在出现不可预知故障的时候，允许损失部分可用性**（但不等同于系统不可用）。
- 响应时间上的损失：某个应用一般可能会在0.5秒内响应请求，但由于出现故障（机房发生断电或断网故障），查询结果的响应时间增加到了1~2秒。
- 功能上的损失：正常情况下，在一个大型电商网站上进行购物，消费者几乎可以顺利完成每一笔订单，且功能都可用，但在一些节日大促流量高峰的时候，为了保证主体购物流程的稳定性，部分消费者可能会被引导至一个降级页面。

#### 弱状态（Soft state）
**弱状态也成为软状态，和硬状态相对，是指允许系统中的数据存在中间状态，并认为该中间状态的存在不会影响系统的整体可用性，即允许系统在不同节点的数据副本之间进行数据同步的过程存在延时**。

#### 最终一致性（Eventually consistent）
**最终一致性强调的是系统中所有的数据副本，在经过一段时间的同步后，最终能够达到一个一致的状态**。因此，最终一致性的本质是需要系统保证数据能够达成一致，而不需要实时保证系统数据的强一致性。

经过很多实战，最终一致性存在以下主要的五类变种：
- 因果一致性：因果一致性是指，如果进程A在完成某个数据项的更新后通知了进程B，那么进程B之后对该数据项的访问都应该能够获取到进程A更新后的数据值，并且如果进程B要对该数据进行更新操作的话，务必基于进程A更新后的最新值，即不能发生丢失更新情况。与此同时，与进程A无因果关系的进程C的数据访问则没有这样的限制。
- 读己之所写：读己之所写是指，进程A更新一个数据项之后，它自己总是能够访问到更新过的最新值，而不会看到旧值。也就是说，对于单个数据获取者来说，其读取到的数据，一定不会比自己上次写入的值旧。因此，读己之所写也可以看作是一种特殊的因果一致性。
- 会话一致性：会话一致性将对系统数据的访问过程定在了一个会话当中：系统能保证在同一个有效的会话中实现“读己之所写”的一致性，也就是说，执行更新操作之后，客户端能够在同一个会话中读取到该数据项的最新值。
- 单调读一致性：单调读一致性指如果一个进程从系统中读取出一个数据项的某个值后，那么系统对于该进程后续的任何数据访问都不应该返回更旧的值。
- 单调写一致性：单调写一致性是指，一个系统需要能够保证来自同一个进程的写操作被顺序地执行。

**BASE理论面向的是大型高可用可扩展的分布式系统** ，和传统事务的ACID特性是相反的，他完全不同于ACID的强一致性模型，而是提出通过牺牲强一致性来获得可用性，允许数据在一段时间内是不一致的，但最终达到一致状态。但同时在实际的分布式场景中，不同业务单元对于数据一致性的要求是不同的，因此在具体系统设计中，可能会综合使用ACID和BASE理论。


## 一致性协议
上面说到了对于分布式系统中可用性和一致性的权衡，于是就产生了一系列的一致性协议。其中最著名的就是二阶段提交协议、三阶段提交协议和Paxos算法。
### 2PC与3PC
在分布式系统中，虽然每一个节点都能明确的知道自己在进行事务操作过程中的结果是成功或失败，但却无法明确获取到其他分布式节点的操作结果。

因此，当一个事务操作需要跨越多个分布式节点的时候，为了保持事务处理的ACID特性，就需要引入一个称为**“协调者（Coordinator）”**的组件来统一调度所有分布式节点的执行逻辑，这些被调度的分布式节点则被称为**“参与者（Participant）”**。协调者负责调度参与者的行为，并最终决定这些参与者是否要把事务进行真正的提交。基于这个思想，衍生出了二阶段提交和三阶段提交两种协议。
#### 2PC（2阶段提交）
**2PC，TWO-Phase Commit的缩写，即二阶段提交**，是计算机网络尤其是在数据库领域内，**为了使基于分布式架构下的所有节点在进行事务处理过程中能够保持原子性和一致性而设计的一致算法**。

**通常，二阶段提交协议被认为是一种一致性协议，用来保证分布式系统数据的一致性**。目前，绝大部分的关系型数据库都是采用二阶段提交协议来完成分布式事务处理的。**利用该协议能够非常方便的完成所有分布式事务参与者的协调，统一决定事务的提交和回滚，从而能够有效的保证分布式数据一致性，因此二阶段提交协议被广泛应用于许多分布式系统中。**

##### 协议说明
二阶段提交协议，顾名思义，它将事务的提交过程分成了两个阶段来进行处理：
###### 阶段一：提交事务请求
1. 事务询问：协调者向所有的参与者发送事务内容，询问是否可以执行事务提交操作，并开始等待各参与者的响应。
2. 执行事务：各参与者节点执行事务操作，**并将Undo和Redo信息记入事务日志中**。
3. 各参与者向协调者反馈事务询问的响应：如果参与者成功执行了事务操作，那么就反馈给协调者Yes响应，表示事务可以执行；如果参与者没有成功执行事务，那么就反馈给协调者No响应，表示事务不可以执行。

根据上面的流程，其实类似于协调者组织各参与者对一次事务操作进行投票表态的过程，因此阶段一也可以叫做**“投票阶段”**。

###### 阶段二：执行事务提交
阶段二会根据阶段一各参与者的反馈情况来决定最终是否可以进行事务提交操作，正常情况下，包含以下两种可能：
- 执行事务提交（参与者反馈的都是Yes）
1. 发送提交请求：协调者向所有参与者节点发出Commit请求。
2. 事务提交：参与者接收到Commit请求后，会正式执行事务提交操作，并在完成提交之后释放在整个事务执行期间占用的事务资源。
3. 反馈事务提交结果：参与者在完成会务提交之后，向协调者发送Ack消息。
4. 完成事务：协调者接收到所有参与者反馈的Ack消息后，完成事务。
- 中断事务（有参与者反馈了No，或者等待超时）
1. 发送回滚请求：协调者向所有的参与者发送Rollback请求。
2. 事务回滚：参与者接收到Rollback请求后，会利用其在阶段一中记录的Undo信息来执行事务回滚操作，并在完成回滚之后释放在整个事务执行期间占用的资源。
3. 反馈事务回滚结果：参与者在完成回滚操作之后，向协调者发送Ack消息。
4. 中断事务：协调者接收到所有参与者反馈的Ack消息后，完成事务中断。

**总结来说，二阶段提交协议将一个事务的处理过程分为了投票和执行两个阶段，其核心是对每个事务都采用了先尝试后提交的处理方式，因此二阶段提交协议也可以被看作是一个强一致性算法。**

![image](https://note.youdao.com/yws/res/23703/EE9857A52645497CA449128CC884FBC3)

##### 优缺点
- 优点：原理简单，实现方便
- 缺点：同步阻塞、单点问题、脑裂、太过保守
1. 同步阻塞:最明显也是最大的问题，极大的限制了分布式系统的整体性能。在二阶段提交的执行过程中，所有参与该事务操作的逻辑都处于阻塞状态，即各个参与者在等待其他参与者响应的过程中，将无法进行其他任何操作。
2. 单点问题：协调者无法很好的扩展，一旦出现问题，二阶段提交流程将无法继续进行，并且其他参与者会一直处于锁定事务状态。
3. 脑裂（数据不一致）：在阶段二的时候，即执行事务提交的时候，当协调者向所有的参与者发送Commit请求之后，发生了局部网络异常或者是协调者在尚未发送完Commit请求之前自身发生了崩溃，导致最终只有部分参与者收到了Commit请求。于是，这部分收到了Commit请求的参与者就会进行事务的提交，而其他没有收到Commit请求的参与者无法进行事务提交，于是系统就会出现数据不一致的情况。
4. 太过保守：如果协调者指示参与者进行事务提交询问的过程中，参与者出现故障没有响应，协调者只能被动根据自身的超时机制来判断是否中断事务，这样的策略很保守。即在二阶段提交协议中没有完善的容错机制，任意一个节点的失败都会造成整个事务的失败。

#### 3PC（3阶段提交）
根据上面2PC的缺点，在2PC上面的基础上进行了改进，提出了3PC（三阶段提交协议）。

##### 协议说明
3PC，是Three-Phase Commit的缩写，即三阶段提交，是2PC的改进版，就是将二阶段提交协议中的阶段一“提交事务请求”过程一分为二，形成了由CanCommit、PreCommit和do Commit三个阶段组成的事务处理协议。如下图：

![image](https://note.youdao.com/yws/res/23854/E72106BD66A34FDC99451836A4B3901D)

###### 阶段一：CanComit
1. 事务询问：协调者向所有的参与者发送一个包含事务内容的CanCommit请求，询问是否可以执行事务提交操作，并开始等待各参与者的响应。
2. 各参与者向协调者反馈事务询问的响应：参与者在接收到来自协调者的CanCommit请求后，正常情况下，如果其自身认为可以顺利执行事务，那么会反馈Yes响应，并进入预备状态，反之反馈No响应。

###### 阶段二：PreCommit
在阶段二中，协调者会根据参与者的反馈来决定是否进行事务的PreCommit操作，正常情况下，有两种可能：
- 执行事务预提交
1. 发送预提交请求：协调者向所有参与者节点发出PreCommit的请求，并进入Prepared阶段。
2. 事务预提交：参与者接收到PreCommit请求后，执行事务操作，并将Undo和Redo信息记录到事务日志中。
3. 各参与者向协调者反馈事务执行的响应：参与者成功执行事务操作，向协调者发出Ack响应，同时等待最终指令：提交（Commit）或中止（abort）。
- 中断事务
存在由参与者反馈No响应或是等待响应超时，进入中断事务：
1. 发送中断请求：协调者向所有参与者发送abort请求。
2. 中断事务：无论是收到来自协调者的abort请求，还是在等待协调者请求过程中出现超时，参与者都会中断事务。

###### 阶段三：doCommit
该阶段进行真正的事务提交，存在以下两种可能：
- 执行提交
1. 发送提交请求：协调者接收到所有参与者的Ack响应，它从“预提交”状态转换到“提交”状态，并向所有参与者发送doCommit请求。
2. 事务提交：参与者在接收到doCommit请求后，会正式执行事务提交操作，并在完成提交之后释放在整个事务执行阶段占用的事务资源。
3. 反馈事务提交结果：参与者在完成事务提交之后，向协调者发送Ack消息。
4. 完成事务：协调者接收到所有参与者反馈的Ack消息后，完成事务。
- 中断事务：存在由参与者反馈No响应或是等待响应超时，进入中断事务：
1. 发送中断请求：协调者向所有参与者发送中断”abort“请求。
2. 事务回滚：参与者接收到abort请求后，会利用其在第二阶段的事务日志Undo信息来执行回滚操作，并在回滚之后释放整个事务期间占用的资源。
3. 反馈事务回滚结果：回滚之后，参与者向协调者发送Ack消息。
4. 中断事务：协调者接收到所有参与者反馈的Ack消息后，中断事务。

需要注意，一旦进入阶段三，可能会存在以下故障：
- 协调者出现异常；
- 协调者和参与者出现网络故障。

无论出现哪种情况，都会导致参与者无法及时接收到来之协调者的doCommit或是abort请求，针对这种情况，参与者都会在等到超时后，继续进行事务提交。

##### 优缺点
- 优点：相较于二阶段提交协议，三阶段提交协议最大的优点就是降低了参与者的阻塞范围，并且能够在出现单点故障后继续达成一致。
- 缺点：三阶段提交协议在去除阻塞的同时也引入了新的问题，那就是在参与者接收到preCommit消息后，如果出现网络分区，该参与者依然会进行事务的提交，这必然会出现数据的不一致性。
### Paxos
Paxos算法是Leslie Lamport于1990年提出的**一种基于消息传递且具有高度容错性的一致性算法**，是目前公认的解决分布式一致性问题最有效的算法之一。

分布式系统中，总会发生机器宕机和网络故障等情况。Paxos算法需要解决的问题就是如何在一个可能发生上述异常的分布式系统中，快速且正确的在集群内部对某个数据的值达成一致，并且保证不论发生以上任何异常，都不会破坏整个系统的一致性。

关于Paxos，著名的场景是“拜占庭将军问题”。但是从理论上来说，在分布式领域，试图在异步系统和不可靠的通道上来达成一致性状态是不可能的，因此在一致性的研究中，都往往假设信道是可靠的。（即假设消息都是完整且没有被篡改的）

#### 算法详解
详见:基于Paxos算法的日志复制同步.md


### ZAB协议
**ZAB协议是为分布式协调服务Zookeeper专门设计的一种支持崩溃恢复的原子广播协议。ZAB协议并不像Paxos算法那样是一种通用的分布式一致性算法，它是一种特别为zookeeper设计的崩溃可恢复的原子广播算法。 基于该协议，Zookeeper实现了一种主备模式的系统架构来保持集群中各副本之间的数据一致性。**

ZAB协议的核心是定义了对于那些会改变Zookeeper服务器的数据状态的事务请求的处理方式，即：
- 所有事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被称为Leader服务器，而余下的其他服务器则成为Follower服务器。
- Leader负责将一个客户端事务请求转换成一个事务Proposal（提议），并将该Proposal分发给集群中所有的Follower服务器。
- 之后Leader服务器需要等待所有Follower服务器的反馈，一旦超过了半数的Follower服务器进行了正确的反馈，那么Leader就会再次向所有的Follower服务器分发Commit消息，要求其将上一个Proposal进行提交。

#### 协议介绍
ZAB协议包含两种基本的模式（形态）：
- 崩溃恢复
- 消息广播

上诉两种模式一般是集群在不同情况下使用ZAB协议需要达到的目的，一般是从快速的崩溃恢复到稳定的消息广播过程。简单介绍如下：
1. 当整个集群服务启动时，或是当Leader服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB协议就会进入恢复模式并选举产生新的Leader服务器；
2. 当选举了新的Leader服务器，同事集群中过半机器与该Leader服务器完成了状态同步之后，ZAB协议就会退出恢复模式；（其中，所谓的状态同步指的是数据同步，用来保证集群中存在过半的机器能够和Leader服务器的数据状态保持一致）
3. 当集群中有过半的Follower服务器与Leader服务器完成了状态同步，那么集群就可以进入消息广播模式了；
4. 当一台遵守ZAB协议的服务器启动后加入集群时，如果此时集群中已经有一个Leader服务器在负责进行消息广播时，那么新加入的服务器会自觉进入数据恢复模式：与Leader服务器进行数据同步，然后加入到消息广播中去；
5. Zookeeper设计只允许一个Leader服务器进行事务请求的处理，Leader服务器在接收到客户端的事务请求后，会生成对应的事务提案发起一轮广播协议；而集群中的其他Follower服务器接收到客户端的事务请求，会首先将这个事务请求转发给Leader服务器；
6. 当Leader服务器出现崩溃或者机器重启时，亦或是集群过半的Follower服务器不能与Leader服务器保持通讯，那么在重新开始新一轮的原子广播事务之前，所有进程首先会使用崩溃恢复协议来使彼此达到一个一致的状态，于是整个ZAB流程就会从消息广播模式进入到崩溃恢复模式。

以上其实就是ZAB协议在实际应用中的实施效果，下面分别具体描述下细节。

##### 消息广播
**ZAB协议中的消息广播过程使用的是个原子广播协议，类似于一个二阶段提交过程。** 流程如下图：

![image](https://note.youdao.com/yws/res/24742/F0AD85200101407D90517A8436E65F7A)

此处的原子广播协议与上面说的2PC协议还是有所不同。在ZAB协议的二阶段提交过程中，移除了中断逻辑，所有的Follower服务器要么正常反馈Leader提出的事务Proposal，要么就抛弃Leader服务器。同时，移除中断逻辑意味着只需要等待过半服务器反馈ack之后就可以提交事务Proposal了，而不需要等待集群中所有Follower服务器都反馈响应。

上面这种简化的二阶段提交模型是无法处理Leader服务器崩溃退出而带来的数据不一致问题的，因此在ZAB协议中添加了另一种模式，即采用崩溃恢复模式来解决这个问题。

**整个消息广播协议是基于有FIFO特性的TCP协议来进行网络通信的，因此能够很容易的保证消息广播过程中消息接收和发送的顺序性。**

**为了更严格的保证事务消息的顺序和因果性，我们为每一个事务请求分配了一个全局唯一单调递增的事务ID（即ZXID），必须将每一个事务Proposal按照其ZXID的先后顺序来进行排序和处理。**

**消息广播的具体过程**：
1. Leader服务器会为每一个Follower服务器都各自分配一个单独的队列，然后将需要广播的事务Proposal依次放入这些队列中去，并根据FIFO策略进行消息发送；
2. 每一个Follower服务器接收到这个事务Proposal之后，都会首先将其以事务日志的方式写入到本地磁盘中去，并且在成功写入之后反馈给Leader服务器一个Ack响应；
3. 当Leader服务器接收到了半数Follower服务器的事务响应Ack后，就会广播一个Commit消息给所有的Follower服务器以通知其进行事务提交，同时Leader服务器自身也会完成对事务的提交，而每一个Follower服务器在接收到Commit消息后，也会完成对事务的提交。

##### 崩溃恢复
上面已经简单接介绍了崩溃恢复，要能实现故障快速崩溃恢复的效果，ZAB协议需要一个高效且可靠的Leader选举算法，从而确保能够快速地选举出新的Leader。同时，Leader选举算法不仅仅需要让Leader知道自身已经被选举为了Leader，同时还需要让集群中的所有其他机器也能够快速地感知到选举产生的新的Leader服务器。

###### 基本特性
- **ZAB协议需要确保那些已经在Leader服务器提交的事务最终被所有服务器提交。** 针对一个事务在Leader服务器上提交了，并且得到了半数Follower服务器的反馈，但是它将Commit消息发送给所有Follower服务器之前宕机了的情况。
- **ZAB协议需要确保丢弃那些只在Leader服务器上被提出的事务。** 针对在崩溃恢复过程中出现一个需要被丢弃的提案，那么在崩溃恢复结束后需要跳过该事务Proposal。

根据上面对于ZAB协议的要求，就决定了ZAB协议必须设计这样一个Leader选举算法：**能够确保提交已经被leader提交的事务Proposal，同时丢弃已经被跳过的事务Proposal** 。针对这个要求，如果让Leader选举算法能够保证新选举出来的Leader服务器拥有集群中所有机器的最高事务编号（即ZXID最大）的事务Proposal，那么久可以保证这个新选举出来的Leader一定具有所有已经提交的提案。更为重要的是，如果让具有最高编号事务Proposal的机器来成为Leader，就可以省去Leader服务器检查Proposal的提交和丢弃工作的这一步操作了。

###### 数据同步
**在完成Leader选举之后，新Leader服务器会首先确认事务日志中的所有Proposal是否都已经被集群中过半的机器提交了，及是否完成数据同步。**

数据同步过程（正常情况下）：
1. Leader服务器需要确保所有的Follower服务器能够接收到每一条事务Proposal，并且能够正确地将所有已经提交了的事务Proposal应用到内存数据库中；
2. Leader服务器为每一个Follower都准备一个队列，并将那些没有被各Follower服务器同步的事务以Proposal消息的形式逐个发送给Follower服务器，并在每一个Proposal消息后面紧接着再发送一个Commit消息，以表示该事务已经被提交。
3. 等到Follower服务器将所有其尚未同步的事务Proposal都从Leader服务器上同步过来并成功应用到本地数据库中后，Leader服务器就会将该Follower加入到真正可用的Follower列表中，并开始之后的其他流程。

处理需要被丢弃的事务Proposal：
- 事务编号ZXID是一个63位的数字，其中低32位可以看作是一个简单的单调递增的计数器，针对每一个客户端的事务请求，Leader服务器产生一个新的事务Proposal的时候都会对该计数器进行加1操作；
- ZXID的高32位则代表了Leader周期epoch的编号，每当选举一个新的Leader服务器，就会从这个新Leader取出其本地事务日志中最大的事务Proposal的ZXID，并从该ZXID中解析出对应的epoch值，然后对其进行加1操作，之后就会以此编号作为新的epoch，并将低32位置0来开始生成新的ZXID。
- ZAB协议通过epoch编号来区分Leader的周期变化的策略，能够有效避免不同的Leader服务器错误的使用相同的ZXID编号提出不一样的事务Proposal，这对于识别崩溃恢复前后生成的Proposal非常有帮助，简化了数据恢复流程；
- 基于这样ZXID的策略，当一个包含了上一个Leader周期中尚未提交过的事务Proposal的服务器启动时，其肯定无法成为Leader。因为当前集群中一定包含个Quorum集合，该集合中的机器一定包含了更高的epoch的事务Proposal，因为这台机器的事务Proposal肯定不是最该，所以无法成为Leader。当这台机器加入到集群中，Leader服务器会根据自己服务器上最后被提交的Proposal来和Follower服务器的Proposal进行比对，比对的结果当然是Leader会要求Follower进行一个回退操作---回退到一个确实已经被集群中过半机器提交的最新的事务Proposal。

#### 算法描述
我们来从算法描述角度来深入讲解ZAB协议的内部原理。

**根据上面说的整个ZAB协议只要包括消息广播和崩溃恢复两个过程，进一步可以细分为三个阶段，分别是：发现（Discovery）、同步（Synchronization）和广播（Broadcast）阶段**。组成ZAB协议的每一个分布式进程，会循环执行这三个阶段，我们将这样一个循环称之为一个**主进程周期**。

##### 阶段一：发现
**阶段一主要就是Leader选举过程，用于在多个分布式进程中选举出主进程。** 准Leader L和Follower F的工作流程如下：
1. Follower F将自己最后接受的事务Proposal的epoch值CEPOCH()发送给准Leader L；
2. 当接收来自过半的Follower的CEPOCH()消息后，准Leader L会生成NEWCEPOCH()消息给这些过半的Follower。（关于这个新的epoch值，准Leader L会从所有接收到的epoch值中选出最大的然后对其进行加1操作）
3. 当Follower接收到来自准Leader L的NEWCEPOCH()消息后，检测当前的epoch是否小于接收到的值，如果小于，就会将接收到的值赋值给epoch，同时向这个准Leader L反馈Ack消息。这个反馈消息中包含了当前该Follower的epoch值，以及该Follower的历史事务Proposal集合：hf。
4. 当Leader L收到过半的确认消息Ack后，Leader L 就会从这过半的服务器中选取出一个Follower F，并使用其作为初始化事务集合Ie。

##### 阶段二：同步
在完成发现流程后，就进入了同步阶段。主要的目的是实现数据同步，即一致性初始化事务集合。

##### 阶段三：广播
完成了同步之后，ZAB协议就可以正式开始接受客户端新的事务请求了，并进行消息广播流程。

##### 三个阶段消息收发总结
![image](https://note.youdao.com/yws/res/25119/EB16B9359AD74D48A40CC9D0DD05EB64)

-  （发现）CEPOCH：Follower进程向准Leader发送自己处理过的最后一个事务Proposal的epoch值（ZXID的高32位）；
-  （发现）NEWEPOCH：准Leader进程根据接收的各进程的epoch值，取其中最大的加1来生成新一轮的epoch值；
-  （发现）ACK-E：Follower向准Leader反馈发来的NEWEPOCH消息；
-  （同步）NEWLEADER：准Leader进程确认了自己的领导地位，发送NEWLEADER消息给各Follower；
-  （同步）ACK-LD：Follower进程反馈准Leader发出的确认领导消息；
-  （同步）COMMIT-LD：Leader要求Follower进程提交对应的历史事务Proposal；
-  （广播）PROPOSE：Leader为每一个客户端事务请求生成对应的Proposal消息；
-  （广播）ACK：Follower持久化并响应Leader发出的Proposal消息；
-  （广播）COMMIT：Leader发出Commit消息，要求所有Follower提交事务PROPOSE。

集群正常运行过程中，ZAB协议会一直运行于阶段三来反复进行消息广播。如果因为Leader宕机或者网络分区，那么ZAB协议会再次进入阶段一。

#### 运行分析
每一个应用ZAB的进程都有可能处于一下三种状态之一：
- LOOKING：Leader选举阶段
- FOLLOWING：Follower服务器和Leader保持同步状态
- LEADING：Leader服务器作为主进程领导状态

#### ZAB和Paxos算法的异同
总的来说，ZAB和Paxos算法的本质区别在于两者的设计目标不一样。
- ZAB协议主要用于构建一个高可用的分布式数据主备系统，例如Zookeeper；
- Paxos用于构建一个分布式的一致性状态机系统。

##### 两者的联系
- 两者都存在一个类似于Leader进程的角色，由其来负责协调多个Follower进程的运行；
- Leader进程都会等待多数的Follower做出正确的反馈后才会对个提案进行提交；
- 在ZAB协议中，每个Proposal都包含一个epoch值用来代表当前的Leader周期，在Paxos算发中，同样存在这样一个标识，只是名字变成了Ballot。

##### 两者的差异
在Paxos算法中，一个新选举产生的Leader会进行两个阶段的工作：读阶段，写阶段。
- 读阶段：新Leader通过和其他进程通信的方式收集上一个Leader提出的提案，并将它们提交；
- 写阶段：当前Leader开始提出它自己的提案。

ZAB在Paxos的算法基础上额外增加了一个同步阶段，在同步阶段之前，ZAB协议的发现阶段与Paxos读阶段非常相似。在同步阶段，新的Leader会确保存在过半的Follower已经提交了之前Leader周期中的所有事务Proposal。同步阶段的引入，能够有效的保证Leader在新的周期提出事务Proposal之前，所有的进程都已经完成了对之前所有事务的提交（保证数据的一致）。一旦完成同步，那么ZAB就进入和Paxos类似的写阶段。

### RAFT协议
Raft协议可以称之为简易的Paxos协议实现，更容易被理解，但其性能、可能性、可用性等方面却是不输于Paxos的。




# 参考
《从Paxos到Zookeeper分布式一致性原理与实践》


```
成员协议：
SWIM（Scalable Weakly-consistent Infection-style Process Group Membership Protocol， 可伸缩的弱一致性传染式进程组成员协议)
https://blog.csdn.net/book_vbstar/article/details/84920802

一致性协议：
2PC
Paxos
https://www.cnblogs.com/hugb/p/8955505.html

Raft算法（强一致性）


Gossip protocol 也叫 Epidemic Protocol （流行病协议），实际上它还有很多别名，比如：“流言算法”、“疫情传播算法”等（最终一致性协议）
https://www.jianshu.com/p/8279d6fd65bb

其他：
DHT算法

自增ID算法：
Snowflake
```