---
title: wifi关键数据提取
date: 2016-04-20 09:56:09
tags: openwrt/wifi
---

本文介绍并实现了使用无线网卡获取周围wifi关键通信数据，捕捉所需要的MAC层管理数据包，通过解包获取到周围AP和终端设备的MAC地址、RSSI等关键数据，并存入数据库中，可用于定位，通信链路分析等。

<!--more-->

### 基本原理
利用工作在monitor模式的无线网卡可以探测到所有经过它的数据流，在AP端使用无线网卡抓取AP与AP、AP与STA之间的MAC层管理数据包，然后对抓取的MAC帧进行实时解包，提取所需要的Timestamp、MAC address、RSSI等信息，并将其存入服务器数据库中，用于后续的wifi定位、链路分析等。

### 实现方法
#### 设置无线网卡工作在monitor模式
如上所述，首先需要设置无线网卡工作在monitor模式，使用shell脚本。
	
	#!/bin/bash
	ifconfig wlan1 down
	iwconfig wlan1 mode monitor
	ifconfig wlan1 up
	iwconfig wlan1 channel 11

#### 数据提取		
在抓包的实现中，借鉴了Openwrt中的iwcap工具包，iwcap是Openwrt上用来测试wifi功能的一个轻量级的抓包工具，可以抓取探测到的所有802.11帧，并获取到pcap格式的网卡原始数据。	    
本方案在iwcap的基础上做了一些改进，主要体现在：     
（1）进行帧过滤，只抓取特定的一些帧如Beacon、Probe Request、Qos Data、 Null Function等，即可提供足够的数据；    
（2）进行实时解包，在获取到帧的时候，提取所需要的关键数据，过滤掉不关注的信息；    
（3）添加了数据库功能，对获取到的关键数据进行存储，用于后续的定位、通信链路分析等。       

### 程序分析
完整程序见: [github](https://github.com/jiongdu/WiFi-Location)  
程序代码结构如下图所示：    
![](http://i.imgur.com/P3IuW9R.png)   
main.c为原iwcap抓取数据包的代码，做了部分改进。   
pcap.c/pcap.h为解包、提取关键数据的操作。     
db.c/db.h为数据库相关操作。     
commad为上述设置网卡monitor脚本。    
period为周期性切换网卡工作频率脚本。   
内层Makefile用于生成可执行程序。             
外层Makefile用于在Openwrt环境下生成工具包。      
 
### 程序执行
在ubuntu 14.04或Openwrt平台运行该程序，服务端接收到的数据如下图所示：       
![](http://i.imgur.com/LS5lcxZ.png)





