---
title: Storm参数配置及代码优化
layout: post
guid: urn:uuid:5d0fcad6-7dce-4855-98fc-015edd59e5a8
tags:
  - storm
  - config
  - code optimization
img: 
---


##背景

本人在维护一套由storm、kafka、zookeeper组成的分布式实时计算系统。当数据量很小的时候，系统处理起来其实是绰绰有余的，基本上按照系统默认配置来就可以了。然而当数据量增长到一定规模的时候，系统的各个配置都对整个系统的性能有着至关重要的影响。在不断的处理现网问题、研究的过程中，对系统的一些关键配置有一些心得。在这里分享出来，同大家一起学习交流。
今天我们在这里只介绍storm一些相关的比较重要的配置项和优化项。

##参数配置

###并行度

本人曾在上一篇文章中翻译过官方关于并行度的解释，但是实际在生产环境，这个并行度配置多少也是需要斟酌的。

####spout并行度

spout的并行度主要和数据源有很大的关系。我们使用的是kafka消息发布订阅系统作为数据源，而kafka也是一套分布式系统。它的每一个topic，也是分布在不同的partition分区上。而这个**partition数量便是spout并行度的上限**。spout会从指定topic的partition分区中取数据，这里有一个很重要的限制，就是**每一个partition只能被一个线程消费**。也就是，如果我们spout的并行度比partition的数量要少，那么，一定会有部分spout线程去消费多个partition，这个是可以的。但是，如果spout的并行度比partition的数量多，那么问题来了，由于一个partition只能被一个线程消费，那么一定会有部分spout线程没有数据可以消费。

所以在配置spout并行度时需要注意，**spout并行度<=topic的partition数量**。如果需要性能最大化，可以配置成**spout并行度=topic的partition数量**。如果这个并行度还是不足以支撑现有的数据，那么你应该考虑去给kafka扩容或者增加分区了。

####bolt并行度

bolt并行度有一个很简单的计算公式。

    	bolt并行度>=每秒需要处理消息数(n/s)*消息处理时间(ms)/1000

从这个计算公式可以很直观的看出原理。比如说一条消息在这个bolt中处理的时间是200ms，那么每一个bolt线程每秒钟可以处理5条数据。如果每秒中有1000个消息需要处理。那么我们至少需要200个线程去处理这些消息。

那么此bolt的并行度需要>=1000*2000/1000=200

###MaxSpoutPending

这个值的官方解释是

		The maximum number of tuples that can be pending on a spout task at any given time.
		This config applies to individual tasks, not to spouts or topologies as a whole.
		A pending tuple is one that has been emitted from a spout but has not been acked or failed yet.
		Note that this config parameter has no effect for unreliable spouts that don't tag their tuples with a message id.

在启用Ack 的情况下，Spout中有个RotatingMap用来保存Spout已经发送出去，但还没有等到Ack 结果的消息。RotatingMap的最大个数是有限制的，为**p*num-tasks**。其中p就是topology.max.spout.pending的值，也就是MaxSpoutPending（也可以由TopologyBuilder在setSpout 通过setMaxSpoutPending方法来设定），num-task是Spout的Task数。

所以这个值可以理解为，**单个spout可以同时处理的消息数**。这个值如果设置的过大，会导致消息在这个队列里积攒的时间过长，如果超过了超时时间，就会导致消息failed，触发重发机制，恶性循环。这个值如果设置的过小，则会引起后面bolt的消息饥饿，而且消息不能及时的处理。没能有效的利用资源，task的处理能力未充分应用，不能达到最佳的吞吐量。

这个值的官方建议是：*开始时设置一个很小的 TOPOLOGY_MAX_SPOUT_PENDING（对于 trident 可以设置为 1，对于一般的 topology 可以设置为 executor 的数量），然后逐渐增大，直到数据流不再发生变化。这时你可能会发现结果大约等于 “2 × 吞吐率(每秒收到的消息数) × 端到端时延” （最小的额定容量的2倍）。*


这里有个计算方法。

        1/Execute latency*complete latency

即使用拓扑的complete latency除以Execute latency。

我们假设一条消息被完整的处理时间为200ms，spout的Execute latency（执行nextTuple的时间）为2ms，则在200ms内，大约有100条数据被发送。也就是使用topo的complete latency除以Execute latency即可。但实际上不应该考虑如此极端的情况，以避免过多的fail出现，所以可以设置为上述值除以1.5左右。
超时时间

如果在storm ui中你看到整个topo或是spout有消息failed，但是单个的bolt并没有filed。那么一般情况是消息超时导致的。一条消息在进入等待队列中就开始计时，如果上面MaxSpoutPending设置的过大，或是机器负载过高等等，一条消息在有限的时间内没有完整的处理，那么这条消息就会failed，触发重发机制。如果是进行磁盘或者DB操作，那么就会引起数据重复。为了避免消息failed，一个方法就是设置合理的超时时间。系统默认的超时时间是30秒，你可以根据需要将它调的更大。

有几种方法设置这个时间。

* 提交Topology 时设置适当的消息超时时间，
    * conf.setMessageTimeoutSecs(60);
    * config.put(Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS,60);
* 也可以在storm.yaml中修改这个参数：
    * topology.message.timeout.secs: 30

##代码优化

###使用组件的并行度代替线程池

在storm中，我们可以很方便的调整spout/bolt的并行度，即使启动拓扑时设置不合理，也可以使用rebanlance命令进行动态调整。

但有些人可能会在一个spout/bolt组件的task内部启动一个线程池，这些线程池所在的task会比其余task消耗更多的资源，因此这些task所在的worker会消耗较多的资源，有可能影响其它拓扑的正常执行。

因此，应该**使用组件自身的并行度来代替线程池**，因为这些并行度会被合理分配到不同的worker中去。除此之外，还可以使用CGroup等技术进行资源的控制。

###不要在spout中处理耗时的操作

在storm中，spout是单线程的。如果nextTuple方法非常耗时，某个消息被成功执行完毕后，acker会给spout发送消息，spout若无法及时消费，则有可能导致 ack消息被丢弃，然后spout认为执行失败了。而在jstorm中将spout分成了3个线程，分别执行nextTuple, fail, ack方法。

###fieldsGrouping的数据均衡性

fieldsGrouping根据某个field的值进行分组。以userId为例，如果一个组件以userId的值作为分组，则具有相同userId的值会被发送到同一个task。如果某些userId的数据量特别大，会导致这接收这些数据的task负载特别高，从而导致数据均衡出现问题。我们在现网中曾经出现过这种数据倾斜的情况，就是因为单一用户刷单所导致单个task负载过高。

因此必须合理选择field的值，或者更换分组策略。

###优先使用localOrShuffleGrouping代替shuffleGrouping

localOrShuffleGrouping是指如果task发送消息给目标task时，发现同一个worker中有目标task，则优先发送到这个task；如果没有，则进行shuffle，随机选取一个目标task。

localOrShuffleGrouping其实是对shuffleGrouping的一个优化，因为消除了网络开销和序列化操作。

##总结

实践出真知。在踩过这些坑之后，才会对这些参数或者优化项有更深刻的认识。希望前面的这些介绍和理解对大家有帮助。也欢迎研究storm或者其它系统的大家随时交流，共同提高。






