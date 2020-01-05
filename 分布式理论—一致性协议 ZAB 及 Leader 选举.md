## 分布式理论—一致性协议 ZAB 及 Leader 选举

### 1、ZAB 协议介绍

​	ZAB 协议全称：Zookeeper Atomic Broadcast（Zookeeper 原子广播协议）。是一个为分布式应用提供高效且可靠的分布式协调服务。在解决分布式一致性方面，Zookeeper 并没有使用 Paxos ，而是采用了 ZAB 协议。**ZAB 协议是为分布式协调服务 Zookeeper 专门设计的一种支持 <u>崩溃恢复</u> 的 <u>原子广播</u> 协议**。

​	在 ZooKeeper 中，主要依赖 ZAB 协议来实现分布式数据一致性，基于该协议，Zookeeper 实现了一种 **主备模式** 的系统架构来保持集群中各个副本之间 **数据一致性**。

​	所有客户端写入数据都是写入到 **主进程**（称为 Leader）中，然后，由 Leader 复制到备份进程（称为 Follower）中。从而保证数据一致性。从设计上看，和 Raft 类似。

​	ZAB 只需要 Follower 有一半以上返回 **Ack** 信息就可以执行提交，大大减小了**同步阻塞**。也提高了可用性。

​	整个 Zookeeper 就是在这**两个模式之间切换**。 简而言之，当 Leader 服务可以正常使用，就进入消息广播模式，当 Leader 不可用时，则进入崩溃恢复模式。

### 2、消息广播

​	ZAB 协议的消息广播过程使用的是一个原子广播协议，类似一个 **二阶段提交过程**。对于客户端发送的写请求，全部由 Leader 接收，Leader 将请求封装成一个事务 Proposal，将其发送给所有 Follwer ，然后，根据所有 Follwer 的反馈，如果超过半数成功响应，则执行 commit 操作（先提交自己，再发送 commit 给所有Follwer）。

​	**整个广播流程分为 3 步骤：**（1）将数据都复制到 Follwer 中、（2）等待 Follwer 回应 Ack，最低超过半数即成功、（3）当超过半数成功回应，则执行 commit ，同时提交自己。如此以来，就能够保持集群之间数据的一致性。实际上，在 Leader 和 Follwer 之间还有一个消息队列，用来解耦他们之间的耦合，避免同步，实现异步解耦。

其中还有一些需要了解的**细节**：

1. Leader 在收到客户端请求之后，会将这个请求封装成一个事务，并给这个事务分配一个全局递增的唯一 ID，称为事务ID（ZXID），ZAB 协议需要保证事务的顺序，因此必须将每一个事务按照 ZXID 进行先后排序然后处理。
2. 在 Leader 和 Follwer 之间还有一个消息队列，用来解耦他们之间的耦合，解除同步阻塞。
3. zookeeper 集群中为保证任何所有进程能够有序的顺序执行，只能是 Leader 服务器接受写请求，即使是 Follower 服务器接受到客户端的请求，也会转发到 Leader 服务器进行处理。
4. 实际上，这是一种简化版本的 2PC，不能解决单点问题（即 Leader 崩溃问题）。

### 3、崩溃恢复

​	所谓崩溃，是指 Leader 节点失去与过半 Follwer 的联系。那么在消息广播过程中，Leader 崩溃怎么办？还能保证数据一致吗？如果 Leader 先本地提交了，然后 commit 请求没有发送出去，怎么办？

##### 首先看下主从架构下，leader 崩溃，数据一致性怎么保证？

​	leader 崩溃之后，集群会选出新的 leader，然后就会进入恢复阶段，新的 leader 具有所有已经提交的提议，因此它会保证让 followers 同步已提交的提议，丢弃未提交的提议（以 leader 的记录为准），这就保证了整个集群的数据一致性。这也就说明了如果 Leader 先本地提交了，然后 commit 请求没有发送出去，这时该提议是未提交状态，所以要丢弃。

##### 选举 leader 的时候，整个集群无法处理写请求的，如何快速进行 leader 选举？

​	这是通过 **Fast Leader Election** 实现的，leader 的选举只需要超过半数的节点投票即可，这样不需要等待所有节点的选票，能够尽早选出 leader。

### 4、Zookeeper Leader 选举

​	服务器初始化启动、服务器运行期间无法和 Leader 保持连接，Leader 节点崩溃，逻辑时钟崩溃。基于这两点，要进行 Leader 选举。

​	zookeeper提供了三种选举算法,默认的算法是 `FastLeaderElection` ,简单点就是投票数大于半数则胜出：

**LeaderElection**、**AuthFastLeaderElection**、**FastLeaderElection** *(default)*。

#### 再来了解几个概念：

**myid** *(服务器ID)*：比如有三台服务器，编号分别是1，2，3。编号越大在选举算法中的权重越大。

**state** *(当前服务器的状态)*：

​	**LOOKING** : 竞选状态。

​	**FOLLOWING** : 随从状态，同步leader状态，参与投票。

​	**OBSERVING** : 观察状态，同步leader状态，不参与投票。

​	**LEADING** : 领导者状态。

**zxid** *(事务ID)*：被推举的Leader事务ID，服务器中存放的最新数据version，值越大说明数据越新，在选举中数据越新权重越大。

**electionEpoch** *(逻辑时钟)*：逻辑时钟，用来判断多个投票是否在同一轮选举周期中，该值在服务端是一个自增序列，每次进入新一轮的投票后，都会对该值进行加1操作。

**peerEpoch**：被推举的Leader的epoch。

#### 选举机制概述：

**投票下处理规则**：

​	首先对比zxid。zxid大的服务器优先作为Leader

​	若zxid相同，比如初始化的时候，每个Server的zxid都为0，就会比较myid，myid大的选出来做Leader。	

**服务器初始化时选举**（目前有3台服务器，每台服务器均没有数据，它们的编号分别是1,2,3按编号依次启动，它们的选择举过程如下：）：

​	Server1启动，给自己投票（1,0），然后发投票信息，由于其它机器还没有启动所以它收不到反馈信息，Server1的状态一直属于Looking。

​	Server2启动，给自己投票（2,0），同时与之前启动的Server1交换结果，由于Server2的编号大所以Server2胜出，**但此时投票数正好大于半数**，所以Server2成为领导者，Server1成为小弟。

​	Server3启动，给自己投票（3,0），同时与之前启动的Server1,Server2换信息，尽管Server3的编号大，但之前Server2已经胜出，所以Server3只能成为小弟。

​	当确定了Leader之后，每个Server更新自己的状态，Leader将状态更新为Leading，Follower将状态更新为Following。

**服务器运行期间的选举**：

​	zookeeper 运行期间，如果有新的 Server 加入，或者非 Leader 的 Server 宕机，那么 Leader 将会同步数据到新 Server 或者寻找其他备用 Server 替代宕机的 Server。若 Leader 宕机，此时集群暂停对外服务，开始在内部选举新的 Leader。假设当前集群中有 Server1、Server2、Server3三台服务器，Server2 为当前集群的 Leader，由于意外情况，Server2 宕机了，便开始进入选举状态。过程如下：

​	变更状态。其他的非 Observer 服务器将自己的状态改变为 Looking，开始进入 Leader 选举。

​	每个 Server 发出一个投票（myid，zxid），由于此集群已经运行过，所以每个 Server 上的 zxid 可能不同。假设 Server1 的zxid为100，Server3 的为99，第一轮投票中，Server1和Server3都投自己，票分别为（1，100）,（3, 99）,将自己的票发送给集群中所有机器。

​	每个 Server 接收接收来自其他 Server的投票，接下来的步骤与启动时步骤相同。

也可参考：<https://juejin.im/post/5b924b0de51d450e9a2de615>

