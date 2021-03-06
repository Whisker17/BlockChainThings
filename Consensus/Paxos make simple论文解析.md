# Paxos make simple论文解析

## 问题产生的背景

在常见的分布式系统中，总会发生诸如**机器宕机**或**网络异常**（包括消息的延迟、丢失、重复、乱序，还有网络分区）等情况。Paxos 算法需要解决的问题就是**如何在一个可能发生上述异常的分布式系统中，快速且正确地在集群内部对某个数据的值达成一致，并且保证不论发生以上任何异常，都不会破坏整个系统的一致性。**

这里某个数据的值并不只是狭义上的某个数，它可以是一条日志，也可以是一条命令（command）。根据应用场景不同，某个数据的值有不同的含义。

## 基本概念

Paxos 要求满足的**前置假设**只有一个：消息内容不会被篡改；更正式的说是无拜占庭将军问题。

------

Paxos 面向的是一个理论的一致问题，这个问题的**通俗描述**是：

有一个变量 v ，分布在 N 个进程中，每个进程都尝试修改自身 v 的值，它们的企图可能各不相同，例如进程 A 尝试令 v=a ,进程 B 尝试令 v=b ，但最终所有的进程会对 v 就某个值**达成一致**，即上述例子中如果 v=a 是 v 达成一致时的值，那么 B 上，最终 v 也会为 a 。**需要注意的是某个时刻达成一致并不等价于该时刻所有进程的本地的 v 值都相同**，有一个原因是进程可能挂掉，你不能要求挂掉的进程任何事；更像是最终所有存活的进程本地 v 的值都会相同。

------

这个一致性需要满足三个要求：

1. **v 达成一致时的值是由某个进程提出的**。这是为了防止像这样的作弊方式：无论如何，最终都令每个进程的 v 为同一个预先设置好的值，例如都令 v=2 ，那么这样的一致也太容易了，也没有任何实际意义。
2. 一旦 v 就某个值达成了一致，那么 v **不能对另一个值再次达成一致**。这个要求称为 Safety。
3. **一致总是能够达成**，即 v 总会被决定为某个值。这是因为不想无休止的等待，这个要求也称为 Liveness。

------

在Paxos算法中,每个节点可以充当三种角色:

1. **Proposer**
2. Acceptor
3. **Learner**

**在具体的实现中，一个进程可能同时充当多种角色**。比如一个进程可能既是 Proposer 又是 Acceptor 又是 Learner 。

------

Paxos 保证的**一致性**如下：**不存在这样的情形，某个时刻 v 被决定为 c ，而另一个时刻 v 又决定为另一个值 d **。由这个定义我们也看到，当 v 的值被决定后，Paxos 保证了它就像是个单机的不可变变量，不再更改。也因此，对于一个客户端可以多次改写值的可读写变量在不同的节点上的一致性问题，Paxos 并不能直接解决，它需要和状态机复制结合。

------

Paxos 基于的数学原理：  我们称**大多数进程组成的集合为法定集合**，**两个法定集合必然存在非空交集，即至少有一个公共进程，称为法定集合性质**。 例如 A,B,C,D,F 进程组成的全集，法定集合 Q1 包括进程 A,B,C ， Q2 包括进程 B,C,D ，那么 Q1 和 Q2 的交集必然不在空， C 就是 Q1，Q2 的公共进程。如果要说Paxos 最根本的原理是什么，那么就是这个简单性质。同时，可以注意到，这个性质和达成一致的定义相呼应。

------

假设不同角色之间可以通过发送消息来进行通信，那么：

1. 每个角色以任意的速度执行，可能因出错而停止，也可能会重启。一个 value 被选定后，所有的角色可能失败然后重启，除非那些失败后重启的角色能记录某些信息，否则等他们重启后无法确定被选定的值。
2. 消息在传递过程中可能出现任意时长的延迟，可能会重复，也可能丢失。但是消息不会被损坏，即消息内容不会被篡改（拜占庭将军问题）。

------



## 算法推导

Paxos 算法运行在允许宕机故障的异步系统中，不要求可靠的消息传递，可容忍消息丢失、延迟、乱序以及重复。它利用大多数 (Majority) 机制保证了 **2F+1** 的容错能力，**即 2F+1 个节点的系统最多允许 F 个节点同时出现故障。**

------

暂且认为『**提案=value**』，即提案只包含 value 。在我们接下来的推导过程中会发现如果提案只包含 value ，会有问题，于是我们再对提案重新设计。

暂且认为『**Proposer 可以直接提出提案**』。在我们接下来的推导过程中会发现如果 Proposer 直接提出提案会有问题，需要增加一个学习提案的过程。

**Proposer** 可以提出（propose）提案；**Acceptor** 可以接受（accept）提案；如果某个提案被选定（chosen），那么该提案里的 value 就被选定了。

下面我们就来一步步推论Paxos算法

### 最简单的方案——只有一个Acceptor

假设只有一个 Acceptor （可以有多个 Proposer ），只要 Acceptor 接受它收到的第一个提案，则该提案被选定，该提案里的 value 就是被选定的 value 。这样就保证只有一个 value 会被选定。

但是，如果这个唯一的 Acceptor 宕机了，那么整个系统就无法工作了！

因此，必须要有**多个 Acceptor**！

![只有一个Acceptor](./pics/Paxos_1.png)

### 多个Acceptor

如果我们希望即使只有一个 Proposer 提出了一个 value ，该 value 也最终被选定。

那么，就得到下面的约束：

```
P1：一个 Acceptor 必须接受它收到的第一个提案。
```

但是，这又会引出另一个问题：如果每个 Proposer 分别提出不同的 value ，发给不同的 Acceptor 。根据 P1 ， Acceptor 分别接受自己收到的 value ，就导致不同的 value 被选定。这样就永远无法达成共识。如下图：

![多个Acceptor](./pics/Paxos_2.png)



因此，我们需要在增加一个约束：

```
规定：一个提案被选定需要被半数以上的 Acceptor 接受
```

这个规定又暗示了：『**一个Acceptor必须能够接受不止一个提案！**』不然可能导致最终没有 value 被选定。比如上图的情况。v1、v2、v3 都没有被选定，因为它们都只被一个 Acceptor 的接受。

最开始讲的『**提案=value**』已经不能满足需求了，于是重新设计提案，给每个提案加上一个提案编号，表示提案被提出的顺序。令『**提案=提案编号+value**』

虽然允许多个提案被选定，但必须保证所有被选定的提案都具有相同的 value 值。否则又会出现不一致。

于是有了下面的约束：

```
P2：如果某个 value 为 v 的提案被选定了，那么每个编号更高的被选定提案的 value 必须也是 v 。
```

一个提案只有被 Acceptor 接受才可能被选定，因此我们可以把 P2 约束改写成对 Acceptor 接受的提案的约束 P2a 。

```
P2a：如果某个 value 为 v 的提案被选定了，那么每个编号更高的被 Acceptor 接受的提案的 value 必须也是 v 。
```

只要满足了 P2a ，就能满足 P2 。

但是，考虑如下的情况：假设总的有5个Acceptor。Proposer2提出[M1,V1]的提案，Acceptor2 ~ 5（半数以上）均接受了该提案，于是对于Acceptor2 ~ 5和Proposer2来讲，它们都认为V1被选定。Acceptor1刚刚从宕机状态恢复过来（之前Acceptor1没有收到过任何提案），此时Proposer1向Acceptor1发送了[M2,V2]的提案（ **V2≠V1且M2>M1** ），对于Acceptor1来讲，这是它收到的第一个提案。根据P1（一个Acceptor必须接受它收到的第一个提案。）,Acceptor1必须接受该提案！同时Acceptor1认为V2被选定。这就出现了两个问题：

1. Acceptor1认为V2被选定，Acceptor2~5和Proposer2认为V1被选定。**出现了不一致**。
2. V1被选定了，但是编号更高的被Acceptor1接受的提案[M2,V2]的value为V2，且V2≠V1。这就跟P2a（如果某个value为v的提案被选定了，那么每个编号更高的被Acceptor接受的提案的value必须也是v）矛盾了。

![多个Acceptor](./pics/Paxos_3.png)

所以我们要对P2a约束进行强化！

P2a是对Acceptor接受的提案约束，但其实提案是Proposer提出来的，所有我们可以对Proposer提出的提案进行约束。得到P2b：

```markdown
P2b：如果某个value为v的提案被选定了，那么之后任何Proposer提出的编号更高的提案的value必须也是v。
```

由P2b可以推出P2a进而推出P2。

那么，**如何确保在某个value为v的提案被选定后，Proposer提出的编号更高的提案的value都是v呢**？

只要满足P2c即可：

```
P2c：对于任意的N和V，如果提案[N, V]被提出，那么存在一个半数以上的Acceptor组成的集合S，满足以下两个条件中的任意一个：
1. S中每个Acceptor都没有接受过编号小于N的提案。
2. S中Acceptor接受过的最大编号的提案的value为V。
```

如果这两条均不满足，即存在Acceptor接受过小于N的提案，且同时Acceptor接受过的最大编号的提案的value不是V，那么这就和我们之前所设计的方案冲突了，所以这是不正确的。

### Proposer生成提案

为了满足P2b，这里有个比较重要的思想：Proposer生成提案之前，应该先去『**学习**』已经被选定或者可能被选定的value，然后以该value作为自己提出的提案的value。**如果没有value被选定，Proposer才可以自己决定value的值**。这样才能达成一致。这个学习的阶段是通过一个**『Prepare请求』**实现的。

于是我们得到了如下的**提案生成算法：**

1. Proposer选择一个**新的提案编号N**，然后向**某个Acceptor集合**（半数以上）发送请求，要求该集合中的每个Acceptor做出如下响应（response）。
    (a) 向Proposer承诺保证**不再接受**任何编号小于N的提案。
    (b) 如果Acceptor已经接受过提案，那么就向Proposer响应**已经接受过**的编号小于N的**最大编号**的提案。

我们将该请求称为**编号为N**的**Prepare请求**。

2. 如果Proposer收到了**半数以上**的Acceptor的**响应**，那么它就可以生成编号为N，Value为V的**提案[N,V]**。这里的V是所有的响应中**编号最大的提案的Value**。如果所有的响应中**都没有提案**，那么此时V就可以由Proposer**自己选择**。
    生成提案后，Proposer将该**提案**发送给**半数以上**的Acceptor集合，并期望这些Acceptor能接受该提案。我们称该请求为**Accept请求**。**（注意：此时接受Accept请求的Acceptor集合不一定是之前响应Prepare请求的Acceptor集合）**

### Acceptor接受提案

Acceptor可以**忽略任何请求**（包括Prepare请求和Accept请求）而不用担心破坏算法的安全性。因此，我们这里要讨论的是什么时候Acceptor可以响应一个请求。

我们对Acceptor接受提案给出如下约束：

```
P1a：一个Acceptor只要尚未响应过任何编号大于N的Prepare请求，那么他就可以接受这个编号为N的提案。
```

如果Acceptor收到一个编号为N的Prepare请求，在此之前它已经响应过编号大于N的Prepare请求。根据P1a，**该Acceptor不可能接受编号为N的提案**。因此，该Acceptor可以忽略编号为N的Prepare请求。当然，也可以回复一个error，让Proposer尽早知道自己的提案不会被接受。

因此，一个Acceptor只需记住：

1. 已接受的编号最大的提案 

2. 已响应的请求的最大编号。

![优化](./pics/Paxos_4.png)

## 算法流程

经过上面的推导，我们总结下Paxos算法的流程。

Paxos算法分为两个阶段。具体如下：

- **阶段一：**

(a) Proposer选择一个提案编号N，然后向半数以上的Acceptor发送编号为N的Prepare请求。

(b) 如果一个Acceptor收到一个编号为N的Prepare请求，且**N大于该Acceptor已经响应过的所有Prepare请求的编号**，那么它就会将它已经**接受过的编号最大的提案**（如果有的话）作为响应反馈给Proposer，同时该Acceptor**承诺不再接受任何编号小于N的提案**。

- **阶段二：**

(a) 如果Proposer收到半数以上Acceptor对其发出的编号为N的Prepare请求的响应，那么它就会发送一个针对[N,V]提案的**Accept请求**给半数以上的Acceptor。注意：V就是收到的响应中编号最大的提案的value，**如果响应中不包含任何提案，那么V就由Proposer自己决定**。

(b) 如果Acceptor收到一个针对编号为N的提案的Accept请求，只要该Acceptor没有对**编号大于N的Prepare请求**做出过响应，它就接受该提案。

![Paxos算法流程](./pics/Paxos_5.png)

Paxos算法流程中的每条消息描述如下：

- **Prepare**: Proposer生成全局唯一且递增的Proposal ID (可使用时间戳加Server ID)，向所有Acceptors发送Prepare请求，这里无需携带提案内容，只携带Proposal ID即可。
- **Promise**: Acceptors收到Prepare请求后，做出“**两个承诺，一个应答**”。

**两个承诺**：

1. 不再接受Proposal ID**小于等于**（注意：这里是<= ）当前请求的**Prepare**请求。

2. 不再接受Proposal ID**小于**（注意：这里是< ）当前请求的**Propose**请求。

**一个应答**：

不违背以前作出的承诺下，回复已经Accept过的提案中Proposal ID最大的那个提案的Value和Proposal ID，没有则返回空值。

- **Propose**: Proposer 收到多数Acceptors的Promise应答后，从应答中选择Proposal ID最大的提案的Value，作为本次要发起的提案。如果所有应答的提案Value均为空值，则可以自己随意决定提案Value。然后携带当前Proposal ID，向所有Acceptors发送Propose请求。
- **Accept**: Acceptor收到Propose请求后，在不违背自己之前作出的承诺下，接受并持久化当前Proposal ID和提案Value。
- **Learn**: Proposer收到多数Acceptors的Accept后，决议形成，将形成的决议发送给所有Learners。

那么，我们来引入一张wiki上的流程图，就可以清晰地分析出Basic Paxos的简单流程了：

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(1,Vn)
   |         |<---------X--X--X------>|->|  Accepted(1,Vn)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

#### Learner学习被选定的value

Learner学习（获取）被选定的value有如下三种方案：

![Learner](./pics/Paxos_6.png)

#### 如何保证Paxos的活性（liveness）

![Liveness](./pics/Paxos_7.png)

通过选取**主Proposer**，就可以保证Paxos算法的活性。至此，我们得到一个既能保证安全性，又能保证活性的分布式一致性算法——Paxos算法。

**接下来，我们来讨论一个比较复杂的情况**：

```
Client   Leader         Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null,null,null})
   |      !             |  |  |       |  |  !! LEADER FAILS
   |         |          |  |  |       |  |  !! NEW LEADER (knows last number was 1)
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER recovers
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 2, denied
   |      X------------>|->|->|       |  |  Prepare(2)
   |      |<------------X--X--X       |  |  Nack(2)
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 3
   |      X------------>|->|->|       |  |  Prepare(3)
   |      |<------------X--X--X       |  |  Promise(3,{null,null,null})
   |      |  |          |  |  |       |  |  !! NEW LEADER proposes, denied
   |      |  X--------->|->|->|       |  |  Accept!(2,Va)
   |      |  |<---------X--X--X       |  |  Nack(3)
   |      |  |          |  |  |       |  |  !! NEW LEADER tries 4
   |      |  X--------->|->|->|       |  |  Prepare(4)
   |      |  |<---------X--X--X       |  |  Promise(4,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER proposes, denied
   |      X------------>|->|->|       |  |  Accept!(3,Vb)
   |      |<------------X--X--X       |  |  Nack(4)
   |      |  |          |  |  |       |  |  ... and so on ...
```

这就是一种**活锁**的现象，要解决这样的问题，Basic Paxos并不可以，就需要引入 [Multi Paxos](https://github.com/Whisker17/BlockChain_Get_Job/blob/master/Consensus/MultiPaxos.md)。