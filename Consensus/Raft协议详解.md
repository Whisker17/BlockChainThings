# RAFT协议详解

我们都知道，当代分布式一致性协议可以说是由Paxos领导的，而Paxos的出发点是从分布式一致性问题的角度出发，而我们本文要介绍的Raft则是从多副本状态机的角度提出推导。

分布式存储系统通常通过维护多个副本来进行容错，提高系统的可用性。要实现此目标，就必须要解决分布式存储系统的最核心问题：**维护多个副本的一致性。**

**一致性：**在一个具有一致性的性质的集群中，同一时刻中所有的节点对于存储在其中的某个值都有相同的结果，即对其共享的存储保持一致。

一致性协议就是用来干这事的，用来保证**即使在部分(确切地说是小部分)副本宕机**的情况下，系统仍然能正常对外提供服务。一致性协议通常基于replicated state machines（复制状态机），即所有结点都从同一个state出发，都经过同样的一些操作序列（log），最后到达同样的state。

## 架构

![架构](./pics/raft_1.png)

系统中每个节点有三个组件：

1. 状态机：**当我们说一致性的时候，实际就是在说要保证这个状态机的一致性。**状态机会从log里面取出所有的命令，然后执行一遍，得到的结果就是我们对外提供的保证了一致性的数据。
2. Log：保存了所有的修改记录。
3. 共识模块：共识模块算法就是用来保证写入的log的命令的一致性，这也是raft算法核心内容。

### 复制状态机

我们知道，复制状态机通过集群方式，由此给客户提供一个强一致、高可用的数据服务。所谓强一致，就是客户看来他们可以一直读到最新的写成功的数据；而在服务内部就是一个所有状态机中的数据高度一致的现象。而高可用，就是即使出现网络延迟、丢包、乱序、服务器宕机的情况，数据已就可以保证一致性。

对于复制状态机来说，他们是按序读取本地的log信息，由此来进行计算从而得到一个输出的。

### Log

![架构](./pics/raft_12.png)

我们可以看到，上图就是我们Raft协议的一个核心数据结构——Log。

1. 每一条日志都有日志序号（log index）
2. 每一条日志记录除了保存 state machine 要执行的操作（如 x ← 0）外，还需要保存该操作对应的 Term
3. **由Leader决定什么时候对记录进行commit**：当一条记录已经在至少半数的Follower上持久化后，Leader可以将该记录commit，并提供给state machine执行对应的操作
4. **Leader给Follower的复制日志请求中（AppendEntries RPC），不仅包含具体的操作，还包含本次插入的前一个位置的index和Term。**

## 协议内容

Raft 是一种分布式一致性协议，类似的协议还有我们之前讲过的Paxos，可以把 Raft 协议看作是 Paxos 协议的变种。Raft 主打的是易理解，其易理解主要在两方面：

1. 强化 leader 的作用，日志只能从 leader 流向 follower
2. 日志是连续的，中间没有空洞

即，**Raft是一种基于多副本的强Leader的一致性算法**

Raft协议的每个副本都会处于三种状态之一：**Leader、Follower、Candidate**

![架构](./pics/raft_11.png)

1. **Leader：**所有请求的处理者，Leader副本接受client的更新请求，本地处理后再同步至多个其他副本；
2. **Follower：**请求的被动更新者，从Leader接受更新请求，然后写入本地日志文件。
3. **Candidate：**如果Follower副本在一段时间内没有收到Leader副本的heartbeat，则判断Leader可能已经故障，此时启动选主过程，此时副本会变成Candidate状态，直到选主结束。

### Leader Election

当Leader节点由于异常（宕机、网络故障等）无法继续提供服务时，可以认为它结束了本轮任期（Term=n），需要开始新一轮的Leader Election，而新的Leader当然要从Follower中产生，开始新一轮的任期（Term=n+1）。

我们在这里简单介绍一下Term的含义，作为一个logic clock，Term有两个主要作用：

1. 识别过期信息
2. 通过限制使得在每个Term之内只能进行一次投票，由此来保证每个Term之中只会存在一个Leader。

首先我们明确，系统的初始状态是不存在Leader的，各个成员都是以Follower的形式存在，由于在一定时间内没有收到Heartbeat，此时Follower会进行一个竞选的过程：

1. 节点状态从Follower变成Candidate，对自己节点的Term自加1
2. 所有Candidate节点给自己投1票，同时向其他所有节点发送拉票请求（RequestVoteRPC）
3. Candidate节点等待投票结果

在这里我们有三种可能：

1. **自己被选成了主。**当收到了majority的投票后，状态切成Leader，并且定期给其它的所有server发Heartbeat消息（不带log的AppendEntriesRPC）以告诉对方自己是current_term_id所标识的term的leader。**每个term最多只有一个Leader**，term id作为logical clock，在每个RPC消息中都会带上，**用于检测过期的消息**。当一个server收到的RPC消息中的rpc_term_id比本地的current_term_id更大时，就更新current_term_id为rpc_term_id，并且如果当前state为leader或者candidate时，将自己的状态切成follower。如果rpc_term_id比本地的current_term_id更小，则拒绝这个RPC消息。
2. **别人成为了主。**如1所述，当Candidator在等待投票的过程中，收到了大于或者等于本地的current_term_id的声明对方是leader的AppendEntriesRPC时，则将自己的state切成follower，并且更新本地的current_term_id。
3. **没有选出主。**当投票被瓜分，没有任何一个candidate收到了majority的vote时，没有leader被选出。这种情况下，每个candidate等待的投票的过程就超时了，**接着candidates都会将本地的current_term_id再加1**，发起RequestVoteRPC进行新一轮的leader election。

投票策略：

* 每个节点只会给每个term投一票，具体的是否同意和后续的Safety有关。
* 当投票被瓜分后，所有的candidate同时超时，然后有可能进入新一轮的票数被瓜分，为了避免这个问题，Raft采用一种很简单的方法：每个Candidate的election timeout从150ms-300ms之间随机取，那么第一个超时的Candidate就可以发起新一轮的leader election，带着最大的term_id给其它所有server发送RequestVoteRPC消息，从而自己成为leader，然后给他们发送心跳消息以告诉他们自己是主。

下面我们来看一个具体的范例：

![架构](./pics/raft_2.png)

我们可以看到，相较于选出的Leader，下面的Follower有些缺少Log，有些则有多余的Log，甚至有的有多也有少。这样的情况是怎么出现的呢？比如节点f，出现这样的情况就可能是：

1. 节点f在Term 2时是Leader，在提交了几条记录后，还未执行commit就下线
2. 节点f在Term 3时再次上线并成为Leader，在提交了几条记录后，还未执行commit（包括Term 2的commit）就又一次下线





![架构](./pics/raft_3.png)

