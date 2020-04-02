### TDPoS共识算法

超级链支持的共识算法有pow、single和自研的TDPoS共识算法。

pow算法是所有矿工一起出块，通过解决一个特定的密码学问题从而达成共识。

Single也称为授权共识，在一个区块链网络中授权固定的address周期出块。

这里重点学习TDPoS共识算法，TDPoS可简单理解为改进的DPoS+BFT算法。 区块链上所有节点都可以对出块节点进行提案和投票，最后票数最高的指定个数的节点被选为出块节点，这些节点在一轮周期内协同出块。

#### Chained-BFT

超级链chained-BFT模块实现了chained HotStuff共识算法，该算法是对BFT算法的改进，普通的验证节点只需要与主节点进行通信，通信复杂度大大降低。

hotstuff算法是基于view的共识算法，view表示一个共识单元，一个view中存在一个主节点进行出块，其他验证节点进行投票，投票成功后，更新节点状态。算法的实现主要在chainedbft/smr模块，smr记录了节点目前的状态，包含三个重要成员：proposalQC，generateQC，lockedQC。当前提案的区块记录在proposalQC，收到足够多投票后将proposalQC移动到generateQC，同时generateQC移到lockedQC。

![handleVote](/Users/wanghongyan01/Desktop/handleVote.png)

在这里，经过三个块的确认便可以达成共识。

![img](https://gexin1023.github.io/pic/chained_hotstuff.png)

每个view的主要流程如下：

- processPropsal：将打包好的区块广播给其他验证节点。
- handleReceivedProposal：其他验证节点收到主节点的提案后，进行投票，并更改本地smr状态。
- handleReceivedVoteMsg：主节点收到投票信息后进行验证，如果投票数大于2/3的验证节点，则更改本地状态。

#### XPoS

XPoS是一种改进的DPoS算法。每一轮的出块周期都会选定若干个验证节点，这些节点在这个周期内轮流出块，周期结束后，下一轮由另一组节点协同出块。每一轮的出块由chained-hotstuff算法来保证安全性。DPoS主要关注如何选取验证节点和如何进行轮值的问题。

##### 验证节点选定：

提案：区块链上的所有节点都可以对任何节点(包括自身)进行验证者的提案，被提案的节点成为候选节点，候选节点接受其他节点的投票，有可能成为验证节点。这里参见runNominateCandidate()函数。

投票：节点可以对候选节点投票，支持一票多投。这里参见runVote()函数。

验票：每轮产生的第一个区块会自动触发检票交易，该交易对提案者的投票数进行统计，得票数最多的若干节点被选做是下一轮的验证节点。这里参见runCheckValidater()函数。

##### 轮换下一个出块节点

节点会持续的判定自己是否为出块节点，具体逻辑参见CompeteMaster()函数。主要利用时间调度算法来判定当前的term、轮值的位置pos和该矿工将要出的blockpos。如果当前矿工已经完成轮值，则需要转换view，需要BFT操作ProcessNewView()函数。这里有三种情况，

- 如果当前节点是新的出块节点，则等待接收2/3以上节点发送newViewMsg，参考addViewMsg()；

- 如果当前节点是普通的验证节点，则发送newViewMsg给出块节点；

- 如果当前节点是前一个出块节点，发送newViewMsg和它的QuorumCert投票签名给新的出块节点。

具体轮换流程参照上面说到的hotStuff共识流程。

##### term切换

切换到下一轮本质上也是出块节点的更换，只是这里需要更新验证节点的集合，主要参考getTermProposer()和UpdateValidateSets()函数。

![TDPoS共识算法](/Users/wanghongyan01/Desktop/超级链/学习总结/TDPoS共识算法.png)

#### 时间调度算法

超级链对于每轮的验证节点数目、每轮每个矿工的出块数、出块的时间间隔、轮换view的时间间隔、轮换term的时间间隔都做了规定，这样就可以根据任意时刻的系统时间和共识初始的时间，计算出当前的term、当前轮值的矿工以及矿工将要出的块数。

![时间调度算法](/Users/wanghongyan01/Desktop/超级链/学习总结/时间调度算法.png)



2020年4月2日 @Hongyan Wang