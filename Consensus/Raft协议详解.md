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

不难解释，不是所有的Follower都有资格成为Leader，因为如果一个Follower缺少之前的Leader已经commit的Log，那么它无论如何都无法复制到缺少的那部分Log的，所以我们在这里有一个约束：

**Candidate在拉票时需要携带自己本地已经持久化的最新的日志信息，等待投票的节点如果发现自己本地的日志信息比竞选的Candidate更新，则拒绝给他投票。**

### Log Replication

当Leader被选出来后，就可以接受客户端发来的请求了，每个请求包含一条需要被replicated state machines执行的命令。leader会把它作为一个log entry append到日志中，然后给其它的server发AppendEntriesRPC请求。当Leader确定一个log entry被safely replicated了（大多数副本已经将该命令写入日志当中），就apply这条log entry到状态机中然后返回结果给客户端。如果某个Follower宕机了或者运行的很慢，或者网络丢包了，Leader会无限的重试 AppendEntries RPC直到所有的Followers最终存储了所有的日志条目。

当一条日志是commited时，Leader才可以将它应用到状态机中。Raft保证一条commited的log entry已经持久化了并且会被所有的节点执行。

当一个新的Leader被选出来时，它的日志和其它的Follower的日志可能不一样，这个时候，就需要一个机制来保证日志的一致性。

下面我们来看一个具体的范例：

![架构](./pics/raft_2.png)

我们可以看到，相较于选出的Leader，下面的Follower有些缺少Log，有些则有多余的Log，甚至有的有多也有少。这样的情况是怎么出现的呢？比如节点f，出现这样的情况就可能是：

1. 节点f在Term 2时是Leader，在提交了几条记录后，还未执行commit就下线
2. 节点f在Term 3时再次上线并成为Leader，在提交了几条记录后，还未执行commit（包括Term 2的commit）就又一次下线

当然，在出现上述情况的时候，我们依靠单纯的复制Log是不行的，所以我们需要这样做：

Leader为每一个Follower节点维护一个nextIndex计数，对于每一个Follower节点，首先设置nextIndex为Leader节点的下一个index位置（上图中为11），然后依次向前比较Leader和Follower对应的记录，直到找到重合的记录为止，（对于节点f来说，即上图中的3）再将所有Leader节点的记录复制到Follower节点。

由此，我们可以总结一句话：**Leader负责一致性检查，同时让所有的Follower都和自己保持一致**

------

之前我们说过Leader的选举不是任意的，而是有限制的，即需要Candidate拥有上一任Leader的所有已经commit的Log，但是光光有这个限制是不够的，我们再来看一个实例：

![架构](./pics/raft_3.png)

1. 在阶段a，term为2，S1是Leader，且S1写入日志（term, index）为(2, 2)，并且日志被同步写入了S2；

2. 在阶段b，S1离线，触发一次新的选主，此时S5被选为新的Leader，（由于S2已同步了term为2的Log，根据之前我们的限制，S2是不会投票给S5的，那么只有可能是S3，S4和S5自己投票给自己，由此让S5成为Leader）此时系统term为3，且写入了日志（term, index）为（3， 2）;

3. S5尚未将日志推送到Followers就离线了，进而触发了一次新的选主，而之前离线的S1经过重新上线后被选中变成Leader，此时系统term为4，此时S1会将自己的日志同步到Followers，按照上图就是将日志（2， 2）同步到了S3，而此时由于该日志已经被同步到了多数节点（S1, S2, S3），因此，此时日志（2，2）可以被commit了。

4. 在阶段d，S1又下线了，触发一次选主，而S5有可能被选为新的Leader（这是因为S5可以满足作为主的一切条件：1. term = 5 > 4，2. 最新的日志为（3，2），比大多数节点（如S2/S3/S4的日志都新）），然后S5会将自己的日志更新到Followers，于是S2、S3中已经被提交的日志（2，2）被截断了。

这就引出了一个问题，我们在增加上述限制后，依旧还存在问题：即使日志（2，2）已经被大多数节点（S1、S2、S3）确认了，但是它不能被提交，因为它是来自之前term（2）的日志，直到S1在当前term（4）产生的日志（4， 4）被大多数Followers确认，S1方可提交日志（4，4）这条日志，当然，根据Raft定义，（4，4）之前的所有日志也会被提交。此时即使S1再下线，重新选主时S5不可能成为Leader，因为它没有包含大多数节点已经拥有的日志（4，4）。

因此，我们对Raft的Leader选举提出了又一个限制：**只允许Leader提交（commit）当前Term的日志**

------

Raft日志同步保证如下两点：

- 如果不同日志中的两个条目有着相同的索引和任期号，则它们所存储的命令是相同的。
- 如果不同日志中的两个条目有着相同的索引和任期号，则它们之前的所有条目都是完全一样的。

第一条特性源于Leader在一个term内在给定的一个log index最多创建一条日志条目，同时该条目在日志中的位置也从来不会改变。

第二条特性源于 AppendEntries 的一个简单的一致性检查。当发送一个 AppendEntries RPC 时，Leader会把新日志条目紧接着之前的条目的log index和term都包含在里面。如果Follower没有在它的日志中找到log index和term都相同的日志，它就会拒绝新的日志条目。

### Safety

我们来回顾一下之前我们定义的一些约束：

1. **Candidate在拉票时需要携带自己本地已经持久化的最新的日志信息，等待投票的节点如果发现自己本地的日志信息比竞选的Candidate更新，则拒绝给他投票。**
2. **只允许Leader提交（commit）当前Term的日志**

第一条帮助我们限制了Leader的选举，从而保证了不会有已经被commit的Log因为宕机或者丢包之类的情况而导致的Log丢失。在任何基于Leader的共识算法里，我们必须要保证Leader的正确性，确定Leader包含所有已经commit的Log，这样就可以保证我们的数据流只会从Leader流向Follower，而Leader永远不会重写自己日志中已经存在的entry log。

------

### Log Compaction

在实际的系统中，不能让日志无限增长，否则系统重启时需要花很长的时间进行回放，从而影响availability。Raft采用对整个系统进行snapshot来处理，**snapshot之前的日志都可以丢弃**。Snapshot技术在Chubby和ZooKeeper系统中都有采用。

Raft使用的方案是：**每个副本独立的对自己的系统状态进行Snapshot，并且只能对已经提交的日志记录（已经应用到状态机）进行snapshot。**

Snapshot中包含以下内容：

- 日志元数据。最后一条已提交的 log entry的 log index和term。这两个值在snapshot之后的第一条log entry的AppendEntries RPC的完整性检查的时候会被用上。一旦这个server做完了snapshot，就可以把这条记录的最后一条log index及其之前的所有的log entry都删掉。
- 系统当前状态。

snapshot的缺点就是**不是增量**的，**即使内存中某个值没有变，下次做snapshot的时候同样会被dump到磁盘**。当leader需要发给某个follower的log entry被丢弃了(因为leader做了snapshot)，leader会将snapshot发给落后太多的follower。或者当新加进一台机器时，也会发送snapshot给它。发送snapshot使用新的RPC，InstalledSnapshot。

做snapshot既不要做的太频繁，否则消耗磁盘带宽， 也不要做的太不频繁，否则一旦节点重启需要回放大量日志，影响可用性。推荐当日志达到某个固定的大小做一次snapshot。

做一次snapshot可能耗时过长，会影响正常日志同步。可以通过使用**copy-on-write**技术避免snapshot过程影响正常日志同步。

### 集群成员变化

集群成员变化是在集群运行过程中副本发生变化，如增加/减少副本数、节点替换等

成员变更也是一个分布式一致性问题，既所有服务器对新成员达成一致。但是成员变更又有其特殊性，因为在成员变更的一致性达成的过程中，参与投票的进程会发生变化。

如果将**成员变更当成一般的一致性问题**，直接向Leader发送成员变更请求，Leader复制成员变更日志，达成多数派之后提交，各服务器提交成员变更日志后从旧成员配置（Cold）切换到新成员配置（Cnew）。

因为各个服务器提交成员变更日志的时刻可能不同，造成各个服务器从旧成员配置（Cold）切换到新成员配置（Cnew）的时刻不同。

成员变更不能影响服务的可用性，但是成员变更过程的某一时刻，可能出现在Cold和Cnew中同时存在两个不相交的多数派，进而可能选出两个Leader，形成不同的决议，破坏安全性。

![架构](./pics/raft_13.png)

**上图就是成员变更的某一时刻Cold和Cnew中同时存在两个不相交的多数派**

