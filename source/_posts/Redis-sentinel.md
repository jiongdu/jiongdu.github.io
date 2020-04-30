---
title: Redis Sentinel原理及实践
date: 2017-1-11 15:21:58
tags: Redis
---
Sentinel（哨兵）是Redis的高可用性解决方案：由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器，以及这些主服务器属下的多个从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器。   
<!--more-->
### Redis Sentinel系统结构
下图展示了一个Sentinel系统监视服务器的例子。当前系统包括主服务器server1，从服务器server2、server3、server4和Sentinel监听系统。
![](http://i.imgur.com/Q5Keiy2.png)   
假设某时刻server1进入下线状态，那么从服务器对主服务器的复制操作将中止，并且Sentinel系统会察觉到server1已下线，并在server1的下线时长超过用户设定的上限时，Sentinel系统会执行故障转移操作：    
（1）首先，Sentinel会挑选一个从服务器，将其升级为新的主服务器，这里涉及到选举算法；    
（2）然后Sentienl向server1所有的从服务器发送新的复制指令，让它们成为新的主服务器的从服务器；   
（3）Sentinel还会继续监视已下线的server1，并在它重新上线时，将它设置为新的服务器的从服务器。    
![](http://i.imgur.com/l2q4BV4.jpg)

### Sentinel与主从Redis通信细节
#### 获取主服务器信息  
Sentinel默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送INFO命令，并通过分析INFO命令的回复来获取主服务器的当前信息。    
通过分析主服务器返回的INFO命令回复，Sentinel可以获取以下两方面的信息：     
（1）主服务器本身的信息，包括服务器运行ID，服务器角色等；    
（2）主服务器属下的所有从服务器信息。包括从服务器的IP地址、端口号等，根据这些信息，Sentinel无须用户提供从服务器的地址信息，就可以自动发现从服务器。   
#### 获取从服务器信息
当Sentinel发现主服务器有新的从服务器出现时，Sentinel除了会为这个新的从服务器创建相应的实例结构之外，Sentinel还会创建连接到从服务器的命令连接和订阅连接。   
在创建命令连接之后，Sentinel在默认情况下，会以每十秒一次的频率通过命令向从服务器发送INFO命令。并收到服务器信息的回复，根据这些信息，对从服务器的实例结构进行更新。   
#### 与所有主从服务器通信
默认情况下，Sentinel会以每两秒一次的频率，通过命令连接向所有被监视的主服务器和从服务器发送命令。信息格式为：
	
	PUBLISH __sentinel__:hello <s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>

其中，以`s_`开头的参数是Sentinel本身的信息。而m_开头的参数则是主服务器的信息。      
另外，当Sentinel与一个主服务器或者从服务器建立起订阅连接之后，Sentinel就会通过订阅连接，向服务器发送命令： `SUBSCRIBE __sentinel__:hello`，Sentinel会在其与服务器之间的连接断开前一直订阅该频道。    
因此，每个与Sentinel连接的服务器，Sentinel既通过命令连接向服务器的 `__sentinel__:hello`频道发送信息，又通过订阅连接从服务器的`__sentinel__:hello`频道接收信息。    
比如，有sentinel1、sentinel2、sentinel3三个sentinel在监视同一个服务器，当sentinel
1向服务器的`__sentinel__:hello`频道发送一条信息时，所有订阅了该频道的sentine（包括自己）都会收到这条信息。这些信息用于更新其他Sentinel对发送信息Sentinel的认知，也会被用于更新其他Sentinel对监视服务器的认知。    
此

### 选举领头Sentinel
默认情况下，Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例发送PING命令，并通过实例返回的PING命令回复来判断实例是否在线。    
当Sentinel将一个主服务器判断为主观下线之后，为了确认这个主服务器是否真的下线，它会向同样监视这一主服务器的其他Sentinel进行询问。当从其他Sentinel那里接收到足够数量的已下线判断之后，Sentinel就会将从服务器判定为客观下线，并对主服务器进行故障转移操作。    
当一个主服务器被判断为客观下线时，监视这个下线服务器的各个Sentinel会进行协商，选举出一个领头Sentinel，并由领头Sentinel对下线主服务器执行故障转移操作（详细的选举算法这里不详述）。     
在选举出领头Sentinel之后，领头Sentinel将对已下线的主服务器执行文章开头所述的故障转移操作。
    
### Sentinel实践
以下实验均是基于ubuntu 14.04 x86-64bits平台，Redis通过apt-get install redis-server简易安装，版本2.8.x，主从服务器都位于本机。
#### 实验架构和目录结构
![](http://i.imgur.com/lXobqNP.png)   
![](http://i.imgur.com/6dgbBnr.png)
#### 搭建Reis主从结构
首先配置master对应的配置redis.conf，重点需要关注的地方有：

	pidfile /var/run/redis.pid
	port 7003
	tcp-keepalive 60
	logfile /var/log/redis/redis-server.log

其余很多选项保持默认即可。然后`redis-server master/redis.conf`启动master。      
然后搭建slave，slave的配置文件和master基本一致，只需要修改相应的pidfile、端口(8003)、日志文件名，并配上master的地址`slaveof 127.0.0.1:7003`
#### Sentinel配置
接下来配置三个哨兵。以sentinel1.conf为例

	port 26371

	daemonize yes
	logfile "./sentinel1.log"
	dir "/home/dujiong/redis-data/sentinel"

	sentinel monitor Master 127.0.0.1 7003 1
	sentinel down-after-milliseconds Master 1500
	sentinel failover-timeout Master 10000

	sentinel config-epoch Master 15

	sentinel known-slave Master 127.0.0.1 8003
	sentinel known-sentinel Master 127.0.0.1 26372 0aca3a57038e2907c8a07be2b3c0d15171e44da5
	sentinel known-sentinel Master 127.0.0.1 26373 e7625d74a5a4b142c495baa8ca522517bd08c65b
其余两个哨兵在此基础上修改相应的参数即可。   
配置好环境后，使用ps命令查看进程运行情况。    
![](http://i.imgur.com/QbnRZN8.png)
#### 查看Sentinel监控的主从服务器
在Sentinel中查看当前的主从服务器状态如下：   
![](http://i.imgur.com/EqCwUhI.png)
![](http://i.imgur.com/UtcunJV.png)
![](http://i.imgur.com/O2zmFTF.png)  、
#### 故障转移 
接下来，使用`redis-cli -p 7003 shutdown`关闭master，按照前面的理论，Sentinel将会进行故障转移操作，选择slave作为主服务器。   
![](http://i.imgur.com/tAqniY8.png)
最后，再次启动master服务器（127.0.0.1:7003），Sentinel将它设置为新的服务器的从服务器。   
![](http://i.imgur.com/rNhhyy6.png)

### 总结
Sentinel是一个运行在特殊模式下的Redis服务器，用以提供Redis的高可用解决方案。在3.0之后的Redis版本中，Redis Cluster重用了Sentinel的代码逻辑，不需要单独启动一个Sentinel集群，Cluster本身就能自动进行Master选举和Failover切换。   



	

	


