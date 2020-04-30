---
title: 配置OpenWrt路由器为Dumb AP
date: 2016-03-08 09:56:09
tags: openwrt/wifi
---

## Dumb AP
Dumb AP，简单说，就是将路由器作为一个纯接入点，没有路由转发，没有DHCP。这时的路由器相当于一台二层交换机，没有三层功能。所以，实验环境中，将Dumb AP连接在上级路由器下，子网段为192.168.1.1/24。
<!--more-->
## 配置Dumb AP 
### 修改网络配置文件(/etc/config/network)
修改OpenWrt的网络配置文件，将wan口和lan口桥接起来:
`config interface lan`
	`option type 'bridge'`
	`option ifname 'eth0.1 eth0.2'`  ##将二者桥接
	`option proto static`
	`option ipaddr '192.168.1.196'`  ##采用静态ip
	`option netmask 255.255.255.0` 	
然后注释掉路由器关于wan口的设置，包括ipv4和ipv6。
需要说明的是，上述的配置文件中桥接的而是eth0.1和eth0.2，但事实上路由器的接口不尽相同，比如有的wan口事实上是eth1。所以，需要因地制宜。
### 关掉DHCP
因为这里将路由器作为Dumb AP使用，作为一个纯无线接入点和交换机使用，不再需要其DHCP功能，所以关掉DHCP。   
可以通过uci或者是修改配置文件(/etc/config/dhcp)设置DHCP。这里采用的是后者，即注释掉文件中lan口dhcp配置相关的设置。
`#config dhcp 'lan'`
	`#option interface 'lan'`
	`#option dhcpv6 'server'`
	`#option ra 'server'`
	`#option ignore '1'`
	`#option ra_management '1'` 
### 关掉防火墙
某些情况下，需要关闭防火墙，同样的，修改配置文件/etc/config/firewall，将相应的REJECT改成ACCEPT即可，具体不再详述。
最后，载入新的配置即可。

## 附：完整的/etc/config/network文件
`config interface 'loopback'`
`option ifname 'lo'`
`option proto 'static'`
`option ipaddr '127.0.0.1'`
`option netmask '255.0.0.0'`

`config globals 'globals'`
`option ula_prefix 'fd9f:91f8:3d14::/48'`

`config interface 'lan'`
`option ifname 'eth0.1 eth0.2'`
`option force_link '1'`
`option macaddr 'b0:68:b6:ff:d6:b8'`
`option type 'bridge'`
`option proto 'static'`
`option ipaddr '192.168.1.196'`
`option netmask '255.255.255.0'`
`option ip6assign '60'`

`#config interface 'wan'`
`#option ifname 'eth0.2'`
`#option force_link '1'`
`#option macaddr 'b0:68:b6:ff:d6:b9'`
`#option proto 'dhcp'`

`#config interface 'wan6'`
`#option ifname 'eth0.2'`
`#option proto 'dhcpv6'`

`以下均保持原状`
`config switch`
`option name 'switch0'`
`option reset '1'`
`option enable_vlan '1'`
`config switch_vlan`
`option device 'switch0'`
`option vlan '1'`
`option ports '0 1 2 3 6t'`
`config switch_vlan`
`option device 'switch0'`
`option vlan '2'`
`option ports '4 6t`
  
