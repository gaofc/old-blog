---
title: Zookeeper部署与动态扩容
layout: post
guid: urn:uuid:c116dfb7-8bd3-4c55-b788-c7f4e28d633b
tags:
  - zookeeper
  - 集群
  - 部署
  - 动态扩容
img: https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20161013/2016101300.jpg
---



## 背景
最近在一直维护以前的一个实时计算的系统，用到了很多有关storm、kafka、zookeeper之类的知识。自己也一直在学习这些系统的架构、源码。
由于一直是在以前集群的基础上不断修改，没有从头开始部署过集群。最近运维提了个问题，就是由于以前zookeeper是单机模式下运行的，万一这台机器出了问题，那就会影响整个实时计算，包括storm和kafka。于是问题来了，那怎么把它扩展成集群模式的呢？扩展的时候会影响现网正在运行的系统吗？

于是在研究研究了zookeeper有关部署和扩容的问题。把一些主要的过程记录在这里。

## 配置部署
首先我们先看看怎么部署zookeeper。在这里主要记录一些部署的步骤。集群的部署在后面会写。

首先从官网上下载zookeeper合适的版本，进行解压。之后上传到服务器端，解压到文件夹。
		
		tar -xvf zookeeper-3.4.6.tar

接下来就主要是修改zookeeper的配置文件了。进入到配置文件目录，有一个叫*zoo_sample.cfg*的文件。它只是一个实例文件，系统在实际运行的时候，读取的并不是它而是*zoo.cfg*文件。这个文件并不存在，需要我们手动创建它。所以我们可以将*zoo_sample.cfg*文件复制为*zoo.cfg*，接下来在*zoo.cfg*文件里面进行配置修改。

		cd zookeeper-3.4.6/conf/
		cp zoo_sample.cfg zoo.cfg
		vim zoo.cfg

打开文件内容如下：

	# The number of milliseconds of each tick
	tickTime=2000
	# The number of ticks that the initial 
	# synchronization phase can take
	initLimit=10
	# The number of ticks that can pass between 
	# sending a request and getting an acknowledgement
	syncLimit=5
	# the directory where the snapshot is stored.
	# do not use /tmp for storage, /tmp here is just 
	# example sakes.
	dataDir=/tmp/zookeeper
	# the port at which the clients will connect
	clientPort=2181
	# the maximum number of client connections.
	# increase this if you need to handle more clients
	#maxClientCnxns=60
	#
	# Be sure to read the maintenance section of the 
	# administrator guide before turning on autopurge.
	#
	# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
	#
	# The number of snapshots to retain in dataDir
	#autopurge.snapRetainCount=3
	# Purge task interval in hours
	# Set to "0" to disable auto purge feature
	#autopurge.purgeInterval=1

稍微解释一下常用的几个配置选项的含义：

- **tickTime** ZK中的一个时间单元。ZK中所有时间都是以这个时间单元为基础，进行整数倍配置的。例如，session的最小超时时间是2*tickTime。
- **initLimit** Follower在启动过程中，会从Leader同步所有最新数据，然后确定自己能够对外服务的起始状态。Leader允许F在 initLimit 时间内完成这个工作。通常情况下，我们不用太在意这个参数的设置。如果ZK集群的数据量确实很大了，F在启动的时候，从Leader上同步数据的时间也会相应变长，因此在这种情况下，有必要适当调大这个参数了。
- **syncLimit** 在运行过程中，Leader负责与ZK集群中所有机器进行通信，例如通过一些心跳检测机制，来检测机器的存活状态。如果L发出心跳包在syncLimit之后，还没有从F那收到响应，那么就认为这个F已经不在线了。注意：不要把这个参数设置得过大，否则可能会掩盖一些问题。
- **maxClientCnxns** 单个客户端与单台服务器之间的连接数的限制，是ip级别的，默认是60，如果设置为0，那么表明不作任何限制。请注意这个限制的使用范围，仅仅是单台客户端机器与单台ZK服务器之间的连接数限制，不是针对指定客户端IP，也不是ZK集群的连接数限制，也不是单台ZK对所有客户端的连接数限制。
- **autopurge.purgeInterval** 3.4.0及之后版本，ZK提供了自动清理事务日志和快照文件的功能，这个参数指定了清理频率，单位是小时，需要配置一个1或更大的整数，默认是0，表不开启自动清理功能。
- **autopurge.snapRetainCount** 这个参数和上面的参数搭配使用，这个参数指定了需要保留的文件数目。默认是保留3个。

一般在生产环境下，我们要将**autopurge.purgeInterval**这个选项打开，设置为12或者24小时或者其他时间，根据生产环境而定。之后可以将**dataDir**这个配置项设置为zookeeper的安装目录，这个是可选的。
**clientPort**这个视情况而定，如果你要在一台机器上部署多个zookeeper，那么就需要将端口号换掉，和其他zookeeper的端口号隔离开来。

## 集群模式
之后如果直接启动zookeeper的话，那就是**单机模式**运行了。但是如果要部署**集群**的话，还需要在文件末尾追加集群的信息。

		server.0=00.11.22.33:2888:3888
		server.1=11.22.33.44:2888:3888
		server.2=22.33.44.55:2888:3888
	
其中*server.0*中0指的是server的id，*00.11.22.33*是server的ip地址，2888是服务器与集群中Leader交换信息的端口，3888是集群中服务器进行选举的端口。*如果是在同一台机器部署的话，将ip地址写一样的，但是端口号就需要变更一下了。*

还记得配置文件中的**dataDir**这个配置项么，我们需要在**dataDir**这个文件目录下新建一个叫*myid*的文件，这是用于区别不同zookeeper的id文件。我们只需要在文件写上这个zookeeper的id就可以。

		cd {dataDir}
		vim myid
		i
		0
		(Esc)
		:wq

上面代码的0指的就是你要部署的这个zookeeper的id。

这样，一台zookeeper就配置完成，可以启动了。使用下面的命令进行启动。

		cd zookeeper-3.4.6/bin/
		./zkServer.sh start

启动完成后，可以通过```./zkServer.sh status```命令来查看zookeeper的状态信息。

![](https://blog-1253353025.cos.ap-chengdu.myqcloud.com/20161013/2016101301.png)

- 如果是单机模式的话，可以看到```Mode: standalone```这样的提示。
- 如果是集群模式的话，可能会看到zookeeper并没有启动成功。因为其他的服务器还没有启动，用于交换信息和选举的端口都没有打开，所以在*zookeeper.out*文件里你会看到出现各种端口连接错误。不用理会，这是正常的。

接下来按照同样的方式，你可以部署其他几台机器了。一般情况下，我们需要将所有服务器上的*zoo.cfg*文件保持一致。不同的是，在*myid*文件中，我们需要将自己不同的id写进去。然后依次启动所有服务，整个zookeeper集群就可以运行了。

我们照样可以使用```./zkServer.sh status```命令来查看zookeeper的运行状态。正常情况下，只会有一个leader，其他都是follower。

## 动态扩容
那么回归最开始的问题，如何在不影响现网的情况下动态扩容呢？
我们需要分2中情况讨论。*（需要注意的是，由于zookeeper的特性：集群中只要有过半的机器是正常工作的，那么整个集群对外就是可用的。所以我们假设所有集群的数量都是奇数）*

1. 集群本来是单机模式，需要将它扩容成集群模式
2. 集群本来就有>2台机器在运行，只是将它扩容成更多的机器

第一种情况在扩容的时候，短暂的停止服务是不可避免的。因为修改了配置文件，需要将原机器进行重启，而其他系统都依赖于此单机zookeeper服务。在扩容的时候，我们需要先将扩容的机器配置部署完成，在最后阶段，修改原机器上的配置文件后对服务进行重启。这个时候就会出现短暂的停止服务。

而且新机器部署的时候，会有端口异常的错误出现，这是因为单机模式下的zookeeper交换信息的端口2888和选举的端口3888都没有打开，所以会出错。这个时候不用理会，等最终原机器重启完成后，错误就会停止了。

第二种情况就比较好了，步骤还是相同的，先部署新机器，再重启老机器。在重启的过程中，需要保证**一台机器重启完成后，再进行下一台机器的重启**。这样就整个集群中每个时刻只有一台机器不能正常工作，而集群中有过半的机器是正常工作的，那么整个集群对外就是可用的。所以这个时候不会出现错误，也不会出现停止服务，整个扩容过程对用户是无感知的。

## END
**实践出真知**。对zookeeper的部署研究了一些时间，进行了很多的实验，得出了上面的一些结论。具体的扩容步骤我会在下一篇文章里面给出。如果上面有遗漏的地方或者不对的地方，欢迎讨论和指正。


