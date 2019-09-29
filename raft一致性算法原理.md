# [raft一致性算法详解](https://i6448038.github.io/2018/12/12/raft/)

2018-12-12

在现实的分布式系统中，不能可能保证集群中的每一台机器都是100%可用可靠的，集群中的任何机器都可能发生宕机、网络连接等问题导致集群中的某个节点不可用，这样，那个节点的数据就有可能和集群不一致，所以需要有一种机制，来保证在大多数机器都存在的情况下向外提供可靠的数据服务。这里的大多数节点指的是`集群半数以上`的节点。

raft算法就是一种在分布式系统中解决集群中多节点之间数据一致性的算法。Golang生态圈中大名鼎鼎的etcd就是使用的raft算法来保持数据一致性的，与raft类似的一致性算法还有Paxos算法、Zab协议等。

其实，raft算法维持数据一致性的核心思想很简单，就是：“少数服从多数”。

# leader选举

保证数据一致性，最好的方式就是只有唯一的一个节点，唯一的这个节点读，唯一的这个节点写，这样数据肯定是一致的；但是分布式架构显然不可以一个节点，于是，raft算法提出，在集群的所有节点中，需要有一个节点来充当这一个唯一的节点，在一段时间内，只有这一个节点负责读写数据，然后其他节点同步数据。这个唯一的节点叫`leader`节点，其他负责同步数据的节点叫做`follower`节点。在集群中，还会有其他状态的节点，例如`candidate`节点，这种节点只有在选举`leader`的时候才会有。
节点的`leader`选举和现实生活中的选举十分类似，就是投票，集群中获票数最多的那个，就是`leader`节点，所以为防止出现平局的情况(平局的情况也有解决方案，下文会说)，一般在部署节点的时候，会将节点数设置为奇数(2n + 1)。

这些节点是如何选举的呢？我们先从`follower`、`leader`、`candidate`这三种状态说起。
在集群中，有三个节点 A 、B、C。

![img](https://i6448038.github.io/img/raft/1.jpeg)

在集群刚刚开始的时候，他们仨都是`follower`。

![img](https://i6448038.github.io/img/raft/2.jpeg)

过一段时间后，A变成了`Candidate`，这是要选举了！

![img](https://i6448038.github.io/img/raft/3.jpeg)

为啥A能变成`Candidate`？凭啥？因为A的`election timeout`到期了，`election timeout`是选举超时时间，集群中的每个节点都有一个`election timeout`，每个节点的`election timeout`都是150ms ~ 300ms之间的一个随机数。每当节点的`election timeout`时间到了，就会触发节点变为`candidate`状态。A的选举超时时间到了，所以A理所当然变为了`Candidate`。
所以，我们知道，其实A、B、C三个节点除了有状态，还有个选举超时时间`election timeout`

![img](https://i6448038.github.io/img/raft/4.jpeg)

此时，`candidate`节点A会向整个集群发起选举投票，它会先投自己一票，然后告诉B、C 大选开始了！

![img](https://i6448038.github.io/img/raft/5.jpeg)

注意！只有`candidate`状态的节点，才可以参加竞选变为`leader`，B、C这两个follower是没有资格的！
除此之外，每个节点中还有一个字段，叫`term`意思就是任期，和美国大选的第几期总统差不多一个意思，这个`term`是一个全局的、连续递增的整数，每进行一次选举，`term`就会加一，如果`candidate`赢得选举，它会当`leader`直到此次任期结束。
此时，A触发了选举，它的`term`应该是加一的。

![img](https://i6448038.github.io/img/raft/6.jpeg)

当B、C收到A发出的大选消息后，B、C开始投票，此时只有A这一个`candidate`，所以理所当然发消息都只能投A。

![img](https://i6448038.github.io/img/raft/7.jpeg)

此时A当选`leader`!
为了巩固自己的“统治”，防止A在任期之间其他节点因为自身`election timout`而触发选举，`leader`节点A会不定时的向两个`follower`节点B、C发送心跳消息，B和C收到心跳消息后，会重置`election timout`。心跳检测时间很短，要远远小于选举超时时间`election timout`。

![img](https://i6448038.github.io/img/raft/8.jpeg)

B、C收到心跳检测后，返回心跳响应，并重置超时时间`election timeout`。

![img](https://i6448038.github.io/img/raft/9.jpeg)

假设A发送的心跳检测消息由于网络原因例如延迟、丢包等等没有传送到B、C中的某个Follower节点，而此时这个节点刚好`election timeout`，则触发选举。
C修改自身节点任期值`term`为2，自身状态变为`candidate`，且投自身一票后，发起选举！

![img](https://i6448038.github.io/img/raft/10.jpeg)

这时候，由于C的任期值`term`变为2大于A的，在raft协议中，但收到任期值大于自身的节点，都会更改自身节点的term值，并切换为`Follower`状态并重置`election time`。
因此，这时候A由`leader`直接变为`Follower`！

![img](https://i6448038.github.io/img/raft/11.jpeg)

我们再来考虑一种极端情况：假设有偶数个节点，并且同时有两个节点进入`candiate`状态！
例如有以下四个节点A、B、C、D。A和B同时进入了`cadidate`状态并开始选举。

![img](https://i6448038.github.io/img/raft/12.jpeg)

假如A和B中任意一个获得了超过半数以上的多数票，则变为leader！

![img](https://i6448038.github.io/img/raft/13.jpeg)

但是假如两个经过一次选举后得的票数相同或者都没有超过半数，则宣告选举失败并结束！等待A和C这两个`candidate`节点中任意一个节点的`election time`超时，然后发起新一轮选举。
注意：虽然票数相同或者都没有超过半数导致的选举失败了，但是任期值`term`还是要叠加的！
A、B票数相同，等待哪个先超时。

![img](https://i6448038.github.io/img/raft/14.jpeg)

此时A先超时。则A发起选举，由于A`term`值显然是最大的，则A会最终当选为`leader`。

![img](https://i6448038.github.io/img/raft/15.jpeg)

# 日志复制

当`leader`选出来后，无论读和写都会由`leader`节点来处理。
是的，读也由`leader`来处理，`leader`拿到请求后，再决定由哪一个节点来处理，要么将请求分发，要么自己处理；即使client端请求的是follower节点，Follower节点也会现将请求信息转给`leader`，再由`leader`决定由哪个节点来处理。

下面来说说写的情况：
以下有A、B、C三个节点，其中A是`leader`节点

![img](https://i6448038.github.io/img/raft/16.jpeg)

当client请求过来要求写操作的时候，`leader` A先把数据写在本身节点的log文件中

![img](https://i6448038.github.io/img/raft/17.jpeg)

然后A将发`append entries`消息发送给B、C节点。
注意！`append entries`消息其实是根据节点的不同而消息也不同的，因为集群中数据可能不一致，一味的传相同数据，显然不可以。具体怎么不一致，稍后再说。

![img](https://i6448038.github.io/img/raft/18.jpeg)

B、C再收到消息后，把数据添加到本地，然后向A发消息，告诉A已经收到。

![img](https://i6448038.github.io/img/raft/19.jpeg)

`leader`A收到后，先提交记录，然后返回客户端响应。

![img](https://i6448038.github.io/img/raft/20.jpeg)

然后，`leader`A继续向B、C两个follower发送写数据commit的通知。

![img](https://i6448038.github.io/img/raft/21.jpeg)

B、C两个节点收到通知后，先commit自身的log数据，然后再通知`leader`A已更新结束。

![img](https://i6448038.github.io/img/raft/22.jpeg)

到此整个数据同步也就结束了。
每次写数据，都需要先更新，然后commit。每个节点中都有两个索引，一个是当前提交的索引值commitIndex，一个是目前数据的最后一行索引值lastApplied。

![img](https://i6448038.github.io/img/raft/23.jpeg)

而leader节点中，除了需要存储自身节点的commitIndex和lastApplied之外，还需要知道所有`Follower`的存储情况，因而`leader`节点中多了一张表，这张表中记录了所有`follower`节点的存储情况，这张表中有两个属性，一个属性叫`nextIndex`记录的是`Follower`节点没有的数据索引，需要发送`append entries`的数据索引；还有一个`matchIndex`记录的是`leader`节点已知的，`follower`节点的数据。如下图所示:

![img](https://i6448038.github.io/img/raft/24.jpeg)

因此，当数据更新的时候，`leader`A 向节点B、C发送不同的`append entries`。

![img](https://i6448038.github.io/img/raft/25.jpeg)

当A节点不再当leader时，其他节点并不能知道`leader`A保存的`matchIndex`和`nextIndex`这两个数组的数据。当其他节点成功当选为`leader`节点后，会将`nextIndex`全部重置为自身的`commitIndex`，而`matchIndex`则全部重置为0。如下图：

![img](https://i6448038.github.io/img/raft/26.jpeg)

则，`leader`B会向A和C节点发`append entries`，去”补全”数据

![img](https://i6448038.github.io/img/raft/27.jpeg)

节点收到请求后，如果存在数据，就不动直接返回，如果没有数据则缺哪个补哪个。

# 总结

- 触发选举的唯一条件是`election timeout`，心跳超时等其他条件仅仅是触发了非`leader`节点的`election timeout`。
- 节点选举的时候，`term`值大的一定会力压`term`值小的当选leader。
- `leader`节点向`follower`节点中发送`append entries`的时候，并不是缺少1、2、3就直接发送1、2、3而是分批次，一次发送一条。1！ 2！ 3！三条数据，分三次发完。(怕图片误导，特此说明!)