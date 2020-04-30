---
title: muduo源码阅读之Reactor模式和muduo  
date: 2016-04-25 18:56:03
tags: muduo
---

muduo是一个基于Reactor模式的非阻塞网络库，常见的使用Reactor模式的网络库（框架）还有libevent,Java Netty等，所以，首先需要弄清楚Reactor模式。

<!--more-->

### Reactor的事件处理机制
首先回忆一下普通函数调用的机制：程序调用某函数执行，程序等待，函数将结果和控制权返回给程序，程序继续处理。这是一种顺序的处理方式。      
Reactor常释义“反应堆”，是一种事件驱动机制，和普通函数调用的不同之处在于：应用程度不是主动的调用某个接口API完成处理，恰恰相反，Reactor逆置了事件处理流程，应用程序需要提供相应的接口并注册到Reactor上，如果关注的事件发生，Reactor将主动调用应用程序注册的接口，这些接口有一个熟悉的名字：“回调函数”。所以，用户使用Reactor模式的网络库向其注册相应的事件和回调函数，当事件发生时，就会调用回调函数处理事件（I/O读写，定时器和信号等）。Reactor模式一般与非阻塞I/O结合，在处理一个事件时不用等待事件的完成，而是转去处理其他的任务，当事件处理完毕后再去通知Reactor。       
所以，Reactor模式具有很多优点：    
1）响应快，不必为单个同步时间所阻塞，虽然Reactor本身依然是同步的；    
2）编程相对简单，可以最大程度的避免复杂的多线程及同步问题，并且避免了多线程/进程的切换开销；     
3）可扩展性，可以方便的通过增加Reactor实例个数来充分利用CPU资源；
4）可复用性，Reactor框架本身与具体事件处理逻辑无关，具有很高的复用性；

### Reactor模式的结构
Reactor模式包括四个部分的组件:    
1）事件源（Handle）       
用于表示通过操作系统管理的各种资源，包括套接字，打开文件，定时器等，在Unix系统中是文件描述符，在Windows系统中是句柄。当关心的事件在其上面发生时，可以通过多路复用器(select/epoll等)得到。    
2）Synchronous Event Demultiplexer    
事件多路分发机制。由操作系统提供的I/O多路复用机制，如select/epoll等。程序首先将关心的事件源注册到event demultiplexer上，当有事件到达时，event demultiplexer会发出通知“在已经注册的句柄集中，一个或多个句柄已经就绪”。        
3）Initiation Dispatcher
事件管理的接口，实现事件的注册，删除，分发的接口，并运行事件循环，是用于驱动的主模块。Synchronous Event Demultiplexer也是它的一个组件，用于等待新事件的发生。当有事件发生时，Demultiplexer通知dispatcher，dispatcher再根据事件的类型使用相应的Event Hander完成事件的处理。     
3）Event Handler
事件处理程序。一般来说会根据事件类型的不同实现各种类型的回调函数。

下图可详细表达各个模块和它们之间的关系：
![](http://i.imgur.com/052r16x.png)

### muduo中的Reactor模式
Reactor模式的各个组件在muduo中都有对应的部分，虽然在具体的实现和逻辑上有其自身的特点和不同，但其内在的组织结构是高度符合Reactor模式的。   
muduo的类图如下：                                                                                    
  
![](http://i.imgur.com/M4uVBf1.png)  
各个模块与Reactor模式各组件的对应关系（功能上）是    
1. Initialization dispatcher —— EventLoop    
2. Synchronous Event Demultiplexer —— Poller     
3. Handles —— FileDescriptor、Channel    
4. Event Handler —— TcpConnection、Acceptor、Connector


### muduo中的思想
有一个main Reactor，这个main Reactor只管接收新的连接，一旦创建好连接，就从EventLoopThreadPool（线程池）中选择一个合适的EventLoop来托管这个连接套接字。这个EventLoop就是一个sub Reactor。       
muduo的线程模型是one loop per thread + thread pool模型。每个线程最多有一个EventLoop，每个TcpConnection必须归某个EventLoop管理，所有的I/O会转移到这个线程。即，一个句柄只能由一个线程读写。    
TcpServer直接支持支持多线程，有两种模式：    
1）单线程，accept()与TcpConnection用同一个线程做IO；      
2）多线程，accept()与EventLoop在同一个线程，另外创建一个EventLoopThreadPool，新的连接会分配到线程池中。

