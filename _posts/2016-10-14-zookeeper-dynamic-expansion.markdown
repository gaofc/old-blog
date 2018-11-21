---
title: Zookeeper动态扩容详细步骤
layout: post
guid: urn:uuid:64d53cd8-345e-4711-8fb3-575b3d0c452d
tags:
  - zookeeper
  - 集群
  - 动态扩容
  - 单机
img: https://blog-1253353025.cos.ap-chengdu.myqcloud.com/2016101300.jpg
---



上一篇文章分析了动态扩容的方法，并没有给出详细步骤，这次给出扩容的详细步骤。
首先分两种情况，第一种情况是以前是单机状态，现在将其扩展为多个机器的集群状态。另一种情况是以前是集群状态，现在扩展为更多的集群状态。

## 单机扩展
### 步骤
1. 首先假设原机器ip为**{OLD_SERVER}**，新机器为2台，**{NEW_SERVER1}、{NEW_SERVER2}**，zookeeper的安装目录为**{ZOO_HOME}**。*新机器的安装目录尽量和原机器保持一致，这样configure文件就可以统一管理，在所有机器上保持一致。*

2. 在新机器**{NEW_SERVER1}、{NEW_SERVER2}**上，下载解压合适的zookeeper版本到指定目录。

		cd {ZOO_HOME}
		tar -xvf zookeeper-3.4.6.tar

3. 将**原机器**zookeeper中的*zoo.cfg*文件复制到**2台新机器的**conf文件夹下。
		
		ssh {OLD_SERVER}
		cd {ZOO_HOME}/zookeeper-3.4.6/conf
		scp zoo.cfg {USERNAME}@{NEW_SERVER1}:/{ZOO_HOME}/conf/
		scp zoo.cfg {USERNAME}@{NEW_SERVER2}:/{ZOO_HOME}/conf/

4. 进入到**新机器**，打开*zoo.cfg*文件，看是否有配置server项。*类似server.x=:xxxxxxxx:xxxx:xxxx*，如果有的话，将其删除，文件后面追加

		server.0={ODL_SERVER}:2888:3888
 		server.1={NEW_SERVER1}:2888:3888
		server.2={NEW_SERVER2}:2888:3888
其中，*server.0、server.1、server.2*中0、1、2代表的是zookeeper的id，下面会用到。这里指定新机器{NEW_SERVER1}的id为1，{NEW_SERVER2}的id为2，原机器的id为0。

5. 检查*zoo.cfg*文件中**dataDir**配置项的值，一般情况下等于{ZOO_HOME}。在{NEW_SERVER1}上，进入{dataDir}目录，新建myid文件,将上面的id写入文件。由于{NEW_SERVER1}配置的id是1，则在myid文件中写入1。

		ssh {NEW_SERVER1}
		cd {dataDir}
		vim myid
		i
		1
		(Esc)
		:wq

	在{NEW_SERVER2}上，myid文件中写入2。

		ssh {NEW_SERVER2}
		cd {dataDir}
		vim myid
		i
		2
		(Esc)
		:wq

6. **依次连接**到新机器{NEW_SERVER1}、{NEW_SERVER2}上，进入bin文件夹下启动zookeeper。
		
		cd {ZOO_HOME}/bin
		./zkServer.sh start

7. 查看zookeeper状态，检查zookeeper.out文件。*一般情况会提示It is probably not running.*，而且zookeeper.out文件里面会提示端口连接的错误，不用理会。

		./zkServer.sh status

8. 连接到**原机器**，将**新的** *zoo.cfg*文件拷贝到原机器上，也就是所有机器的配置文件**保持一致**。
9. 进入{dataDir}目录，查看是否有myid文件，有的话将其删除,将原机器的id写入myid文件。由于原机器配置的id是0，则在myid文件中写入0。

10. 进入bin文件夹下重启zookeeper。

		cd {ZOO_HOME}/bin
		./zkServer.sh restart

zookeeper重启时会有**短暂的停止服务**，在此期间所有依赖zookeeper的系统都会停止服务，比如storm和kafka。不过这个重启过程不会很长，重启之后就恢复正常了。

### 大功告成
这个时候在**新旧机器**上运行```./zkServer.sh status```命令，可以看到，有1个leader，2个follower。这时表明zookeeper已经由单机模式扩展为3台机器构成的集群模式了。

之后要做的就是在所有使用到zookeeper的地方，将以前单机的server地址替换3台机器所有的server地址就可以了。

## 集群扩展
由于zookeeper的特性：集群中只要有过半的机器是正常工作的，那么整个集群对外就是可用的。所以我们假设所有集群的数量都是奇数，我们以原集群是3台机器，现在将其扩展为5台为例。

### 步骤
1. 首先假设原机器ip为**{OLD_SERVER1}、{OLD_SERVER2}、{OLD_SERVER3}**，新机器为2台，**{NEW_SERVER4}、{NEW_SERVER5}**，zookeeper的安装目录为**{ZOO_HOME}**。*新机器的安装目录与原机器保持一致*

2. 在新机器**{NEW_SERVER4}、{NEW_SERVER5}**上，下载解压合适的zookeeper版本到指定目录。

		cd {ZOO_HOME}
		tar -xvf zookeeper-3.4.6.tar

3. 我们假设原机器上所有的*zoo.cfg*文件是一致的。将任意一台**原机器**zookeeper中的*zoo.cfg*文件复制到**2台新机器的**conf文件夹下。
		
		ssh {OLD_SERVER1}
		cd {ZOO_HOME}/zookeeper-3.4.6/conf
		scp zoo.cfg {USERNAME}@{NEW_SERVER4}:/{ZOO_HOME}/conf/
		scp zoo.cfg {USERNAME}@{NEW_SERVER5}:/{ZOO_HOME}/conf/

4. 进入2台到**新机器**，打开*zoo.cfg*文件，查看配置server项，一般情况server信息如下：

		server.1={ODL_SERVER1}:2888:3888
 		server.2={ODL_SERVER2}:2888:3888
		server.3={ODL_SERVER3}:2888:3888

5. 将2台新机器的id和ip、端口号等追加到文件末尾，则上述server信息修改为：

		server.1={ODL_SERVER1}:2888:3888
 		server.2={ODL_SERVER2}:2888:3888
		server.3={ODL_SERVER3}:2888:3888
		server.4={NEW_SERVER4}:2888:3888
		server.5={NEW_SERVER5}:2888:3888

	其中，4、5为新机器的id。

5. 在{NEW_SERVER4}、{NEW_SERVER5}上，进入{dataDir}目录，新建myid文件,将上面的id写入文件。

6. 依次连接到新机器{NEW_SERVER4}、{NEW_SERVER5}，进入bin文件夹下启动zookeeper。
		
		cd {ZOO_HOME}/bin
		./zkServer.sh start

7. 启动后使用命令```./zkServer.sh status```查看zookeeper状态。和**单机模式下不同**的是，这个时候，可以看到，**新机器的模式为follower**，而且没有出错。

8. 依次连接到**原机器**，将**新的zoo.cfg**文件拷贝到原机器上，也就是所有机器的配置文件**保持一致**。
9. **依次**连接到原机器{ODL_SERVER1}、{ODL_SERVER2}、{ODL_SERVER3}，对zookeeper进行重启。并且需要确保**一台机器启动成功后，再重启下一台机器。**

		cd {ZOO_HOME}/bin
		./zkServer.sh restart

### 大功告成

由于每个时刻只有一台机器会停止服务，所以整个集群中是有过半的机器正常工作的。所以整个集群在重启期间也是**可以正常对外工作**，对用户来说这个重启过程是无感知的。






