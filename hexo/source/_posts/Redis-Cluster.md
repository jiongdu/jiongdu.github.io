---
title: 搭建Redis集群
date: 2017-1-15 15:08:05
tags: Redis
---

Redis集群是Redis提供的分布式数据库方案，集群通过分片来进行数据共享，并提供复制和故障转移功能。    
<!--more-->
### Redis集群概述
Redis集群使用数据分片而非一致性哈希来实现，一个Redis集群包含16384个哈希槽（slot），数据库中的每个键都属于这16384个哈希槽中的其中一个，集群中的每个节点可以处理0个或最多16384个槽，当数据库中的16384个槽都有节点在处理时，集群处于上线状态；而如果数据库中有任何一个槽没有得到处理，那么集群处于下线状态。        
集群中的每个节点负责处理一部分哈希槽，比如，现在有三个独立的节点127.0.0.1:7000,127.0.0.1:7001,127.0.0.1:7002，其中各节点处理的哈希槽关系：节点7000负责处理0到5000号哈希槽，节点7001负责处理5001到10000号哈希槽，节点7002负责处理10001到16384号哈希槽。      
从而，当向集群中添加或删除节点时，集群只需在对应节点的哈希槽做移动即可。不会造成节点阻塞、集群下线。    
当然，为了使得集群在一部分节点下线的情况下仍然可以正常运作，Redis集群对节点提供了主从复制功能，集群中的每个节点都有1到N个复制节点，形成主-从模型。

### Redis集群架构图
Redis集群架构如下图所示：
![](http://i.imgur.com/O4QfdDF.jpg)    
Redis集群架构的主要特点有:
（1）所有节点批次互联（PING-PONG机制），没有中心控制协调节点，内部使用二进制协议优化传输速度和带宽；     
（2）节点的失效通过集群中超过半数的节点“投票”监测；        
（3）客户端与Redis节点直连，不需要中间proxy层，客户端连接集群中任何一个可用节点即可；    

### Redis集群实践
下面搭建一个简单的Redis集群环境，集群共包含6个节点，其中3个主节点，3个从节点。节点都部署在本机（ubuntu 14.04 x86-64），以端口号区分，分别为127.0.0.1:7000~127.0.0.1:7005。    
1.首先安装3.0版本之后的Redis（因为Redis集群是在3.0版本提出的功能），这里采用源码(3.2.8版本)安装。

	wget http://download.redis.io/releases/redis-3.2.8.tar.gz

2.然后解压，安装
	
	tar xf redis-3.2.8.tar.gz    
	cd redis-3.2.8       
	make && make install

3.安装完成后查看Redis版本信息，验证安装是否成功。    
![](http://i.imgur.com/6QuQXPQ.png)

4.创建存放多个节点实例的目录

	mkdir redis-data/cluster -p
	cd redis-data/cluster
	mkdir 7000 7001 7002 7003 7004 7005

5.复制并修改配置文件redis.conf

	cp /etc/redis/redis.conf redis-data/cluster/7000

修改配置文件中下面选项
	
	port 7000
	daemonize yes
	cluster-enabled yes
	cluster-config-file nodes.conf
	cluster-node-timeout 5000
	appendonly yes

然后把修改完成的redis.conf复制到7001-7005目录下，端口修改成和文件夹对应。 
	  
6.分别启动6个Redis实例

	cd redis-data/cluster/7000
	redis-server redis.conf
	...
	cd redis-data/cluster/7001
	redis-server redis.conf

7.查看进程是否运行
![](http://i.imgur.com/nWUeGum.png)

8.接下来创建集群，首先安装依赖包

	apt-get install ruby

然后安装gem-redis，下载地址https://rubygems.org/gems/redis/versions/3.3.0

	gem install redis-3.3.0.gem

接下来复制集群管理程序到/usr/local/bin

	cp redis-3.2.8/src/redis-trib.rb /usr/local/bin/redis-trib

就可以使用redis-trib创建集群了。

	redis-trib create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005

其中，命令create表示创建一个新的集群，选项--replicas 1表示未集群中的每个主节点创建一个从节点。即，上述命令运行后，redis-trib将创建一个包含三个主节点和三个从节点的集群。 
![](http://i.imgur.com/6jhul4S.png)   
键入“yes”后一个三主三从的集群架构就创建完毕。
![](http://i.imgur.com/WTTZPh8.png)

### 一些简单的测试
创建完毕后，可以随时通过redis-cli -p 7000(node number) cluster nodes来查看集群中各节点的信息。包括唯一的节点ID，主从关系，每个主节点分配的slots范围。 
![](http://i.imgur.com/jJcOqwB.png)
#### 集群查询
做一个简单的测试，	在7000节点上存储K-V数据，然后分别在其从节点和其他主节点上获取该数据。
![](http://i.imgur.com/X3GWKHJ.png)    
![](http://i.imgur.com/a0P55nA.png)  
要找的数据没有在当前节点上时，cluster发送MOVED指令指示到对应的节点上穿，可以通过-c参数指定查询时接收到MOVED指令自动跳转。

#### 删除节点
Slave节点的作用是提供冗余和高可用，大部分情况下用于分担读的压力。移除Slave节点无须多余的动作，直接删除即可。    
![](http://i.imgur.com/Hr2xMz6.png)   
而如果是Master节点，则需要将其负责的slots范围迁移到其他节点上。保证数据不丢失。   
所以，先使用redis-trib reshard ...来将待删除的节点上的数据迁移到其他节点，然后删除该节点。    
![](http://i.imgur.com/xLG6ctH.png)

### 总结
Redis Cluster是3.0版本之后提供的新功能，采用了P2P的去中心化架构，而没有采用像Codis之类的Proxy解决方案中的中心协调节点设计。本文只是简单搭建了一个Redis集群环境，后续还将在此基础上，进一步深入研究高可用、可扩展的Redis集群方案。

