---
title: 翻译 理解Storm拓扑的并行性
layout: post
guid: urn:uuid:ae08ed4e-defd-4da5-94ac-397897ea9ff0
tags:
  - storm
  - 并行性
  - worker
  - 参数
  - executor
img: 
---


*本文是本人根据Storm官方文档个人翻译整理的，如果有不妥或者错误之处，欢迎指正。[原英文官方文档](http://storm.apache.org/releases/0.9.6/Understanding-the-parallelism-of-a-Storm-topology.html)*

## 是什么使一个拓扑运行的

Storm区分了用于在Storm集群中实际运行拓扑的以下三个主要实体：

1. 工作进程（Worker processes）
2. 执行器（Executors）
3. 任务（Tasks）

这里是他们的关系的简单说明：

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/2017012601.png)

工作进程运行着一个拓扑的子集。一个工作进程（worker processes）是属于一个特定的拓扑的，工作进程可以运行这个拓扑中一个或多个组件（spouts或者bolts）的一个或多个执行器（executors）。一个运行的拓扑是由多个这样的进程组成的，这些进程都是运行在storm集群中的多个机器中。

执行器（executor）是一个由工作进程创建出来的线程。它会运行一个或多个任务（tasks），而且这些任务都属于同一个组件（spouts或者bolts）的。 

任务（task）是真正执行数据处理的--在代码中实现的每个spout或bolt在集群中执行任意数量的任务。

一个组件的任务数量在一个拓扑的生命周期中总是相同的，但是一个组件的执行器（线程）可能会随时间而变化。这就意味着下面这个情况总是成立的：`#threads ≤ #tasks`。 默认情况下，任务的数量设置为与执行器的数量相同，即Storm将为每个线程运行一个任务。

## 配置拓扑的并行性

注意，在Storm的术语中，“并行性（parallelism）”特别用于描述所谓的*并行性提示（parallelism hint）*，这指的就是组件的执行器（线程）的初始数量。 在本文中，在更一般的意义上，我们不仅使用术语“并行性”来描述如何配置执行程序的数量，还用来描述配置工作进程的数量和Storm拓扑的任务数。 当我们在Storm的正常，狭义的定义中使用“并行性（parallelism）”时，我们会特别提出。

以下部分概述了各种配置选项以及如何你的在代码中进行设置。设置这些选项有不止一种方式，但是表中只列出了其中一部分。Storm目前具有以下配置设置优先级顺序：

    defaults.yaml < storm.yaml < topology-specific configuration < internal component-specific configuration < external component-specific configuration.

### 工作进程（Worker processes）的数量

- 说明：要为群集中的计算机上的拓扑创建多少个工作进程。
- 配置选项：`TOPOLOGY_WORKERS`
- 如何在代码中设置（示例）：
	- `Config#setNumWorkers`

### 执行器（Executors）的数量（线程）
- 说明：每个组件生成多少个executors。
- 配置选项：无（将parallelism_hint参数传递给setSpout或setBolt）
- 如何在代码中设置（示例）：
	- `TopologyBuilder#setSpout()`
	- `TopologyBuilder#setBolt()`
	- 注意Storm 0.8版本，parallelism_hint参数现在指定该bolt的执行器的初始数量（而不是任务！）。

### 任务（Tasks）的数量
- 说明：每个组件生成多少个任务。
- 配置选项：`TOPOLOGY_TASKS`
- 如何在代码中设置（示例）：
	- `ComponentConfigurationDeclarer#setNumTasks()`

以下是在实践中设置的示例代码段：

	topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
		.setNumTasks(4)
		.shuffleGrouping("blue-spout");

在上面的代码中，我们配置Storm运行GreednBolt时，初始数量为两个执行器（Executors）和四个相关任务（Tasks）。 Storm将对每个执行器（线程）运行两个任务。 如果没有显式配置任务数，Storm将默认为每个executor运行一个task。

## 一个运行拓扑的实例

下图展示了一个简单的拓扑在运行中是什么样的。这个拓扑包含了3个组件，一个叫BlueSpout的spout，2个bolts分别为GreenBolt和YellowBolt。

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/2017012602.png)

这些组件是这样连接的，BlueSpout将其输出发送到GreenBolt，GreenBolt又将自己的输出发送到YellowBolt。

GreenBolt是根据上面的代码片段配置的，而Blue Spout和Yellow Bolt只设置parallelism hint（executors的数量）。 这里是相关的代码：

	Config conf = new Config();
	conf.setNumWorkers(2); // use two worker processes

	topologyBuilder.setSpout("blue-spout", new BlueSpout(), 2); // set parallelism hint to 2

	topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
	               .setNumTasks(4)
	               .shuffleGrouping("blue-spout");

	topologyBuilder.setBolt("yellow-bolt", new YellowBolt(), 6)
	               .shuffleGrouping("green-bolt");

	StormSubmitter.submitTopology(
	        "mytopology",
	        conf,
	        topologyBuilder.createTopology()
	    );


当然，Storm还提供了额外的配置设置来控制拓扑的并行性，包括：

- `TOPOLOGY_MAX_TASK_PARALLELISM`：此设置为单个组件生成的executor数量设置了上限。它通常用于在测试期间，限制在本地模式下运行拓扑时生成的线程数。你可以设置这个选项`Config#setMaxTaskParallelism()`。

## 如何更改运行拓扑的并行性

Storm一个很好的特性就是，你可以增加或减少worker进程或executors的数量，而无需重新启动集群或拓扑。 这个行为被称为rebalancing（重新平衡）。

你有2个选项去重新平衡一个拓扑：

1. 使用Storm UI去平衡拓扑
2. 使用CLI工具，如下所述

以下是使用CLI工具的示例：

	## Reconfigure the topology "mytopology" to use 5 worker processes,
	## the spout "blue-spout" to use 3 executors and
	## the bolt "yellow-bolt" to use 10 executors.

	$ storm rebalance mytopology -n 5 -e blue-spout=3 -e yellow-bolt=10

## 参考文献

- [概念](http://storm.apache.org/releases/1.0.1/Concepts.html)
- [配置](http://storm.apache.org/releases/1.0.1/Configuration.html)
- [在生产集群中运行拓扑](http://storm.apache.org/releases/1.0.1/Running-topologies-on-a-production-cluster.html)
- [本地模式](http://storm.apache.org/releases/1.0.1/Local-mode.html)
- [教程](http://storm.apache.org/releases/1.0.1/Tutorial.html)
- [Storm API文档（最值得注意的是Config类）](http://storm.apache.org/releases/1.0.1/javadocs/)


