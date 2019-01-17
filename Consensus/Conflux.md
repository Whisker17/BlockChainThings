# Conflux

## 概述

conflux中其设计最重要的一环在于**推迟交易的总排序并优化处理并发交易和块**。

这是为什么呢？理由在于，在区块链中交易是不太会发生冲突的，那么Conflux就乐观的假设并发块中的交易默认是不会发生冲突的，所以我们只需要考虑在块生成器指定的“happens-before”关系就行了。这样就可以减少Conflux对于共识的约束，从而加快达成共识的速度。

## 架构

![Conflux架构](./pics/conflux_1.png)

通过上图，我们可以看到Conflux的整体架构，那么，接下来我们来介绍一下各个组件。

* **Gossip Network**

Conflux中的所有节点都是通过Gossip Network进行连接的。每当新产生一笔交易时，通过Gossip Network进行广播；每当新生成一个区块，也是通过Gossip Network进行广播，例如，上图中Tx9和B这个区块。

* **Pending Transaction Pool**

上图右下角即为Transaction Pool。每一个节点维护一个Transaction Pool，交易池中包含所有尚未被打包的的交易，一旦节点通过Gossip Network接收到一笔交易，就会打入到交易池中；而一旦节点得知一个新区块的产生（无论是自身产生的还是监听到的），就会将交易池中的所有这个区块中的交易清除，比如上图中的Tx1，Tx2，Tx5和Tx3。这些交易我们会通过共识模块来处理，以防止在Conflux并行块中的重复打包。

* **Block Generator**

每个节点都运行一个Block Generator，用以生成新的有效区块对交易进行打包，Block Generator生成的区块可以通过调整区块头来适应不同的共识机制，比如PoW或者PoS之类的。

* **Local DAG State**

每个节点都维护一个Local DAG State，用以记录整体的交易记录。

------

