# zookeeper 工作原理
## zookeeper 简介
ZooKeeper是一个开放源码的分布式应用程序协调服务，它包含一个简单的原语集，分布式应用程序可以基于它实现同步服务，配置维护和命名服务等。
## zookeeper 设计目的
- 最终一致性：client不论连接到哪个Server，展示给它都是同一个视图，这是zookeeper最重要的性能。
- 可靠性：具有简单、健壮、良好的性能，如果消息m被到一台服务器接受，那么它将被所有的服务器接受。
- 实时性：Zookeeper保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息。但由于网络延时等原因，Zookeeper不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用sync()接口。
- 等待无关（wait-free）：慢的或者失效的client不得干预快速的client的请求，使得每个client都能有效的等待。
- 原子性：更新只能成功或者失败，没有中间状态。
- 顺序性：包括全局有序和偏序两种：全局有序是指如果在一台服务器上消息a在消息b前发布，则在所有Server上消息a都将在消息b前被发布；偏序是指如果一个消息b在消息a后被同一个发送者发布，a必将排在b前面。
## zookeeper 角色
- 领导者（leader），负责进行投票的发起和决议，更新系统状态
- 学习者（learner），包括跟随者（follower）和观察者（observer）
  - follower用于接受客户端请求并想客户端返回结果，在选主过程中参与投票。
  - Observer可以接受客户端连接，将写请求转发给leader，但observer不参加投票过程，只同步leader的状态，observer的目的是为了扩展系统，提高读取速度
- 客户端（client），请求发起方

## zookeeper 数据模型
Zookeeper会维护一个具有层次关系的数据结构，它非常类似于一个标准的文件系统。
Zookeeper这种数据结构有如下这些特点：
1. 每个子目录项如NameService都被称作为znode，这个znode是被它所在的路径唯一标识，如Server1这个znode的标识为/NameService/Server1。
2. znode可以有子节点目录，并且每个znode可以存储数据，注意EPHEMERAL（临时的）类型的目录节点不能有子节点目录。
3. znode是有版本的（version），每个znode中存储的数据可以有多个版本，也就是一个访问路径中可以存储多份数据，version号自动增加。
4. znode的类型：
   - Persistent 节点，一旦被创建，便不会意外丢失，即使服务器全部重启也依然存在。每个 Persist 节点即可包含数据，也可包含子节点。
   - Ephemeral 节点，在创建它的客户端与服务器间的 Session 结束时自动被删除。服务器重启会导致 Session 结束，因此 Ephemeral 类型的 znode 此时也会自动删除。
   - Non-sequence 节点，多个客户端同时创建同一 Non-sequence 节点时，只有一个可创建成功，其它匀失败。并且创建出的节点名称与创建时指定的节点名完全一样。
   - Sequence 节点，创建出的节点名在指定的名称之后带有10位10进制数的序号。多个客户端创建同一名称的节点时，都能创建成功，只是序号不同。
5. znode可以被监控，包括这个目录节点中存储的数据的修改，子节点目录的变化等，一旦变化可以通知设置监控的客户端，这个是Zookeeper的核心特性，Zookeeper的很多功能都是基于这个特性实现的。
6. ZXID：每次对Zookeeper的状态的改变都会产生一个zxid（ZooKeeper Transaction Id），zxid是全局有序的，如果zxid1小于zxid2，则zxid1在zxid2之前发生。
## zookeeper session
如果Client因为Timeout和Zookeeper Server失去连接，client处在CONNECTING状态，会自动尝试再去连接Server，如果在session有效期内再次成功连接到某个Server，则回到CONNECTED状态。

注意：如果因为网络状态不好，client和Server失去联系，client会停留在当前状态，会尝试主动再次连接Zookeeper Server。client不能宣称自己的session expired，session expired是由Zookeeper Server来决定的，client可以选择自己主动关闭session。

## zookeeper watch
Zookeeper watch是一种监听通知机制。Zookeeper所有的读操作getData(), getChildren()和 exists()都可以设置监视(watch)，监视事件可以理解为一次性的触发器

Watch的三个关键点：
1. 一次性触发。当设置监视的数据发生改变时，该监视事件会被发送到客户端，例如，如果客户端调用了getData("/znode1", true) 并且稍后 /znode1 节点上的数据发生了改变或者被删除了，客户端将会获取到 /znode1 发生变化的监视事件，而如果 /znode1 再一次发生了变化，除非客户端再次对/znode1 设置监视，否则客户端不会收到事件通知。
2. 发送至客户端。Zookeeper客户端和服务端是通过 socket 进行通信的，由于网络存在故障，所以监视事件很有可能不会成功地到达客户端，监视事件是异步发送至监视者的，Zookeeper 本身提供了顺序保证(ordering guarantee)：即客户端只有首先看到了监视事件后，才会感知到它所设置监视的znode发生了变化。网络延迟或者其他因素可能导致不同的客户端在不同的时刻感知某一监视事件，但是不同的客户端所看到的一切具有一致的顺序。
3. 被设置 watch 的数据。这意味着znode节点本身具有不同的改变方式。你也可以想象 Zookeeper 维护了两条监视链表：数据监视和子节点监视(data watches and child watches) getData() 和exists()设置数据监视，getChildren()设置子节点监视。或者你也可以想象 Zookeeper 设置的不同监视返回不同的数据，getData() 和 exists() 返回znode节点的相关信息，而getChildren() 返回子节点列表。因此，setData() 会触发设置在某一节点上所设置的数据监视（假定数据设置成功），而一次成功的create() 操作则会出发当前节点上所设置的数据监视以及父节点的子节点监视。一次成功的 delete操作将会触发当前节点的数据监视和子节点监视事件，同时也会触发该节点父节点的child watch。

Zookeeper 中的监视是轻量级的，因此容易设置、维护和分发。当客户端与 Zookeeper 服务器失去联系时，客户端并不会收到监视事件的通知，只有当客户端重新连接后，若在必要的情况下，以前注册的监视会重新被注册并触发，对于开发人员来说这通常是透明的。只有一种情况会导致监视事件的丢失，即：通过exists()设置了某个znode节点的监视，但是如果某个客户端在此znode节点被创建和删除的时间间隔内与zookeeper服务器失去了联系，该客户端即使稍后重新连接 zookeeper服务器后也得不到事件通知。

## zookeeper 读写机制
1. Zookeeper是一个由多个server组成的集群
2. 一个leader，多个follower
3. 每个server保存一份数据副本
4. 全局数据一致
5. 分布式读写
6. 更新请求转发，由leader实施

## zookeeper 保证
1. 更新请求顺序进行，来自同一个client的更新请求按其发送顺序依次执行
2. 数据更新原子性，一次数据更新要么成功，要么失败
3. 全局唯一数据视图，client无论连接到哪个server，数据视图都是一致的
4. 实时性，在一定事件范围内，client能读到最新数据

## zookeeper 工作原理
在zookeeper的集群中，各个节点共有下面3种角色和4种状态：
- 角色：leader,follower,observer
- 状态：leading,following,observing,looking

Zookeeper的核心是原子广播，这个机制保证了各个Server之间的同步。实现这个机制的协议叫做Zab协议（ZooKeeper Atomic Broadcast protocol）。Zab协议有两种模式，它们分别是恢复模式（Recovery选主）和广播模式（Broadcast同步）。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。

为了保证事务的顺序一致性，zookeeper采用了递增的事务id号（zxid）来标识事务。所有的提议（proposal）都在被提出的时候加上了zxid。实现中zxid是一个64位的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个新的epoch，标识当前属于那个leader的统治时期。低32位用于递增计数。

每个Server在工作过程中有4种状态：

LOOKING：当前Server不知道leader是谁，正在搜寻。

LEADING：当前Server即为选举出来的leader。

FOLLOWING：leader已经选举出来，当前Server与之同步。

OBSERVING：observer的行为在大多数情况下与follower完全一致，但是他们不参加选举和投票，而仅仅接受(observing)选举和投票的结果。

## zookeeper 节点数据操作流程
1. 在Client向Follwer发出一个写的请求
2. Follwer把请求发送给Leader
3. Leader接收到以后开始发起投票并通知Follwer进行投票
4. Follwer把投票结果发送给Leader
5. Leader将结果汇总后如果需要写入，则开始写入同时把写入操作通知给Leader，然后commit;
6. Follwer把请求结果返回给Client

## Leader 工作流程
Leader主要有三个功能：
1. 恢复数据；
2. 维持与follower的心跳，接收follower请求并判断follower的请求消息类型；
3. follower的消息类型主要有PING消息、REQUEST消息、ACK消息、REVALIDATE消息，根据不同的消息类型，进行不同的处理。

PING消息是指follower的心跳信息；REQUEST消息是follower发送的提议信息，包括写请求及同步请求；

ACK消息是follower的对提议的回复，超过半数的follower通过，则commit该提议；

REVALIDATE消息是用来延长SESSION有效时间。

## Follower 工作流程
Follower主要有四个功能：
1. 向Leader发送请求（PING消息、REQUEST消息、ACK消息、REVALIDATE消息）；

2. 接收Leader消息并进行处理；

3. 接收Client的请求，如果为写请求，发送给Leader进行投票；

4. 返回Client结果。

Follower的消息循环处理如下几种来自Leader的消息：
1. PING消息：心跳消息

2. PROPOSAL消息：Leader发起的提案，要求Follower投票

3. COMMIT消息：服务器端最新一次提案的信息

4. UPTODATE消息：表明同步完成

5. REVALIDATE消息：根据Leader的REVALIDATE结果，关闭待revalidate的session还是允许其接受消息

6. SYNC消息：返回SYNC结果到客户端，这个消息最初由客户端发起，用来强制得到最新的更新。
## Observer　
- Zookeeper需保证高可用和强一致性；
- 为了支持更多的客户端，需要增加更多Server；
- Server增多，投票阶段延迟增大，影响性能；
- 权衡伸缩性和高吞吐率，引入Observer
- Observer不参与投票；
- Observers接受客户端的连接，并将写请求转发给leader节点；
- 加入更多Observer节点，提高伸缩性，同时不影响吞吐率

## zxid
ZooKeeper状态的每一次改变, 都对应着一个递增的Transaction id, 该id称为zxid. 由于zxid的递增性质, 如果zxid1小于zxid2, 那么zxid1肯定先于zxid2发生.

创建任意节点, 或者更新任意节点的数据, 或者删除任意节点, 都会导致Zookeeper状态发生改变, 从而导致zxid的值增加.

# zookeeper 应用场景
## 统一命名服务
分布式应用中，通常需要有一套完整的命名规则，既能够产生唯一的名称又便于人识别和记住，通常情况下用树形的名称结构是一个理想的选择，树形的名称结构是一个有层次的目录结构，既对人友好又不会重复。

Name Service 是 Zookeeper 内置的功能，只要调用 Zookeeper 的 API 就能实现。

## 配置管理
将配置信息保存在 Zookeeper 的某个目录节点中，然后将所有需要修改的应用机器监控配置信息的状态，一旦配置信息发生变化，每台应用机器就会收到Zookeeper的通知，然后从 Zookeeper 获取新的配置信息应用到系统中。

## 集群管理
Zookeeper 能够很容易的实现集群管理的功能，如有多台 Server 组成一个服务集群，那么必须要一个“总管”知道当前集群中每台机器的服务状态，一旦有机器不能提供服务，集群中其它集群必须知道，从而做出调整重新分配服务策略。同样当增加集群的服务能力时，就会增加一台或多台 Server，同样也必须让“总管”知道。

Zookeeper 不仅能够维护当前的集群中机器的服务状态，而且能够选出一个“总管”，让这个总管来管理集群，这就是 Zookeeper 的另一个功能 LeaderElection

规定编号最小的为master,所以当我们对SERVERS节点做监控的时候，得到服务器列表，只要所有集群机器逻辑认为最小编号节点为master，那么master就被选出，而这个master宕机的时候，相应的znode会消失，然后新的服务器列表就被推送到客户端，然后每个节点逻辑认为最小编号节点为master，这样就做到动态master选举

## 分布式锁
完全分布式锁是全局同步的，这意味着在任何时刻没有两个客户端会同时认为它们都拥有相同的锁，使用 Zookeeper 可以实现分布式锁，需要首先定义一个锁节点（lock root node）。

需要获得锁的客户端按照以下步骤来获取锁：
1. 保证锁节点（lock root node）这个父根节点的存在，这个节点是每个要获取lock客户端共用的，这个节点是PERSISTENT的。
2. 第一次需要创建本客户端要获取lock的节点，调用 create( )，并设置 节点为EPHEMERAL_SEQUENTIAL类型，表示该节点为临时的和顺序的。如果获取锁的节点挂掉，则该节点自动失效，可以让其他节点获取锁。
3. 在父锁节点（lock root node）上调用 getChildren( ) ，不需要设置监视标志。 (为了避免“羊群效应”).
4. 按照Fair竞争的原则，将步骤3中的子节点（要获取锁的节点）按照节点顺序的大小做排序，取出编号最小的一个节点做为lock的owner，判断自己的节点id是否就为owner id，如果是则返回，lock成功。如果不是则调用 exists( )监听比自己小的前一位的id，关注它锁释放的操作（也就是exist watch）。
5. 如果第4步监听exist的watch被触发，则继续按4中的原则判断自己是否能获取到lock。

释放锁：需要释放锁的客户端只需要删除在第2步中创建的节点即可。

注意事项：
一个节点的删除只会导致一个客户端被唤醒，因为每个节点只被一个客户端watch，这避免了“羊群效应”。
## 分布式队列
分布式队列是通用的数据结构，为了在 Zookeeper 中实现分布式队列，首先需要指定一个 Znode 节点作为队列节点（queue node）， 各个分布式客户端通过调用 create() 函数向队列中放入数据，调用create()时节点路径名带"qn-"结尾，并设置顺序（sequence）节点标志。 由于设置了节点的顺序标志，新的路径名具有以下字符串模式："_path-to-queue-node_/qn-X"，X 是唯一自增号。需要从队列中获取数据/移除数据的客户端首先调用 getChildren() 函数，有数据则获取（获取数据后可以删除也可以不删），没有则在队列节点（queue node）上将 watch 设置为 true，等待触发并处理最小序号的节点（即从序号最小的节点中取数据）。
实现步骤基本如下：
前提：需要一个队列root节点dir
入队：使用create()创建节点，将共享数据data放在该节点上，节点类型为PERSISTENT_SEQUENTIAL，永久顺序性的（也可以设置为临时的，看需求）。
出队：因为队列可能为空，2种方式处理：一种如果为空则wait等待，一种返回异常。
等待方式：这里使用了CountDownLatch的等待和Watcher的通知机制，使用了TreeMap的排序获取节点顺序最小的数据（FIFO）。
抛出异常：getChildren()获取队列数据时，如果size==0则抛出异常。