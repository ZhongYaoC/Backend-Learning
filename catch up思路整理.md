系统实际运行过程中，因网络或者软硬件设备故障，必然会导致服务器数据间的不同步问题，当集群规模很大时，硬件故障率数量被大样本放大，导致不同步问题更加严重。（PS，论文中此处可增加若干真实故障数据，具体可参考muduo书中数据）所以数据的追赶必不可少。

##### 已有的分布式共识算法中同步方案：

raft本质可理解为一个选举算法，追赶是由leader负责主动补发数据；

zab算法可分为五部分，其中同步阶段专用于解决数据不一致，且论文中的同步方案和修正版的zab实际算法也不同，实际版中传输的追赶数据更少，效率更高，是一种工程实践做法；

multi paxos算法可伴随正常的paxos流程补发数据，数据也以leader的数据为基准

（PS，以上皆可具体扩充，详细扩展）

以上三种算法中，均采用由中心方式，即有leader



##### 需要同步的场景：

（可以发现数据不一致的场景）考虑paxos流程和paxos lease（选举）流程



正常paxos流程中，P1和P2均可以发现数据不一致，或者说只要有数据通信，就可以借此时机检查实例号，从而发起追赶；而proposer、acceptor和learner都有各自的实例号，对应不同阶段或者说角色，当然这个实例号是由ReplicatedLog类统一管理，且只有acceptor的最新实例号做了持久化。

在通信期间，尤其是通信未进入到paxos前检查，刚好可以保证只有实例相同的节点之间才可以参与paxos。

所以借助P1、P2的通信期间，可以检查不同节点间实例，只要节点间的实例不同就说明需要数据同步，则由实例小的节点做数据追赶？数据追赶还是数据补发？master数据小了怎么办？

数据追赶和数据补发都要有！接收方发现自己的实例高，那就主动补发（但是此时说明本节点高，本节点应该当proposer才对啊，或者说本节点当master才对？？如果选举没问题，此情况不应该发生，或者说只有当proposer、acceptor部署在不同服务器而不是不同进程才会发生吧）；如果接收方实例低，那就由发送方（master）发送数据，而对于当前的接收方来说，需要发送请求追赶请求，让master知晓接收方需要数据；

主动补发：(给谁发，发哪个实例，对应的value)

请求追赶（需要对方补发）：（请求的实例， 发给谁）



需要注意的一点是：追赶或者补发数据的负责角色是learner，所以如果catch up的数据learner有（即本轮的），learner发生即可，否则需要去缓存或者数据库中get



那么以数据追赶为例，因为个人觉得在P1 P2阶段出现proposer的数据小，导致acceptor补发给proposer很难发生，一旦发生反而说明问题很大。

假设数据小的一方为B，master为A，A在P1/2阶段发生prepare或accept消息，B发现自身数据版本较低，于是主动发生请求追赶消息，并且不再参与接下来的paxos决议，A收到此消息后，从缓存或者数据库获取对应实例号的value，然后发送给B，B将此值存入缓存和数据库，然后将自身的实例号递增；此时还需要检查自身的实例是否最新（即数据是否追上了），如果没有还需继续以上流程；

那么，追赶何时结束呢？只有当追赶到当前正在决议的实例号时才算结束；此时考虑一种情况，追赶方刚刚追上正在决议的实例，此时这个正在决议的实例决议成功，开始发送learn消息，也就是说刚追上，就又出现了数据差距；

此时，如果仍按照之前的数据检测发送在P1/2阶段，那么效率较差，B节点只有在其他节点开始又下一个节点时才会感知数据差距，然后追赶，因为漏掉了learn消息阶段，所以在learn消息阶段也需要检查通信双方的数据差距。

如此设计之下，B节点的追赶消息就可以和其他节点的P2的accept消息几乎同时发出，而且追赶不需要等待半数以上，所以可以充分利用此时间差，赶在下一个实例的决议完成前实现数据持平。

PS：追赶到的值，只能给learner，如果赋给acceptor，可能会污染当前正在决议的实例



如何解决那些没有数据差距，但是未能参与决议的节点，也就是说节点C没能参与P1/2过程中，但是也仅仅是此轮未能参与（或者此轮其acceptor中承诺的值与其他节点不同），数据没有差距，那么收到learn消息时，是选择直接接收此值，还是请求追赶？keyspace中设计为追赶，甚至为此在追赶的报文中设计如果追赶的实例号为learner的实例号（即当前轮次），由learner负责value，而非缓存或者数据库，因为此时缓存和数据库可能还没有值！

直接用追赶的另一个原因是keysapce的设计中，正常的learn proposal报文并不包含value，此value值直接使用了接收节点本地的acceptor的value

![keyspace追赶1](F:\Markdown\研一上\图片\keyspace追赶1.png)

即在learn时，也采用类似两阶段提交？如果改为直接接受value会有影响吗？



现在问题来到，如何继续下一个实例的Propose（）？？？





TASK：

-[x] 整理原paxos中的所有类，包含变量名规范，函数注释等，重点paxosMsg
-[x] paxosMsg
-[x] 实例衔接问题
-[x] Instance对客户端提供Append接口
-[x] 问题：续租没发起，续租计时器启动有问题，已解决
-[ ] 追赶计时器超时函数
-[ ] 如何发起下一个实例
-[ ] 加入快照，将原先的发送文件类整合进instance
-[x] 结构体对齐问题，如paxos learn消息时：怎么发的，怎么收；解决





目前计划：

一旦发现实例差距，缺少方请求追赶，leader检查追赶的差距大小

1）如果发现差距在LogCache范围内，则利用paxos消息发送追赶；此操作计划交由CatchUper，在另一个loop(thread)中完成；

2）如果差距超出LogCache范围，则发送快照，而LogCache范围内的实例补发仍由1）方式完成；计划发送的快照范围是缺失的实例所在快照文件及之后所有快照；此操作交由FileTransport在另一个loop中完成；

3）继续接收新的chosen实例，不参与投票（这个可能会破坏现有的体系）



以上可能出现的问题：

1）可能存在的跨线程数据传输

2）快照接收时，新的实例仍在被chosen，所以LogCache仍在存入Push，所以可能导致快照接收时，另一个线程中操作最新的快照文件fd，而LogCache也可能会达到阈值并写入此文件，lseek混乱！（这是作为快照接收方角度考虑）

3）快照发送时将最新快照文件一并发送，最新快照文件如果未满，那么leader新学习的实例有可能会因为LogCache达到阈值，而同时写入到最新的快照文件，lseek问题仍在；（这是作为快照发送方角度考虑）



解决思路：

1）在发送快照之前，先将LogCache的数据全部存入快照，然后统一发送快照，取消快照时的LogCache的补齐，为实现此目的，需要修改LogCache的存入快照策略，目前为LogCache满才会存入快照文件，如果提前存入，会导致快照文件中实例数量错误！进而影响后续的所有。所有可以在存入时记录实例数量？？

2）慎重考虑补齐数据时，继续接收新chosen实例