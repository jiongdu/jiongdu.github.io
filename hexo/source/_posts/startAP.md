---
title: OpenWrt设置程序开机自启动
date: 2016-03-18 17:05:23
tags: openwrt/wifi
---
### OpenWrt启动脚本
需要在OpenWrt中-将自己的程序设置为开机自启动。虽然OpenWrt是基于Linux的嵌入式发行版，但是和其设置方法还是略有差异，在此做一份记录。参考：[http://wiki.openwrt.org/doc/techref/initscripts](http://wiki.openwrt.org/doc/techref/initscripts)    
OpenWrt的启动脚本在/etc/init.d/目录下，而系统开机时自动运行/etc/rc.d/目录下的脚本，所以在rc.d目录下，有init.d脚本的连接文件。
<!--more-->
### 编写自己的启动脚本
按照以下结构编写自己的shell脚本(这里以我的startAP为例说明)。特别注意一下，OpenWrt中的shell解析器与常用的Linux桌面、服务器（bash）不一样，记得当时写的时候就用的是#/bin/bash，后面找了好久才发现。

    #!/bin/sh /etc/rc.common
    #/init.d/startAP
    START=50
    start()
    {
    	...
    }
	stop()
	{
		...
	}
在start()中写入需要开机运行的程序命令，在stop()中写入终止程序的命令。START=50是指优先级，数字越大，优先级越低。一般优先级高的脚本会先运行。
编写好自己的程序启动脚本后，熟悉Linux的都知道，要让程序执行，需要给脚本赋予可执行权限。所以，运行命令chmod+x xxx。
### 为启动脚本做一个软链接
如上所述，系统启动时会按顺序自动运行/etc/rc.d/目录下的脚本链接，对应执行/etc/init.d/目录下的启动脚本。所以，需要在/etc/rc.d/下为启动脚本创建一个链接。注意，链接文件要命名要规范，在脚本名前加S+启动顺序数字。顺便提一句，这里的启动顺序数字和前面所说到的优先级可是两码事。  
所以，执行命令ln -s /etc/init.d/startAP /etc/rc.d/S95startAP创建链接。   
最后，重启，就可以开机启动程序了。不妨使用ps查看一下吧。
