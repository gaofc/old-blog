---
title: 实时大数据开发实践
layout: post
guid: urn:uuid:f2d36d25-c14c-462f-bdbf-e6b67622ead6
tags:
  - 大数据
  - spark
  - flink
  - storm
  - 容错
img: 
---

*本文主要从大数据起源谈起，介绍了几种主要的大数据处理框架，包括其中的容错机制，实现细节及原理等。再主要介绍了使用storm进行大数据开发的具体过程，以及开发过程中遇到的坑和一些优化。以下内容基于本人上次部门内分享整理，去掉了一些业务性的内容，尽量给大家展现一些技术细节。*


![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120702.PNG)

首先看一张图，截止2018年7月15日，这是大数据与AI的所有产品以及框架等全景，晕了没？可以看出来各大小公司、机构都在投入研发大数据及AI。我们身为开发人员，主要关注倒数第二行，open source的所有产品就可以了。接下来我会详细给大家介绍几个大数据框架，尤其是**实时大数据**框架，一些主要的实现细节以及原理等。

## 大数据起源

说起大数据处理，一切都起源于Google公司的经典论文。在当时（2000年左右），由于网页数量急剧增加，Google公司内部平时要编写很多的程序来处理大量的原始数据：爬虫爬到的网页、网页请求日志；计算各种类型的派生数据：倒排索引、网页的各种图结构等等。这些计算在概念上很容易理解，但由于输入数据量很大，单机难以处理。所以需要利用分布式的方式完成计算，并且需要考虑如何进行并行计算、分配数据和处理失败等等问题。

针对这些复杂的问题，Google决定设计一套抽象模型来执行这些简单计算，并**隐藏并发、容错、数据分布和均衡负载等方面的细节**。受到Lisp和其它函数式编程语言map、reduce思想的启发，论文的作者意识到许多计算都涉及对每条数据执行map操作，得到一批中间key/value对，然后利用reduce操作合并那些key值相同的k-v对。这种模型能很容易实现大规模并行计算。

事实上，与很多人理解不同的是，MapReduce对大数据计算的最大贡献，其实并不是它名字直观显示的Map和Reduce思想（正如上文提到的，Map和Reduce思想在Lisp等函数式编程语言中很早就存在了），而是这个计算框架可以运行在一群廉价的PC机上。MapReduce的伟大之处在于给大众们普及了工业界对于大数据计算的理解：**它提供了良好的横向扩展性和容错处理机制，至此大数据计算由集中式过渡至分布式。**以前，想对更多的数据进行计算就要造更快的计算机，而现在只需要添加计算节点。

话说当年的Google有三宝：**MapReduce、GFS和BigTable**。但Google三宝虽好，寻常百姓想用却用不上，原因很简单：它们都不开源。于是Hadoop应运而生，初代Hadoop的MapReduce和HDFS即为Google的MapReduce和GFS的开源实现（另一宝BigTable的开源实现是同样大名鼎鼎的HBase）。自此，大数据处理框架的历史大幕正式的缓缓拉开。

## 大数据架构

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120704.PNG)

刚才说了谷歌的三驾马车，说到实时大数据，我们一般把**消息队列、大数据框架、底层持久化**这三部分称为实时大数据架构的三驾马车。

但今天我们主要想讲的是中间这部分，大数据框架。

我们通常会把大数据框架分为三部分。

- **批处理系统**
- **流处理系统**
- **混合处理系统**

接下来给大家分别科普下这三种系统

### 批处理系统

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120706.PNG)

由于批处理系统在处理海量的持久数据方面表现出色，所以它通常被用来处理历史数据，很多OLAP（在线分析处理）系统的底层计算框架就是使用的批处理系统。但是由于海量数据的处理需要耗费很多时间，所以批处理系统一般**不适合用于对延时要求较高**的场景。

然后批处理系统的代表就是Hadoop。Hadoop是首个在开源社区获得极大关注的大数据处理框架，在很长一段时间内，它几乎可以作为大数据技术的代名词。

而且Hadoop不断发展完善，还集成了众多优秀的产品如非关系数据库HBase、数据仓库Hive、数据处理工具Sqoop、机器学习算法库Mahout、一致性服务软件ZooKeeper、管理工具Ambari等，形成了相对完整的生态圈和分布式计算事实上的标准。

### 流处理系统

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120707.PNG)

流处理系统好理解，那什么是流处理系统呢？小学的时候我们都做过这么一道数学题：一个水池有一个进水管和一个出水管，只打开进水管x个小时充满水，只打开出水管y个小时流光水，那么同时打开进水管和出水管，水池多长时间充满水？

Apache Storm是一种侧重于低延迟的流处理框架，它可以处理海量的接入数据，以近实时方式处理数据。Storm延时可以达到亚秒级。值得一提的是，一些国内的公司在Storm的基础上进行了改进，为推动流处理系统的发展做出了很大贡献。阿里巴巴的JStorm参考了Storm，并在网络IO、线程模型、资源调度及稳定性上做了改进。

提到Apache Samza，就不得不提到当前最流行的大数据消息中间件：Apache Kafka。Apache Kafka是一个分布式的消息中间件系统，具有高吞吐、低延时等特点，并且自带了容错机制。
如果已经拥有Hadoop集群和Kafka集群环境，那么使用Samza作为流处理系统无疑是一个非常好的选择。由于可以很方便的将处理过的数据再次写入Kafka，Samza尤其适合不同团队之间合作开发，处理不同阶段的多个数据流。

### 混合处理系统

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120708.PNG)

下面介绍下混合处理系统的代表框架Spark和Flink。

- **Spark**

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120709.PNG)

如果说如今大数据处理框架处于一个群星闪耀的年代，那Spark无疑就是所有星星中最闪亮的那一颗。

Spark由加州大学伯克利分校AMP实验室开发，最初的设计受到了MapReduce思想的启发，但不同于MapReduce的是，Spark通过内存计算模型和执行优化大幅提高了对数据的处理能力

而且除了最初开发用于批处理的Spark Core和用于流处理的Spark Streaming，Spark还提供了其他编程模型用于支持图计算（GraphX）、交互式查询（Spark SQL）和机器学习（MLlib）。


- **Flink**

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120710.PNG)

有趣的是，同样作为混合处理框架，Flink的思想与Spark是完全相反的：Spark把流拆分成若干个小批次来处理，而Flink把批处理任务当作有界的流来处理。

除了流处理（DataStream API）和批处理（DataSet API）之外，Flink也提供了类SQL查询（Table API）、图计算（Gelly）和机器学习库（Flink ML）。而令人惊讶的是，在很多性能测试中，Flink甚至略优于Spark。

在目前的数据处理框架领域，Flink可谓独树一帜。虽然Spark同样也提供了批处理和流处理的能力，但Spark流处理的微批次架构使其响应时间略长。Flink流处理优先的方式实现了低延迟、高吞吐和真正逐条处理。

同样，Flink也并不是完美的。Flink目前最大的缺点就是缺乏在大型公司实际生产项目中的成功应用案例。相对于Spark来讲，它还不够成熟，社区活跃度也没有Spark那么高。但假以时日，Flink必然会改变数据处理框架的格局。

## 容错

说到容错，首先必须要明确几个概念。

- **at most once**：最多消费一次，会存在数据丢失
- **at least once**：最少消费一次，保证数据不丢，但是有可能重复消费 
- **exactly once**：精确一次，无论何种情况下，数据都只会消费一次，这是我们最希望看到的结果

### Spark Streaming容错

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120712.PNG)

我们先看看Spark streaming的at least once是如何实现的，Spark streaming的每个batch可以看做是一个Spark任务，receiver会先将数据写入WAL，保证receiver宕机时，从数据源获取的数据能够从日志中恢复（注意这里，早期的Spark streaming的receiver存在重复接收数据的情况），并且依赖RDD实现内部的exactly once（可以简单的理解采用批量计算的方式来实现）。

上面简单解释了Spark streaming依赖源数据写WAL和自身RDD机制提供了容灾功能，保证at least once，但是依然无法保证exactly once，在回答这个问题前，我们再来看一下，什么情况Spark streaming的数据会重复计算。

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120713.PNG)

这里我们主要关注图中的3个红框：
Spark streaming的RDD机制只能保证内部计算exactly once（图中的1），但这是不够的，假如某个batch中，sink处一部分数据已经入库，这时候某个sink节点宕机，导致该节点处理的数据重复输出（图中的3）。

还有另一种情况就是receiver处重复接收数据（图中的2），我们看一下receiver重复接收数据的情况：

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120714.PNG)

根据上面的讨论，可以得出：一个流式系统如果要做到exactly once，必须满足3点：

1. receiver处保证exactly once
2. 流式系统自身保证exactly once
3. sink处保证exactly once

这里数据源采用Kafka举例是因为Kafka作为目前主流的分布式消息队列，比较有代表性。Kafka consumer的position可以保存在ZK或者Kafka中，也可以由consumer自己来保存。前者的话就可能存在数据消费和position更新不一致的问题（因为无法保证**原子性**，也是之前Spark streaming采用的方式），而采用后者的话，consumer可以采用**事务更新**的方式（写本地或者采用事务的方式写数据库），保证数据消费和position更新的**原子性**，从而实现exactly once

Spark streaming自己维护position，streaming的worker直接从Kafka读取数据，position由Spark streaming管理，不再依赖ZK保存，同时保证数据消费和更新position的原子性，从而实现exactly once。
而sink处的exactly once的实现则**视外部系统而定**，比如文件系统本身就支持**幂等**（同一个操作执行多次，不会改变之前的结果），同时Spark streaming也提供了api，用户可以自己实现sink处的事务更新，receiver、sink和Spark streaming三者结合起来才能实现了真正的exactly once。


### Flink容错

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120716.PNG)

barrier是分布式snapshot实现中一个非常核心的元素，barrier和records一起在流式系统中传输，barrier是当前snapshot和下一个snapshot的分界点，它携带了当前snapshot的id，假设目前在做snapshot N，算子在发送barrier N之前，都会对当前的状态做checkpoint（checkpoint数据可以保存在外部系统中，如HDFS），checkpoint只包含了barrier N之前的数据状态，不会涉及barrier N之后的数据。

Kafka position也是由Flink自己维护的，所以能够保证receiver处的exactly once，sink处也同样存在Spark streaming一样的问题，exactly once依赖外部系统或需要用户自己实现。

## Storm

### Storm核心概念

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120718.PNG)

**Topology**：storm中运行的一个实时应用程序，因为各个组件间的消息流动形成逻辑上的一个拓扑结构。
**Spout**：在一个topology中产生源数据流的组件。通常情况下spout会从外部数据源中读取数据，然后转换为topology内部的源数据。Spout是一个主动的角色，其接口中有个nextTuple()函数，storm框架会不停地调用此函数，用户只要在其中生成源数据即可。
**Bolt**：在一个topology中接受数据然后执行处理的组件。Bolt可以执行过滤、函数操作、合并、写数据库等任何操作。Bolt是一个被动的角色，其接口中有个execute(Tuple input)函数,在接受到消息后会调用此函数，用户可以在其中执行自己想要的操作。
**Tuple**：一次消息传递的基本单元。本来应该是一个key-value的map，但是由于各个组件间传递的tuple的字段名称已经事先定义好，所以tuple中只要按序填入各个value就行了，所以就是一个value list.
**Stream**：源源不断传递的tuple就组成了stream。
**stream grouping**：即消息的partition方法。Storm中提供若干种实用的grouping方式，包括shuffle, fields hash, all, global, none, direct和localOrShuffle等

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120717.PNG)

storm启动过程如图，不再赘述。

### Storm容错

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120719.PNG)

如图所示，如果boltB节点宕机了，那么storm自身的ack机制，保证了每条消息必须处理一次，检测到boltB节点的失败，storm会将数据重放，则导致有些数据被处理了两次。所以根据之前spark和flink的思想，我们需要在sink处实现幂等，才会保证数据不会出错。

### Storm Ack机制

上图红线表示了storm是如何保证数据至少处理一次的，而具体的ack实现则用了非常优雅的方式。

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120720.PNG)

我们知道两个相同的数字**异或**值为零。如果一个异或表达式，其中的每一个数字都出现了**两次**，则整个表达式的结果为零。有了这个认知，如图所示，每一个节点在发送、接受一条消息时候，会向acker进程发送当前消息id及一个随机数，如果这个消息id下的值变成了0，那么说明这条消息被完整的处理完成了。如果在一个超时时间内没有变成0，则说明在某一个节点上处理失败了，storm则会重放这条消息，重新处理一次，由此机制，保证了at least once。

## 业务场景

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120721.PNG)

我们所在风控组，主要使用了实时大数据框架完成了如图业务场景，使用架构如图所示。其中计算pv/uv/amt使用了spark streaming，主要原因是这几个指标是聚合指标，比如1分钟内，5分钟内等，所以这种业务场景非常适合使用spark streaming这种**微批处理**的特性。

## 代码示例

首先是create topology，需要指定spout、bolt具体的处理类，指定一个id，指定并行度，还需要指定上游节点是什么，分组方式是什么等。

```java
    public static StormTopology createTopology(String zkHosts, JSONObject config) {
        TopologyBuilder topologyBuilder = new TopologyBuilder();
        topologyBuilder.setSpout(DEVICE_INFO_SPOUT,
                KafkaSpoutBuilder.createSpout(zkHosts, (String) config.get(Constants.KAFKA_TOPIC), DEVICE_INFO_SPOUT_ID),
                (Long) config.get(SPOUT_PARALLEL));
        topologyBuilder.setBolt(DEVICE_INFO_PARSE_BOLT,
                new LogParseBolt(DeviceInfoParser.class.getName(),
                        new String[]{DeviceInfoIos.DEVICE_ACCOUNTS_STREAM, DeviceInfoIos.DEVICE_COMMON_STREAM, DeviceInfoIos.DEVICE_CHECK_STREAM}),
                (Long) config.get(PARSE_BOLT_PARALLEL))
                .localOrShuffleGrouping(DEVICE_INFO_SPOUT);
        topologyBuilder.setBolt(DEVICE_COMMON_CHECK_BOLT,
                new DeviceCommonCheckBolt(),
                (Long) config.get(COMMON_CHECK_BOLT_PARALLEL))
                .fieldsGrouping(DEVICE_INFO_PARSE_BOLT, DeviceInfoIos.DEVICE_COMMON_STREAM, new Fields(Constants.PARSER_KEY_FIELD));
        topologyBuilder.setBolt(DEVICE_ACCOUNTS_CHECK_BOLT,
                new DeviceAccountsCheckBolt(),
                (Long) config.get(ACCOUNTS_CHECK_BOLT_PARALLEL))
                .fieldsGrouping(DEVICE_INFO_PARSE_BOLT, DeviceInfoIos.DEVICE_ACCOUNTS_STREAM, new Fields(Constants.PARSER_KEY_FIELD));
        topologyBuilder.setBolt(DEVICE_CHECK_RESULT_BOLT,
                new DeviceCheckResultBolt(),
                (Long) config.get(CHECK_RESULT_BOLT_PARALLEL))
                .fieldsGrouping(DEVICE_INFO_PARSE_BOLT, DeviceInfoIos.DEVICE_CHECK_STREAM, new Fields(Constants.PARSER_KEY_FIELD));
        return topologyBuilder.createTopology();
    }
```

之后再main方法里，提交拓扑。

```java
StormSubmitter.submitTopology(DEVICE_INFO_TOPOLOGY, config, DeviceInfoTopology.createTopology(zkHosts, object));
```

提交成功后任务拓扑图如下

![大数据产品](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120730.png)

spout类

```java
public class KafkaSpoutBuilder {
    private static final String ZK_ROOT = "/kafka_storm";

    public static KafkaSpout createSpout(String zkHosts, String topicName, String spoutId) {
        return createSpout(zkHosts, topicName, spoutId, false);
    }
    public static KafkaSpout createSpout(String zkHosts, String topicName, String spoutId, boolean fromStart) {
        BrokerHosts hosts = new ZkHosts(zkHosts);
        SpoutConfig kafkaConfig = new SpoutConfig(hosts, topicName, ZK_ROOT, spoutId);
        kafkaConfig.maxOffsetBehind = Long.MAX_VALUE;
        kafkaConfig.scheme = new SchemeAsMultiScheme(new StringScheme());
        kafkaConfig.forceFromStart = fromStart;
        KafkaSpout spout = new KafkaSpout(kafkaConfig);
        return spout;
    }
}
```

bolt类
LogParseBolt类用了工厂方法，继承了BaseRichBolt类，需要覆盖3个方法```prepare、execute、declareOutputFields```，值得一提的是如果继承了BaseRichBolt类，需要自己处理ack和fail，所以在业务处理完成之后必须对消息进行ack或fail。但是如果继承了BaseBasicBolt类，你只需要实现业务逻辑即可，不需要关注ack。

```java
public class LogParseBolt extends BaseRichBolt {
    private static final Logger logger = Logger.getLogger(LogParseBolt.class);
    private transient TopologyContext context;
    private transient OutputCollector collector;
    private transient LogParser parser;
    private final String parserClassName;
    private final String[] streams;

    public LogParseBolt(String parserClassName) {
        this.parserClassName = parserClassName;
        this.streams = new String[]{};
    }

    public LogParseBolt(String parserClassName, String[] streams) {
        this.parserClassName = parserClassName;
        this.streams = streams;
    }

    @Override
    public void prepare(Map map, TopologyContext topologyContext, OutputCollector outputCollector) {
        context = topologyContext;
        collector = outputCollector;
        try {
            parser = (LogParser)Class.forName(parserClassName).newInstance();
            parser.init(map);
        } catch (Exception e) {
            logger.error("fail to init log parser.", e);
        }
    }

    @Override
    public void execute(Tuple tuple) {
        String log = "";
        try {
            log = tuple.getString(0);
            for (TupleValue value : parser.parse(log)) {
                if (value.getStream() == null)
                    collector.emit(tuple, new Values(value.getGroupKey(), value));
                else
                    collector.emit(value.getStream(), tuple, new Values(value.getGroupKey(), value));
            }
        } catch (Exception e) {
            logger.error("fail to parse log.\n" + log, e);
        }
        finally {
            collector.ack(tuple);
        }
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer outputFieldsDeclarer) {
        if (streams.length == 0)
            outputFieldsDeclarer.declare(new Fields(Constants.PARSER_KEY_FIELD, Constants.PARSER_DATA_FIELD));
        else {
            for (String stream : streams) {
                outputFieldsDeclarer.declareStream(stream,
                        new Fields(Constants.PARSER_KEY_FIELD, Constants.PARSER_DATA_FIELD));
            }
        }
    }
}
```

## 现网问题

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20181207/2018120726.PNG)

现网上出现过比较大的两次问题。
一次是因为用户刷单，导致的数据倾斜。因为之前用的分组方式是根据用户id进行fieldGrouping，所以相同的用户会被分配到相同的结点，但是刷单导致了数据倾斜，单个结点负载过高，阻塞了任务。

第二次是因为数据量过大导致机器负载太高，在重启的时候拉不起来服务。也针对这种情况进行了一些改进。


## 参数调优

**spout并行度**
spout的并行度主要和数据源有很大的关系。我们使用的是kafka消息发布订阅系统作为数据源，而kafka也是一套分布式系统。它的每一个topic，也是分布在不同的partition分区上。而这个partition数量便是spout并行度的上限。spout会从指定topic的partition分区中取数据，这里有一个很重要的限制，就是每一个partition只能被一个线程消费。也就是，如果我们spout的并行度比partition的数量要少，那么，一定会有部分spout线程去消费多个partition，这个是可以的。但是，如果spout的并行度比partition的数量多，那么问题来了，由于一个partition只能被一个线程消费，那么一定会有部分spout线程没有数据可以消费。

所以在配置spout并行度时需要注意，spout并行度<=topic的partition数量。如果需要性能最大化，可以配置成spout并行度=topic的partition数量。如果这个并行度还是不足以支撑现有的数据，那么你应该考虑去给kafka扩容或者增加分区了。

**bolt并行度**
bolt并行度有一个很简单的计算公式。

	bolt并行度>=每秒需要处理消息数(n/s)*消息处理时间(ms)/1000
	
从这个计算公式可以很直观的看出原理。比如说一条消息在这个bolt中处理的时间是200ms，那么每一个bolt线程每秒钟可以处理5条数据。如果每秒中有1000个消息需要处理。那么我们至少需要200个线程去处理这些消息。

那么此bolt的并行度需要>=1000*2000/1000=200

**MaxSpoutPending**
这个值的官方解释是

	The maximum number of tuples that can be pending on a spout task at any given time.
	This config applies to individual tasks, not to spouts or topologies as a whole.
	A pending tuple is one that has been emitted from a spout but has not been acked or failed yet.
	Note that this config parameter has no effect for unreliable spouts that don't tag their tuples with a message id.

在启用Ack 的情况下，Spout中有个RotatingMap用来保存Spout已经发送出去，但还没有等到Ack 结果的消息。RotatingMap的最大个数是有限制的，为p*num-tasks。其中p就是topology.max.spout.pending的值，也就是MaxSpoutPending（也可以由TopologyBuilder在setSpout 通过setMaxSpoutPending方法来设定），num-task是Spout的Task数。

所以这个值可以理解为，单个spout可以同时处理的消息数。这个值如果设置的过大，会导致消息在这个队列里积攒的时间过长，如果超过了超时时间，就会导致消息failed，触发重发机制，恶性循环。这个值如果设置的过小，则会引起后面bolt的消息饥饿，而且消息不能及时的处理。没能有效的利用资源，task的处理能力未充分应用，不能达到最佳的吞吐量。

这个值的官方建议是：开始时设置一个很小的 TOPOLOGY_MAX_SPOUT_PENDING（对于 trident 可以设置为 1，对于一般的 topology 可以设置为 executor 的数量），然后逐渐增大，直到数据流不再发生变化。这时你可能会发现结果大约等于 “2 × 吞吐率(每秒收到的消息数) × 端到端时延” （最小的额定容量的2倍）。

这里有个计算方法。

    1/Execute latency*complete latency
即使用拓扑的complete latency除以Execute latency。

我们假设一条消息被完整的处理时间为200ms，spout的Execute latency（执行nextTuple的时间）为2ms，则在200ms内，大约有100条数据被发送。也就是使用topo的complete latency除以Execute latency即可。但实际上不应该考虑如此极端的情况，以避免过多的fail出现，所以可以设置为上述值除以1.5左右。

**超时时间**
如果在storm ui中你看到整个topo或是spout有消息failed，但是单个的bolt并没有filed。那么一般情况是消息超时导致的。一条消息在进入等待队列中就开始计时，如果上面MaxSpoutPending设置的过大，或是机器负载过高等等，一条消息在有限的时间内没有完整的处理，那么这条消息就会failed，触发重发机制。如果是进行磁盘或者DB操作，那么就会引起数据重复。为了避免消息failed，一个方法就是设置合理的超时时间。系统默认的超时时间是30秒，你可以根据需要将它调的更大。

## 代码优化

**使用组件的并行度代替线程池**
在storm中，我们可以很方便的调整spout/bolt的并行度，即使启动拓扑时设置不合理，也可以使用rebanlance命令进行动态调整。

但有些人可能会在一个spout/bolt组件的task内部启动一个线程池，这些线程池所在的task会比其余task消耗更多的资源，因此这些task所在的worker会消耗较多的资源，有可能影响其它拓扑的正常执行。

因此，应该使用组件自身的并行度来代替线程池，因为这些并行度会被合理分配到不同的worker中去。除此之外，还可以使用CGroup等技术进行资源的控制。

**不要在spout中处理耗时的操作**
在storm中，spout是单线程的。如果nextTuple方法非常耗时，某个消息被成功执行完毕后，acker会给spout发送消息，spout若无法及时消费，则有可能导致 ack消息被丢弃，然后spout认为执行失败了。而在jstorm中将spout分成了3个线程，分别执行nextTuple, fail, ack方法。

**fieldsGrouping的数据均衡性**
fieldsGrouping根据某个field的值进行分组。以userId为例，如果一个组件以userId的值作为分组，则具有相同userId的值会被发送到同一个task。如果某些userId的数据量特别大，会导致这接收这些数据的task负载特别高，从而导致数据均衡出现问题。我们在现网中曾经出现过这种数据倾斜的情况，就是因为单一用户刷单所导致单个task负载过高。

因此必须合理选择field的值，或者更换分组策略。

**优先使用localOrShuffleGrouping代替shuffleGrouping**
localOrShuffleGrouping是指如果task发送消息给目标task时，发现同一个worker中有目标task，则优先发送到这个task；如果没有，则进行shuffle，随机选取一个目标task。

localOrShuffleGrouping其实是对shuffleGrouping的一个优化，因为消除了网络开销和序列化操作。

## END
以上都是本人上次部门分享的全部内容，可能内容较多，但是都是干货，如果你是一个大数据初学者或者入门，会对你有很大帮助。也欢迎研究大数据的随时交流。




