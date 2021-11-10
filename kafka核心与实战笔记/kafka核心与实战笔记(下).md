## 二十四、请求是怎么被处理的？

1.Apache kafka自己定义了一组请求协议，用于实现各种各样的交互操作。比如PRODUCE请求用于生产消息，FETCH请求用于消费消息，METADATA是用于请求Kafka集群元数据信息

2.Kafka是如何处理请求的？kafka使用的是reactor模式(多线程模式)

![image-20211015141638511](pic\image-20211015141638511.png)

3.Kafka的Broker端有个SocketServer组件，类似于Reactor模式中的Dispatcher，它也有一个Acceptor线程和一个工作线程池(网络线程池)。kafka提供了Broker端参数num.network.threads,用于调整该网络线程池的线程数，默认值为3，标识每台Broker启动时会创建3个网络线程，专门处理客户端发送的请求。

![image-20211015143227515](pic\image-20211015143227515.png)

4.当网络线程拿到请求后，它不是自己处理，而是将请求放入到一个共享请求队列中，Broker端还有个IO线程池，负责从该队列中取出请求，执行真正的处理；如果是PRODUCE生产请求，则将消息写入到底层的磁盘日志中；如果是FETCH请求，则从磁盘或者页缓存中读取消息。IO线程池中的线程才是执行请求逻辑的线程。Broker端参数num.io.threads控制了这个线程池中的线程数。目前该参数默认值是8，表示每台Broker启动后自动创建8个IO线程处理请求，如果机器上上的CPU资源非常充裕，你可以调大该参数。

5.请求队列是所有网络线程共有的，响应队列却是每个网络线程专属的。

6.Purgatory组件，这是kafka中的"炼狱"组件。它是用来缓存延时请求(Delayed Request)的。所谓延时请求，就是一时那些未满足条件不能立刻处理的请求。

7.kafka社区把PRODUCE和FETCH这类请求称为数据类请求，把LeaderAndIsr、stopReplica这类请求称为控制类请求。

8.控制类请求有这样的一种能力：它可以直接令数据类请求失效。

9.Apache kafka2.3版本将数据类和请求类请求完全分离开来了。



## 二十五、消费者组重平衡全流程解析

1.Rebalance的作用是让组内所有的消费者实例就消费哪些主题分区达成一致

2.Rebalance需要借助Kafka Broker端的coordinator组件，在这个组件的帮助下完成整个消费者组的分区重分配。

3.Rebalance的触发与通知：1.组成员数量发生变化；2.订阅主题数量发生变化；3.订阅主题的分区数发生变化

4.Rebalance过程是如何通知到其他消费者实例的，是通过customer的心跳线程。

5.Kafka customer需要定期的发送心跳请求到Broker端的协调者，表明自己的状态。kafka 0.10.1版本之前，发送心跳请求是在customer主线程完成的，也就是KafkaCustomer.poll方法的那个线程。这个设计存在诸多弊病，由于发送心跳请求与消息处理逻辑也在主线程中，所以一旦消息处理消耗了过长时间，可能会导致协调者错误认为消费者已死。

6.Rebalance的通知机制正是通过心跳线程来完成。当Coordinator决定开启新一轮重平衡，它会将"REBALANCE_IN_PROCRSS"封装进心跳请求的响应中，以此让customer来得到Rebalance的通知。

7.heartbeat.interval.ms的真实用途。从字面上来看，它就是设置了心跳的间隔时间，但这个参数的真正作用是控制Rebalance通知的频率。如果该值设置的小一些，Customer就能更快感知到Rebalance已经开始了。

8.REBALANCE一旦开启，Broker端的coordinator组件就要开始忙了，主要涉及到控制CustomerGroup的状态流转。Kafka设计了一套消费者组状态机(State Machine),来帮助coordinator完成整个Rebalance流程。

9.kafka为customerGroup定义了五种状态，它们分别是Empty、Dead、PreparingRebalance、CompletingRebalance、Stable。

![screenshot-20211018-110630](pic\screenshot-20211018-110630.png)

10.Kafka CustomerGroup的状态流转。

![screenshot-20211018-111050](pic\screenshot-20211018-111050.png)

一个消费组最开始是Empty状态，当Rebalance过程开启后，它会被置于PreparingRebalance状态等待成员加入，之后变更到CompletingRebalance状态等待分配方案，最后流转到Stable状态完成Rebalance。

当有新成员加入或已有成员退出时，消费组的状态从stable直接跳到PreparingRebalance状态，此时，所有现存成员就必须重新申请加入组。当所有成员都退出组后，消费组状态变更为Empty。Kafka会定期自动删除过期位移的条件就是，组要处于Empty状态。因此，如果你的消费组组停掉了很长的时间(超过七天)，那么kafka很可能就把该组的位移数据删除了，这就是kafka在尝试定期删除过期位移。只有状态为Empty的组，才会执行过期位移删除操作。

11.Customer端Rebalance流程。

在Customer端，Rebalance有两个步骤：加入组、等待Leader Consumer分配方案。这两个步骤分别对应两类特定的请求：JoinGroup请求和SyncGroup请求。当组内成员加入组时，它会向coordinator发送JoinGroup请求。在该请求中，每个成员都要将自己订阅的主题上传，这样coordinator就可以收到所有消费组成员的订阅信息了。然后coordinator会从这些成员中选择一个担任这个CustomerGroup的Leader。

通常来说，第一个发送JoinGroup请求的成员自动成为领导者，Leader Consumer的任务是收集所有成员的订阅信息，然后根据这些信息，指定具体的分区分配方案。选出Leader Consumer之后，coordinator会将CustomerGroup的订阅信息发送到JoinGroup的请求体中，Leader Consumer收到后做出分配方案，发送SyncGroup请求。这时，Leader Consumer和其他Customer都会发送SyncGroup请求，coordinator收到分配方案后，统一以SyncGroup的响应分发给所有成员，这样所有Customer都知道自己该消费哪个分区了。

12.Broker端Rebalance流程

分析coordinator处理Rebalance的全流程，从以下场景来进行分析：新成员加入组、组成员主动离组、组成员奔溃离组、组成员提交位移。

**场景一：**新成员入组

新成员入组是指ConsumerGroup处于Stable状态后，有新成员加入。如果是全新启动一个ConsumerGroup，kafka会有一些小优化，流程略微不同。

当coordinator收到新的JoinGroup请求之后，它会通过心跳请求响应的方式通知组内现有的所有成员，强制它们开始新一轮的ReBalance。



**场景二：**组成员主动离组

何谓主动离组？就是Consumer实例所在的线程或者进程调用close()方法主动通知coordinator它要退出。这个场景涉及到了第三类请求：LeaveGroup请求。



**场景三：**组成员崩溃离组

崩溃离组是指Consumer实例出现严重故障，突然宕机导致的离组。它和主动离组是有区别的，因为后者是主动发起的离组，coordinator能马上感知并处理。但崩溃是被动的，coordinator通常需要等待一段时间才能感知到，这段时间一般是由session.timeout.ms控制的。也就是说，kafka一般不会超过这个session.timeout.ms就能感知这个崩溃。



**场景四**: 重平衡时coordinator对组内成员提交位移的处理

正常情况下，每个组内成员都会定期汇报位移给coordinator。当ReBalance开启时，协调者会给予成员一段缓冲时间，要求每个成员必须在这段时间内快速地上报自己的位移信息，然后再开启正常的JoinGroup、SyncGroup请求发送。



## 二十六、你一定不能错过的Kafka控制器

1.控制器组件(Controller),是Apache kafka的核心组件。它主要的作用是在Apache Zookeeper的帮助下管理和协调整个kafka集群。

2.控制器是重度依赖Zookeeper的。

3.Apache Zookeeper是一个提供高可靠性的分布式协调服务框架。它使用的数据模型类似于文件系统的树形结构，根目录也是以" / "开始。该结构上的每个节点被称为Znode，用来保存一些元数据协调信息。Znode可以分为持久性Znode和临时Znode。持久性Znode不会因为Zookeeper集群重启而消失，而临时Znode则与创建该Znode的Zookeeper回话绑定，一旦会话结束，该节点就被删除。

4.Zookeeper赋予客户端监控Znode变更的能力，即所谓的Watch通知功能。一旦Znode节点被创建、删除，子节点数量发生变化，抑或是Znode所存的数据本身变更，Zookeeper会通过节点变更监听器(ChangeHandle)的方式显式通知客户端。

5.依赖于这些功能，Zookeeper常被用来实现集群成员管理、分布式锁、领导者选举等功能。Kafka Controller大量使用Watch功能实现对集群的协调管理。

![screenshot-20211018-164348](pic\screenshot-20211018-164348.png)

6.Controller是如何被选举出来的？在Broker启动时，会尝试去ZK中创建/controller节点。Kafka当前选举控制器的规则是：第一个成功创建/controller节点的Broker会被指定为控制器。

7.controller是做什么的呢？

一、主题管理(创建、删除、增加分区)

eg：kafka-topics命令

二、分区重分配

eg：kafka-reassign-partitions命令

三、Preferred领导者选举

四、集群成员管理(新增Broker、Broker主动关闭、Broker宕机)

包括自动检测新增Broker、Broker主动关闭及被动宕机。这种检测机制是依赖于前面提到的Watch功能和Zookeeper临时节点组合实现的。Controller会利用Watch机制检查ZK的/brokers/ids节点下的子节点数目变更。当有新的Broker启动时，它会在/brokers下创建专属的znode节点。一旦创建完毕，ZK会通过Watch机制将消息通知推送给Controller，这样，控制器就能自动地感知到这个变化，进而开启后续的新增Broker作业。侦测Broker存活性则是依赖于刚刚提到的另外一个机制：临时节点。每个Broker启动后，会在/brokers/ids下创建一个临时Znode。当broker宕机或者主动关闭后，该Broker与ZK的会话结束，这个Znode被删除。同理，ZK的Watch机制将这一变更推送给Controller，这样Controller就知道有Broker关闭或宕机了，从而进行后续处理。

五、数据服务

控制器保存了什么数据？

当前存活的broker列表、正在关闭中broker列表、正在进行preferred leader选举的分区、分配给每个分区的副本列表、topic列表、每个分区的leader和ISR信息、移除某个topic的所有信息、获取某个broker上的所有分区、某组broker上的所有副本、某个topic上的所有副本、某个topic的所有分区、当前存货的所有副本、正在进行重分配的分区列表、某组分区下的所有副本。

8.Controller故障转移(FailOver)

在一个kafka集群中，任意时刻只能存在一个Controller，这时候就存在单点失效问题(Single Point Of Failure)的风险。kafka通过FailOver为控制器提供故障转移功能。故障转移指的是，当运行中的控制器突然宕机或者意外终止时，kafka能够快速地感知到，并立即启用备用控制器来代替之前失败的控制器。

9.Controller内部设计原理

在kafka 0.11版本之前，Controller是多线程设计，这些线程还会访问controller共享的数据资源，为了解决数据一致性问题，kafka使用了大量了同步机制，从而拖慢了整个控制器的处理速度。鉴于这些原因，社区0.11版本重构了controller的设计，修改为了单线程加事件队列的方案。

![screenshot-20211018-180051](pic\screenshot-20211018-180051.png)

9.1、社区引入了一个事件处理线程，统一处理各种controller事件，然后控制器将原来执行的操作全部建模程一个个独立的时间，发送到专属的事件队列中，供次线程消费。

9.2、将之前同步操作Zk全部修改为异步操作。当有大量主题分区发生变更时，ZK容易成为系统的瓶颈。

9.3、当你觉得控制器组件出现问题时，比如主题无法删除了，或者重分区hang住了，你不用重启kafka Broker或控制器，有一个简单快速的方式是，去ZK中手动删除/controller节点。具体命令是rmr /controller。这样做的好处是，既可以引发控制器的重选举，又可以避免重启Broker导致的消息处理中断。



二十七、关于高水位和Leader Epoch的讨论

1.高水位(High WaterMark)，Leader Epoch是Apache kafka在0.11版本中推出的，主要是用来弥补高水位机制的一些缺陷。

2.什么是高水位？或者说什么是水位？水位一次多用于流式处理领域。教科书中的定义：在时刻T，任意创建时间(Event Time)为T',且T'<=T的所有事件都已经到达或被观测到，那么T就被定义为水位。

![image-20211019094718636](pic\image-20211019094718636.png)

用一张图来理解，"Complete"的蓝色部分代表已完成的共奏，标注"In-fight"的红色部分代表正在进行中的工作，两者的边界就是水位线。

3.在Kafka中，高水位(HW)使用消息唯一来表征的，低水位(LW)，它是删除消息相关联的概念。

4.在kafka中高水位的作用：定义消息可见性，即用来标识分区下的哪些消息是可以被消费者消费的；帮助kafka完成副本同步。

5.位移值等于高水位的消息也属于未提交消息。也就是说，高水位上的消息是不能被消费者消费的。

![image-20211019100034164](pic\image-20211019100034164.png)

6.上图中还有一个日志末端位移的概念，即Log End Offset(LEO)。它表示副本写入下一条消息的位移值。注意，上图中的数字15是虚线，这就说明，这个副本当前只有15条消息，位移似乎从0-14，下一条新消息的位移是15.介于高水位与LEO之间的消息就似乎未提交消息。这也说明一个问题：高水位不会超过LEO值。

7.高水位和LEO是副本对象的两个重要属性。Kafka所有副本都有对应的高水位和LEO值，而不仅是Leader副本。只不过Leader副本比较图书，kafka使用Leader副本高水位来定义所在分区的高水位。分区的高水位就是其Leader副本的高水位。

8.高水位更新机制。

现在我们知道了每个副本对象都保存了一组高水位值和LEO值，但实际上，在Leader副本所在的Broker上，还保存了其他Follwer副本的LEO值。

![image-20211019101230513](pic\image-20211019101230513.png)

在这张图中，我们可以看到，Broker0上保存了某分区的Leader副本和所有Follower副本的LEO值，而Broker1仅仅保存了该分区的某个Follower副本。kafka把Broker0上保存的这些Follower副本称为远程副本。Kafka副本机制在运行机制过程中，会更新Broker1上Follwer副本的高水位和LEO值，同时也会更新Broker0上的Leader副本HW和LEO以及所有远程副本的LEO，但不会更新远程副本的HW，也就是灰色部分。为什么需要在Broker0上保存这些远程副本呢？其实，它们的主要作用是，帮助Leader副本确定其HW，也就是分区HW。

9.只有搞清楚了更新机制，才能开始讨论Kafka副本机制的原理，以及它是如何使用HW来执行副本消息同步的。

![image-20211019102348730](pic\image-20211019102348730.png)

10.Leader Epoch

依托HW，kafka既定了消息的对外可见性，又实现了副本同步机制，不过仍然存在部分问题。可能会出现的Follwer副本因为重启，没有进行数据同步，等到同步时，leader副本又宕机，这时选举副本B为Leader，这时候就会导致数据丢失。



## 二十八、主题管理知多少？

1.kafka提供了自带的kafka-topics脚本，用于帮助用户创建主题。

2.查询所有主题的列表

eg:   bin/kafka-topics.sh --bootstrap-server broker_host:port --list

3.修改主题分区(目前kafka中不支持减少某个主题的分区数)

4.修改主题级别参数

5.变更副本数

6.修改主题限速

7.主题分区迁移



## 二十九、Kafka动态配置了解下？

1. 1.1.0版本添加了动态Broker参数。所谓动态，就是指修改参数值后，无需重启Broker就能立即生效，之前在server.properties中配置的参数则称为静态参数。
2. 使用场景：动态调整Broker端各种线程池大小，实时应对突发流量；动态调整Broker端连接信息或安全配置信息；动态更新SSL KeyStore有效期
3. 动态调整Broker端Compact操作性能
4. 实时变更JMX指标收集器(JMX Metrics Reporter)



## 三十、怎么重设消费者组位移？

1.为什么要重设消费者组位移？

kafka和传统的消息引擎在设计上有很大区别，其中一个就是kafka的消费者读取消息时可以重演的。反之，RabbitMQ和ActiveMQ就不支持，它们的消费都是破坏式的。

2.如何在传统消息引擎与kafka中做选择？如果在你的场景中，消息处理逻辑非常复杂，处理代价很高，同时又不关心消息之间的顺序，那么传统的消息中间件是比较合适的；反之，如果需要较高的吞吐量，但每条消息的处理时间很短，同时又很在意消息的顺序，那么就选择Kafka。

3.重设位移策略

不管是哪种设置方式，重设位移大致可以从两个维度来进行。

1）位移维度。这是指根据位移值来重设。也就是说，直接把消费者的位移值重设程我们给定的位移值

2）时间维度。我们可以给定一个时间，让消费者把位移调整成大于该时间的最小位移；也可以给出一段时间间隔，比如30分钟前，然后让消费者直接将位移调回30分钟前的位移值

![image-20211019143908472](pic\image-20211019143908472.png)

Earliest策略表示将位移调整到主题当前最早位移处。这个最早位移不一定就是0，因为在生产环境中，很久远的消息会被kafka自动删除，所以当前最早位移很可能是一个大于0的值。如果你想要重新消费主题的所有消息，那可以使用Earliest策略。

Latest策略表示把位移重设为最新末端位移。如果你想跳过所有历史消息，打算从最新的消息处开始消费的话，可以使用Latest策略。

Current策略表示将位移调整成消费者当前提交的最新位移。

Specified-Offset策略则是比较通用的策略，表示消费者把位移值调整到你指定的位移处。这个策略的典型使用场景是，消费者程序在处理某条错误消息时，你可以手动地"跳过"此消息的处理。

如果说Specified-Offset策略要求你指定位移的绝对数值的话。那么Shift-By-N策略指定的就是位移的相对数值。比如你想把位移重设为当前位移的前100条位移处，此时你需要指定N为-100。

DateTime允许你指定一个时间，然后将位移重置到该时间之后的最早位移处。常见的使用场景是，你想重新消费昨天的数据，那么你可以使用该策略重设位移到昨天0点。

Duration策略则是指给定相对的时间间隔，然后将位移调整到举例当前给定时间间隔的位移处，具体格式是PnDTnGnMns。

目前，重设消费者位移的方式有两种：通过消费者API来实现，通过Kafka-consumer-group命令行来实现(kafka 0.11版本之后支持)。



## 三十一、常见工具脚本大汇总

1.在0.10.2.0之前，kafka是单向兼容的，即高版本的broker能够处理低版本Client的请求，反过来不行。自0.10.2.0版本开始，kafka正式支持双向兼容，也就是说，低版本的Broker也能处理高版本的Client请求了。

2.kafka-console-consumer和kafka-console-producer

3.kafka-producer-perf-test和kafka-consumer-perf-test，它们分别是生产者和消费者的性能测试工具

4.kafka-consumer-groups命令，重设消费者位移

5.kafka-delegation-tokens，它是管理delegation Token的。基于Delegation Token的认证是一种轻量级的认证机制，补充了现有的SASL认证机制。

6.kafka-delete-records脚本用于删除kafka的分区消息。鉴于kafka有自己的自动消息删除策略，这个脚本的实际出场率并不高。

7.kafka-dump-log脚本是非常实用的脚本。它能查看kafka消息文件的内容，包括各种元数据信息，甚至是消息体本身。

8.kafka-log-dirs是比较新的脚本，可以帮助查看各个Broker上的各个日志路径的磁盘占用情况。

9.kafka-mirror-maker是帮助你实现kafka集群间的消息同步

10.kafka-preferred-replica-election是执行preferred Leader选举的。它可以为指定的主题执行"换Leader"的操作。

11.kafka-reassign-partitions用于执行分区副本迁移以及副本文件路径迁移。

12.kafka-topics 用于主题管理操作。



