---
title: CentOS配置双网卡（Mellanox和传统以太网卡）
date: 2017-03-20 17:57:25
tags:  [network/linux,NVMe over RDMA]
---
刚给服务器装上了操作系统，期待已久的Mellanox网卡终于到了，关于该网卡这里就不做过多的介绍了，可以参考[Mellanox官网](http://www.mellanox.com/)，下面记录在服务器上Mellanox网卡和传统以太网的配置。
<!--more-->
### 安插Mellanox网卡并安装相应驱动
首先，在服务器背板安插Mellanox网卡，然后执行`ifconfig`查看是否多了一张新的网卡。初始状态下`ifconfig`看到的Mellanox网卡的名字是ib0，这说明该网卡的默认链路层协议时InfiniBand协议，而不是Ethernet协议。由于我们后续需要使用NVMe over RoCEv2，所以需要将Mellanox的链路层协议改为Ethernet。因此，需要先下载并安装相应的驱动。    
在官网上确定好操作系统对应的驱动版本号（比如CentOS7.2对应的版本是MLNX\_OFED\-3.4-2.0.0.0-rhel7.2-x86\_64）后，下载并安装驱动。
	
	wget http://www.mellanox.com/downloads/ofed/MLNX_OFED-3.4-2.0.0.0/MLNX_OFED_LINUX-3.4-2.0.0.0-rhel7.2-x86_64.tgz
	tar -xvf MLNX_OFED_LINUX-3.4-2.0.0.0-rhel7.2-x86_64.tgz
    cd MLNX_OFED_LINUX-3.4-2.0.0.0-rhel7.2-x86_64/
    ./mlnxofedinstall  (--add-kernel-support)       ###kernel 4.8.15可能需要添加特定的选项
	/etc/init.d/openibd restart

### 修改Mellanox网卡配置
如前文所述，Mellanox网卡的默认链路层协议是Infiniband协议，不是Ethernet协议。所以，接下来修改网卡的链路层协议。       
在修改之前，可以先通过`lspci | grep Mellanox`查看网卡信息，确定是否为InfiniBand协议。

	mst start
	ibv_devinfo | grep vendor_part_id	##得到vendor_part_id(我这里是4115)
	mlxconfig -d /dev/mst/mt4115_pciconf0 q  ##再次查询网卡信息
	mlxconfig -d /dev/mst/mt4115_pciconf0 set LINK_TYPE_P1=2  ##修改为Ethernet协议

修改并重新启动后，可以看到网卡的名称发生了变化（这里是enp133s0）。最后，使用`lspci | grep Mellanox`确认修改。

### CentOS双网卡配置
现在服务器的网卡连接情况是一张Mellanox网卡通过专用网线连接到EdgeCore 100G白牌交换机上，另外一张传统以太网卡连接到以太网交换机（192.168.1.1）。接下来双网卡配置双IP，一个192.168.1.x，可通过以太网交换机连接外网，另一个192.168.5.x，连接到100G白牌交换机。   

以太网卡在服务器上对应的名称为eno1。修改其配置文件。

	vim /etc/sysconfig/network-scripts/ifcfg-eth0

修改和添加以下内容：
	
	BOOTPROTO=static
	HWADDR=6c:92:bf:42:27:2e	#mac address
	IPADDR=192.168.1.152
	NETMASK=255.255.255.0
	GATEWAY=192.168.1.1
	ONBOOT=yes

修改后的配置文件如下图所示。    
![](http://i.imgur.com/WqcbomA.png)   
通过`service network restart`重启网络后，就成功的将该以太网卡设置IP设置成了静态IP（192.168.1.152）。

Mellanox网卡的配置和以太网卡一样。所以，这里只列举出修改后的配置文件（/etc/sysconfig/network-scripts/ifcfg-enp133s0）     
![](http://i.imgur.com/jcLC6h9.png)

配置好后的服务器网络接口情况如下图所示。
![](http://i.imgur.com/rNo8Xe3.png)
 
### 配置路由表
目前系统默认网关是192.168.1.1，所以需要增加两个路由表，实现双网关正常访问。      
	vim /etc/iproute2/rt_tables
增加两行内容：
	252 net2
	251 net3
在/etc/rc.local添加静态路由规则。

	ip route flush table net2
	ip route add default via 192.168.1.1 dev eno1 src 192.168.1.152 table net2
	ip rule add from 192.168.1.152 table net2

	ip route flush table net3
	ip route add default via 192.168.5.1 dev enp133s0 src 192.168.5.86 table net3
	ip rule add from 192.168.5.86 table net3

这时，双网卡双IP应该就配置好了，`service network restart`重启网络就可以实现不同网卡对应不同网络访问。


	
    
