---
title: 6-824-LAB-2B
date: 2022-12-08 10:22:53
cover: https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/img000.jpeg
categories: [编程, 编程.Labs, MIT 6.824]
toc: true
---
`TODO`
客户端发送的命令会被`leader`包装成一个**entry**并添加到它的日志中，然后广播**append**消息让其他服务器复制日志。当**entry**被<u>**安全复制**</u>后，`leader`上的复制状态机会应用其内部命令，并返回执行结果给客户端。另外，即使`leader`已经答复了客户端，为了满足一致性，也会对那些因为网络、奔溃或运行缓慢等而没成功复制日志的服务器重复发送**append**消息。服务器底层日志本质是一个**entry**数组，每个**entry**包含一条命令和**term**（指代**entry**创建时所处的任期或者说接收到客户端命令的`leader`的任期）。
<!--more-->
## 流程分析

![appendentries-rpc](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/appendentries-rpc.png)

下面讨论安全复制，当一个**entry**被复制到主体服务器后，我们称它和在其之前的全部entry都是可提交的。而`leader`会一直记录可提交**entry**的最高下标即**commitIndex**，并在发送**append**消息以及心跳时一并传递。当一个`follower`知道一个**entry**时可提交时，它会将其应用到本地状态机上。

Raft协议的***Log Matching***属性保证了<u>通过下标和**term**可以唯一锁定一个**entry**</u>。当`leader`发送**append**消息时，会同时发送紧邻新**entry**的前一个**entry**的下标和**term**，如果`follower`没找到这个**entry**就拒绝，找到就日志复制。这样当`leader`的**append**消息一旦返回成功，就表示该`follower`此刻日志和**leader**一致。

Raft用`leader`的日志去覆盖`follower`的冲突日志部分来解决非一致性问题。简单来说，就是找到`leader`和`follower`日志公共部分的最后一个**entry**，删除`follower`在该**entry**之后的全部日志，然后用`leader`该**entry**之后的日志替换。为了便于管理，`leader`为每个`follower`准备了一个**nextIndex**去指代替换日志的第一个**entry**的下标。<u>当leader当选时，会将每个nextIndex都指向自己日志最后一个entry之后的位置（last index + 1）</u>。当日志不一致时，**append**消息会返回失败给`leader`，`leader`会减少**nextIndex**，这样反复调整直到**nextIndex**到达正确的位置。`follower`结果日志复制后和`leader`保持一致，并返回成功。

对上述算法进一步改进，给`reply`添加两个参数**conflictIndex**和**ConflictTerm**，当`follower`因为日志太短而找不到**prevIndex**时则让**nextIndex**指向其日志最后一个**entry**之后的位置；当`follower`发现**prevIndex**指向下标所在**entry**的**term**和自己的日志不一致时，则看`leader`有没有**entry**是那个**term**的，如果有就让**nextIndex**指向`follower`那个**term**的第一个**entry**所在下标，否则就让**nextIndex**指向`leader`那个**term**的最后一个**entry**之后一个位置（根据***leaderCompleteness***可以证明，`leader`必然要短一些）。

***Leader Completeness***属性保证了一个**entry**一旦被提交，就不会被更替。为了实现该属性，Raft规定投票时需要判断是否***up-to-date***；还规定规定不能通过判断是否主体复制来提交非当前任期的**entry**（figure 8）。

如果`follower`和`candidate`崩溃了，那么RPCs返回失败，Raft会重试RPCs请求。即使在完成RPCs任务后返回响应前失败，也做相似处理。



## 细节处理

来自[guide](https://thesquareplanet.com/blog/students-guide-to-raft/)的关于常见问题的分析和处理，这些问题通常源于我们对Raft论文的理解和原作者产生分歧，或者遗漏掉一些限定以及用错误的方式去处理问题导致。

***Livelocks***

在Lab2中，产生**活锁**的常见场景是，刚选出一个新`leader`，此时一个其他节点又发起了新的选举导致新`leader`立马被抛弃。解决这个问题的关键在于重置选举计时器的时机，我们只会在如下三个场景重置：<u>a）从当前leader处获得**append消息**时；b）发起新的选举时；c）向candidate投票时。</u>

![requestvote-rpc](https://cdn.jsdelivr.net/gh/zion4h/picture-home@main/requestvote-rpc.png)

其中c情况尤为特别，当网络不稳定时，不同服务器底层日志可能有差异，这时候只会有少数几个服务器满足***up-to-date***要求。如果每当有投票请求就重置计时器而不是确定投票时重置，那么满足***up-to-date***日志的服务器可能迟迟不能开始选举。另外，一旦candidate超时就要立刻发起新的选举。

在处理RPCs时，时刻遵循**Rules for All Servers**的第二条规则：

<article class="message is-info">
  <div class="message-body">
      If RPC request or response contains term T > currentTerm: set currentTerm = T, convert to <strong>follower</strong> (§5.1)
  </div>
</article>


***Incorrect RPC handlers***

虽然已经有论文的**figure 2**，但要想正确处理RPCs的细节，还需要注意：

- “**reply false**”还暗示我们应该立刻返回，如果想要做其他的动作，需要将其放在前面，再观察逻辑是否合理；

- 收到**append**消息时，即使**prevLogIndex**越过本地日志最大长度，也应按**term**不匹配的情况处理而不是单独处理；

- 即使**append**消息未携带日志，仍然要检查：

<article class="message is-info">
  <div class="message-body">
      1. Reply false if log doesn’t contain an entry at <strong>prevLogIndex</strong>
      whose term matches <strong>prevLogIndex</strong> (§5.3)
  </div>
</article>



***Failure to follow The Rules***

- 在处理RPCs过程中，如果`commitIndex > lastApplied`，则要应用日志到本地状态机上。
- 确保定期或者**commitIndex**更新后检查`commitIndex > lastApplied`
- 如果`leader`发出**append**消息并且被拒绝，而且不是因为日志不一致，那么`leader`应该立刻下台
- `leader`不能将**commitIndex**更新到上个**term**的某个位置，因此我们需要特别检查`log[N].term == currentTerm`，具体参考图8；



***Term confusion***

正确处理RPCs中**term**的正确方式是，首先在`reply`中记录**term**，响应后将这个**term**和**currentTerm**做对比，如果不同就不处理，当且仅当这个**term**和**currentTerm**相等时才继续处理。另外，不要用`matchIndex = nextIndex - 1`或者 `matchIndex = len(log)`，因为它们可能在发送RPC之后已经被更新过了，正确的做法是使用`matchIndex = prevLogIndex + len(entries[])`。



***我的补充***

- `leader`一开始是不存在的，而**nextIndex**和**matchIndex**都和`leader`相关，因此，在`candidate`选举成功并转换成`leader`时，要让**nextIndex**赋初值，即`lastIndex+1`，表示不需要替换任何**entry**，这里还隐含了乐观者设定。

- 由于`leader`可能会对某个`follower`发送多个**append**消息，因此在处理回复时，即更新**matchIndex**和**nextIndex**时要按`args`而不是`leader`本身来判定。

- 每个`leader`只能**commit**当前**term**的**entry**（5.4）
