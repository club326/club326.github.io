# NSQ维护文档
&#160; &#160; &#160; &#160;MQ主要负责在一些微服务或者业务之间进行消息投递；一般情况下，如果直接调用业务之间的接口，会导致业务之间存在过多的依赖从而导致耦合性过强；因此，在业务重构过程中，MQ是重要的基础组件。
## 1.MQ核心概念
### 1.1主题
&#160; &#160; &#160; &#160;主题是用来发布消息的一个概念，发布者通过向某个主题发布消息，只有订阅这个主题的频道才能接收消息；这样实现了不同业务或者微服务的消息隔离；
### 1.2频道
&#160; &#160; &#160; &#160;频道连接消费者并在他们之前进行负载均衡，可以理解为一种排队机制；每次发布者发布一个消息到主题，每个频道就会去拷贝这个消息，然后消费者从特定的频道去读取消息（频道在消费者第一次订阅时创建）；如果某些消息一直没有被读取，频道将会对这些消息进行排序（最新的保留在内存中，溢出部分将会持久化到磁盘）
### 1.3 消息
&#160; &#160; &#160; &#160;消息构成了我们数据流的主干。 消费者可以选择完成消息，指示他们正常处理，或者请求他们稍后发送。 每个消息包含传递尝试次数的计数。当某些消息达到一定交付门槛后将会被客户端丢弃；
## 2.NSQ组件简介
&#160; &#160; &#160; &#160;NSQ是一个国外的一个致命短连接服务商提供的一个分布式消息队列中间件，具有高性能、高可靠和无单点故障等有点。
<div align="center"> 
![NSQ-architecture](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/NSQARCHITECTURE.jpg)
图2.0 NSQ架构
</div>
&#160; &#160; &#160; &#160;NSQ包括nsqlookupd、nsqd、nsqadmin、curl及nsq_to_file;nsqlookupd主要是用来管理拓扑信息，nsqd则主要负责接收和发送消息到客户端，nsqadmin提供了一个查看集群状态和管理的界面，curl主要是用于向指定URL发送网络请求，nsq_to_file主要用户将数据持久化。
### 2.1 nsqd
&#160; &#160; &#160; &#160;单个nsqd实例被设计为一次处理多个数据流。 流被称为“主题”，主题具有一个或多个“频道”。 每个频道都收到一个主题的所有消息的副本 。 在实践中，频道映射到消费主题的下游服务。
通过发布到命名主题或通过订阅命名主题上的频道，首次创建主题。 通过订阅命名的频道首次使用频道。当nsqd节点启动时，它将会注册到nsqlookupd节点，并广播告知，本节点存储的主题和频道信息；
&#160; &#160; &#160; &#160;主题和频道分别独立的缓存数据，从而防止消费者慢于其他频道的积压（同样适用于主题级别）。频道通常具有多个连接的客户端。 假设所有连接的客户端处于准备好接收消息的状态，每个消息将被传递给随机客户端
<div align="center"> 
![nsqd](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/nsqd.gif)
图2.1 NSQ主题频道订阅模式
</div>
&#160; &#160; &#160; &#160;总的来讲，消息是从主题 ->频道（每个频道接收该主题的所有消息的副本）的多播，但从频道->消费者均匀分布（每个消费者接收该频道的一部分消息）
### 2.2 nsqlookupd
&#160; &#160; &#160; &#160;Nsqlookupd是NSQ的一个帮助程序，它提供了一个目录查询服务，消费者可以在其中查找他们想要订阅的主题的nsqd实例的地址。 在配置方面，消费者与生产者分离。
&#160; &#160; &#160; &#160;每个nsqd有一个长TCP连接，周期性地推送其状态。 该数据用于通知nsqlookupd将向消费者提供哪些nsqd地址。 对于消费者来说，一个HTTP /lookup端点被公开用于轮询。
### 2.3 nsqadmin
&#160; &#160; &#160; &#160;它提供了一个Web UI来浏览主题/频道/消费者的层次结构，并检查每个层的深度和其他关键统计信息。 此外，它还支持一些管理命令，例如删除和清空频道
<div align="center"> 
![nsqadmin](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/nsqadmin.png)
图2.3-0 nsqadmin
</div>
<div align="center"> 
![nsqadmin](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/nsq-admin.png)
图2.3-1 nsqadmin
</div>
### 2.4 其他命令
* nsq_pubsub - 将类似HTTP接口的pubsub公开到NSQ集群中的主题
* nsq_to_file - 将给定主题的所有消息持久写入文件
* nsq_to_http - 对主题到（多个）端点的所有消息执行HTTP请求
<div align="center"> 
![nsq_to_http](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/nsq_to_http.png)
图2.4 nsq_to_http工作模式
</div>
## 3 NSQ设计理念
### 3.1 simplequeue
<div align="center"> 
![simplequeue](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/simplequeue.jpg)
图3.1 simplequeue
</div>
### 3.2 消除单点故障
&#160; &#160; &#160; &#160;NSQ通过分布式方式消除单点故障，因为在实际的实践中，可以启动多个nsqd节点，而每个节点都会在nsqlookupd中注册,然后nsqd客户端(消费者)通过nsqlookupd查询所有nsqd节点，然后通过TCP连接到指定主题的所有实例；
<div align="center"> 
![NSQ分布式架构](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/nsqds-consumer.jpg)
图3.2 NSQ分布式架构
</div>

### 3.3 消息到达保证
&#160; &#160; &#160; &#160;NSQ保证尽快发送一条消息至少一次，对于消费者来说，这种操作最好是幂等性的；
&#160; &#160; &#160; &#160;1. 客户端表示他们已准备好接收消息
&#160; &#160; &#160; &#160;2. NSQ发送消息并在本地临时存储数据（在重新队列或超时的情况下）
&#160; &#160; &#160; &#160;3. 客户端分别回复表示成功或失败的FIN（完成）或REQ（重新队列）。 
&#160; &#160; &#160; &#160;如果客户端不回复，NSQ将在可配置的持续时间后超时，并自动重新排队消息。
一种极端的情况是，nsqd进程非法关闭导致内存中的消息丢失（缓存数据未持久化到磁盘）；为解决可能导致的消息丢失问题，一种解决方案就是建立冗余的nsqd来接收一部分消息的副本；由于已经将消费者编写为幂等性操作，因此在这些消息上执行两次操作不会对下游产生影响，并允许系统在不丢失消息的情况下忍受任何单个节点故障。
### 3.4 有限内存容量
&#160; &#160; &#160; &#160;nsqd提供了一个配置选项--mem-queue-size ，它将确定给定队列内存中保留的消息数。 如果一个队列的深度超过这个阈值，消息将被透明地写入磁盘。 这将给定nsqd进程的内存占用nsqd为mem-queue-size；因此，通过将该值设为较小值将能更方便的保证消息的送达；磁盘备份的队列设计为在不清洁的重新启动时生效（尽管消息可能会传送两次）；正常关闭nsqd进程将能保证在内存、缓存已经正在发送中的消息的完整性；
<div align="center"> 
![消息溢出](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/messageoverflow.jpg)
图3.4 消息溢出
</div>
### 3.5 效率
&#160; &#160; &#160; &#160;NSQ旨在通过“memcached”命令协议进行通信，并具有简单的大小前缀响应。 所有消息数据保存在核心中，包括元数据，如尝试次数，时间戳等。这消除了从服务器到客户端来回复制数据，这是在重新排队消息时先前工具链的固有属性。 这也简化了客户端，因为它们不再需要负责维护消息状态。
此外，通过降低配置复杂性，大大减少了设置和开发时间（特别是在主题有1个消费者的情况下）。
对于数据协议，我们做出了一个关键的设计决策，通过将数据推送到客户端而不是等待其拉动来最大限度地提高性能和吞吐量。 这个我们称之为RDY状态的概念基本上是客户端流控制的一种形式。
当客户端连接到nsqd并订阅一个通道时，它的RDY值为0，这意味着频道不会向客户端发送任何消息。 只有当客户端准备好接收消息时，它会再次发送一个命令，将其RDY状态更新为一些具体的值，比如说100，表示它可以接受100个消息；然后，频道就会将100个消息将推送到客户端，然后在服务端递减该客户端RDY技术器。
客户端还会通过将RDY计数器和max-in-flight进行比较，当达到这个值的25%左右时，将会更新RDY计数器，来平衡性能（当然也会考虑连接多个nsqd实例的情况）。
<div align="center"> 
![nsq消息发送机制](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/nsqd.jpg)
图3.5 nsq消息发送机制
</div>
&#160; &#160; &#160; &#160;通过提高下游客户端批量处理消息的能力以及并行发送的能力，极大提高了消息处理的性能。

## 4 消息的生命周期
### 4.1
&#160; &#160; &#160; &#160;首先发布者将消息发送到本地的nsqd；发布者首先建立一个连接并发送一个PUB命令，该命令携带有主题以及消息体，然后events topic将会拷贝该消息到该消息下的每个频道。 比如，在这里我们有三个频道，消费者将从频道接收消息；
<div align="center"> 
![simplequeue](https://raw.githubusercontent.com/club326/club326.github.io/master/NSQ/message-worker.png)
图4.1 message-worker
</div>
每个消息会在频道里排队直到有worker线程消费它们，如果队列长度超过内存限制，消息将会持久到硬盘；
&#160; &#160; &#160; &#160;nsqd节点首先会把自己的地址广播给nsqlookupd，然后一旦它们注册了，工作线程worker就能通过nslookupd查询到每个nsqd节点以及其节点上的主题；然后，每个工作线程worker将会订阅每个nsqd服务器，表明它们已经为接收消息做好准备；同时，每个频道必须有足够的消费者来接收消息，否则这个频道的消息将会排队。
对消息有三种不同的处理策略：

* 丢弃--当消息尝试发送次数超过一定阀值时（MAX_DELIVERY_ATTEMPTS * BACKOFF_TIME）
* 结束--当消息已经被正常处理
* 重新排队--当错误发生时，消息将会重新排队等待再次发送
#### 优势：
* 协议简单--二进制通信协议速度快，实现方便；
* 启动简单--参数较少并且简单；
* 分布式--share nothing架构，在线扩展方便
#### 劣势：
* 无法建立主从复制--虽然可以通过设置来持久化队列，但还是存在丢失的可能
* 基本的消息路由--没有基于key的路由
* 非严格排序--消息会在任何时间以任何时间进入，所以要求每个数据带有时间戳
* 无法删除重复数据--NSQ通过心跳检测来探测消费者是否存活，存在worker线程心跳检测失败的可能性，因此必须通过单独的步骤来保证worker线程的幂等性；

## NSQ性能测试
### 测试1--磁盘持久化
开启消息队列持久化，所有消息队列都存放于磁盘中时,nsqd启动方式如下：
<code>./nsqd --lookupd-tcp-address=192.168.11.103:4160 -broadcast-address=192.168.11.103 -http-address=192.168.11.103:4151 -tcp-address=192.168.11.103:4150 -mem-queue-size=1
-mem-queue-size=1</code>
写入测试：<code>./bench_writer -nsqd-tcp-address=192.168.11.103:4150  -size=100 -topic=sub_bench01 -runfor=10m</code>
只读测试：<code>./bench_reader -nsqd-tcp-address=192.168.11.103:4150  -size=100 -topic=sub_bench01 -runfor=5m</code>
测试结果如下：
<table><tr><th>bench_type</th><th>ops</th><th>io</th></tr><tr><td>bench_writer</td><td>175248.414ops/s</td><td>16.713mb/s</td></tr><tr><td>bench_reader</td><td>123424.754ops/s</td><td>11.771mb/s</td></tr></table>
### 测试2--纯内存
虽然开启消息队列持久化，但所有测试的消息队列都保存在内存中，nsqd启动方式如下：
<code>./nsqd --lookupd-tcp-address=192.168.11.103:4160 -broadcast-address=192.168.11.103 -http-address=192.168.11.103:4151 -tcp-address=192.168.11.103:4150 -mem-queue-size=200000000</code>
写入测试：<code>./bench_writer -nsqd-tcp-address=192.168.11.103:4150  -size=100 -topic=sub_bench02 -runfor=1m</code>
只读测试：<code>./bench_reader -nsqd-tcp-address=192.168.11.103:4150  -size=100 -topic=sub_bench02 -runfor=1m</code>
测试结果如下：
<table><tr><th>bench_type</th><th>ops</th><th>io</th></tr><tr><td>bench_writer</td><td>1647930.402ops/s</td><td>157.159mb/s</td></tr><tr><td>bench_reader</td><td>214555.273ops/s</td><td>20.462mb/s</td></tr></table>
### 测试3--字节大小对性能影响
bench_writer测试结果如下：
<table><tr><th>bench_size</th><th>ops</th><th>io</th></tr><tr><td>100</td><td>1643414.337ops/s</td><td>156.728mb/s</td></tr><tr><td>200</td><td>1018577.658ops/s</td><td>194.278mb/s</td></tr><tr><td>300</td><td>637696.813ops/s</td><td>182.447mb/s</td></tr></table>

bench_reader测试结果如下：
<table><tr><th>bench_size</th><th>ops</th><th>io</th></tr><tr><td>100</td><td>237791.475ops/s</td><td>22.678mb/s</td></tr><tr><td>200</td><td>241534.862ops/s</td><td>46.069mb/s</td></tr><tr><td>300</td><td>197937.726ops/s</td><td>56.630mb/s</td></tr></table>







