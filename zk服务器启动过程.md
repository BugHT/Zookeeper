## 服务端启动主要分为预启动和初始化过程。
### 单机模式启动
#### 预启动
1. 统一由QuorumPeerMain作为启动类，无论单机或集群，在zkServer.cmd和zkServer.sh中都配置了QuorumPeerMain作为启动入口类。
2. 解析配置文件zoo.cfg。zoo.cfg配置运行时的基本参数，如tickTime，dataDir，clientPort等参数。
3. 创建并启动历史文件俺清理器DatadirCleanupManager。对事务日志和快照文件进行定时清理。
4. 判断当前是集群模式还是单机模式启动，若是单机模式，则委托ZookeeperServerMain进行启动。
5. 在此进行配置文件zoo.cfg的解析。
6. 创建服务器实例zookeeperServer。zookeeper服务器首先会进行服务器实例的创建，然后对该服务器实例进行初始化，包括连接器，内存数据库，请求处理器等组件的初始化。

#### 初始化
1. 创建服务器统计器ServerStats。其是zk服务器运行时的统计器。
2. 创建zk数据管理器FileTxnSnapLog。其是zk上层服务器和底层数据存储之间的对接层，提供了一系列操作数据文件的接口，如事务日志文件和快照数据文件。zk根据zoo.cfg文件解析出的快照数据目录datadir和事物日志目录dataLogDir来创建FileTxnSnapLog
3. 设置服务器tickTime和会话超时时间限制。

4. 创建ServerCnxnFactory。通过配置系统性zookeeper.ServerCnxnFactory来指定使用zk自己实现的NIO还是使用Netty框架作为zk服务器网络连接工厂。
5. 初始化ServerCnxnFactory。zk会初始化Thread作为ServerCnxnFactory的主线程，然后再初始化NIO服务器。
6. 启动ServerCnxnFactory主线程，进入Thread的run方法，此时服务端还不能处理客户端请求。
7. 恢复本地数据，启动时，需要从本地快照数据文件和事务日志文件进行数据恢复。
8. 创建并启动会话管理器。zk会创建会话管理器SessionTracker进行会话管理。
9. 初始化zk的请求处理链。zk请求处理方式为责任链模式的实现，会有多个请求处理器依次处理一个客户端请求，在服务端启动时，会将这些请求处理器串联成一个请求处理链。
10. 注册JMX服务，zk会将服务器运行时的一些信息以JMX的方式暴露给外部。
11. 注册zk服务器实例，将zk服务器实例注册给ServerCnxnFactory，之后zk就可以对外提供服务。

### 集群模式启动

#### 预启动
1. 统一由QuorumPeerMain作为启动类。
2. 解析配置文件zoo.cfg
3. 创建并启动历史文件清理器DatadirCleanupManager。
4. 判读当前是集群模式还是单机模式的启动，在集群模式中，在zoo.cfg文件中配置了多个服务器地址，可以选择集群启动。

#### 初始化
1. 创建ServeCnxnFactory。
2. 初始化ServeCnxnFactory。
3. 创建zk数据管理器FileTxnSnapLog。
4. 创建QuorumPeer实例，Quorum是集群模式下特有的对象，是zk服务器实例zookeeperServer的托管者，QuorumPeer代表了集群中的一台机器，在运行期间，QuorumPeer会不断检测当前服务器实例的运行状态，同时根据情况发起Leader选举。
5. 创建内存数据库zkDataBase。zkDatabase负责管理zk的所有会话记录以及dataTree和事务日志的存储。
6. 初始化QuorumPeer。将核心组件如FileTxnSnapLog，serverCnxnFactory，ZKDatabase注册到QuorumPeer中，同时配置QuorumPeer的参数，如服务器列表地址，leader选举算法和会话超时时间限制等。
7. 恢复本地数据。
8. 启动ServerCnxnFactory主线程。

#### leader选举
1. 初始化Leader选举，集群模式特有，zk首先会根据自身的服务器id（sid），最新的zxid和当前的服务器epoch来生成一个初始化投票，在初始化过程中，每个服务器都会给自己投票，然后，根据zoo.cfg的配置，创建相应leader选举算法实现，zk提供了三种默认算法（leaderElection，authFastLeaderElection，FastLeaderElection），可通过zoo.cfg的electionAlg属性来指定，在初始化阶段，zk会创建leader选举所需的网络I/O层QuorumCnxManager，同时启动对Leader选举端口的监听，等待集群中其他服务器创建连接。
2. 注册JMX服务。
3. 检测当前服务器状态，运行期间，QuorumPeer会不断检测当前服务器状态，在正常情况下，zk服务器的状态在Looking，Leading，Following，Observing之间进行切换。在启动阶段，QuorumPeer的初始状态是Looking，因此开始Leader选举。
4. Leader选举。通过投票确定Leader，其余机器称为Follower和Observer。

#### Leader和Follower启动期交互过程
1. 创建Leader服务器和Follower服务器，完成Leader选举后，每个服务器会根据自己服务器的角色相应的服务器实例，并进入各自角色的主流程。
2. Leader服务器启动Follower接收器LearnerCnxAcceptor。运行期间，Leaader服务器需要和所有其余的服务器保持连接以确认集群的机器存活情况，LearnerCnx负责接收所有非Leader服务器的连接请求。
3. Leader服务器开始和Leader建立连接。所有learner会找到Leader服务器，并与其建立连接。
4. leader服务器创建LearnerHandler。leader接受到来自其他机器连接创建请求后，会创建一个LearnerHandler实例，每个LearnerHandler实例都对应一个leader与learner服务器之间的链接，其负责leader和learner之间几乎所有的消息通信和数据同步。
5. 向leader注册，learner完成和leader的连接后，会向Leader进行注册，即将learner服务器的基本信息，包括sid和zxid，发送给leader服务器
6. leader解析learner信息，计算新的epoch，leader接受到learner服务器基本信息后，会解析出该learner的sid和zxid，然后根据zxid解析出对应的epoch_of_learner，并和当前leader服务器的epoch_of_learner进行比较，如果该learner的epoch_of_learner更大，则更新leader的epoch_of_learner=epoch_of_learner+1。然后learnHandler进行等待，直到过半learner已经向leader进行了注册，同时更新了epoch_of_learner后，leader就可以确定当前集群的epoch了。
7. 发送leader状态，计算出新的epoch后，leader会将该信息以一个leaderinfo消息的形式发送给learner，并等待learner的响应。
8. learner发送ACK消息。learner接受到leaderinfo后，会解析出epoch和zxid，然后向leader反馈一个ackepoch响应。
9. 数据同步，leader收到leader收到learner的ackepoch后，即可进行数据同步。
10. 启动leader和learner服务器，当有过半的learner已经完成了数据同步，那么leader和learner服务器实例就可以启动了。

#### leader和follower启动
1. 创建启动会话管理器。
2. 初始化zk请求处理链，集群模式的每个处理器也会在启动阶段串联请求处理链。
3. 注册JMX服务。
