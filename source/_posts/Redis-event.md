---
title: Redis之事件驱动
date: 2016-11-5 12:03:53
tags: Redis
---
事件处理是Redis的核心机制之一，通过在文件、网络和时间等类型的事件上进行多路复用，为Redis的高性能提供保证。事件驱动在Redis源码中主要涉及ae.h/ae.c，Ae_evport/epoll/kqueue/select.c等文件。
<!--more-->
### Redis事件驱动相关数据结构
Redis事件驱动内部包含四个主要的数据结构(定义在ae.h中)，分别是：文件事件结构体、时间事件结构体、就绪事件结构体和循环结构体。
#### 文件事件

文件事件定义在ae.h/aeFileEvent结构体中。
	
	typedef struct aeFileEvent {
		int mask;			//状态

		aeFileProc* rfileProc;		
		aeFileProc* wfileProc;
		void* clientData;	//多路复用库的私有数据
	} aeFileEvent;
其中，rfileProc和wfileProc为aeFileProc类型函数指针，分别对应于读、写事件处理函数。其声明为：    
	
	typedef void aeFileProc(struct aeEventLoop *eventLoop, int fd, void *clientData, int mask);
clientData保存执行命令时所需要的数据，每次处理事件时，它会作为参数被传入事件处理器。
#### 就绪事件
就绪事件定义在ae.h/aeFiredEvent结构体中。

	typedef struct aeFiredEvent {
		int fd;				//就绪文件描述符
	
		int mask;			//事件类型掩码
	} aeFiredEvent;
就绪事件结构体将在IO多路复用（aeApiPoll函数）返回时设置。
#### 时间事件
时间事件定义在ae.h/aeTimeEvent结构体中。

	typedef struct aeTimeEvent {
		long long id;		//时间事件的唯一标识
		
		long when_sec;
		long when_ms;
		
		aeTimeProc* timeProc;
		aeEventFinalizerProc* finalizerProc;	 
		void* clientData;
		struct aeTimeEvent* next;		//指向下个时间事件结构，形成链表
	}
其中，id为时间事件的全局唯一标识号，其值按从小到大的顺序递增，新事件的id号比旧事件的id大。when为毫秒级的UNIX时间戳，记录时间事件的到达时间。next为指向下个时间事件结构的指针，Redis服务器将所有时间事件都放在一个无序（到达时间）链表中，每当时间事件执行器运行时，就遍历链表，查找所有已到达的时间事件，并调用相应的事件处理器。注意，Redis的时间事件链表是按id排序的链表，对于到达时间，它是无序的。      
timeProc和finalizerProc分别为时间事件处理器和事件释放函数的函数指针。其声明为：
	
	typedef int aeTimeProc(struct aeEventLoop *eventLoop, long long id, void *clientData);
	typedef void aeEventFinalizerProc(struct aeEventLoop *eventLoop, void *clientData);
此外，Redis的时间事件可以分为以下两类：   
（1）定时事件：让一段程序在指定的时间之后执行一次；      
（2）周期性事件：让一段程序每隔指定的时间就执行一次。      
一个时间事件是定时事件还是周期性事件取决于时间事件处理器的返回值(ae.c/processTimeEvents/retval = te->timeProc(eventLoop, id, te->clientData))：    
（1）如果事件处理器返回返回ae.h/AE\_NOMORE,那么这个事件为定时事件，该事件在到达一次之后就会被删除。     
（2）如果事件处理器返回一个非AE\_NOMORE的整数值，那么这个事件为周期性事件，此时，服务器会对时间事件的when属性进行更新，让该事件在一段时间之后再次到达，并以这种方式更新、运行。   

#### 事件循环结构体
在事件驱动的实现中，需要有一个事件循环结构来监控调度所有的事件。在Redis中的事件驱动库中，事件循环结构结构定义在ae.h/aeEventLoop中。

	typedef struct aeEventLoop {
		int maxfd;					//注册的最大的文件描述符
		int setsize;				//监听的文件描述符的最大个数
		
		long long timeEventNextId;	//生成时间事件id
		time_t lastTime;			//最后一次执行时间事件的时间
		
		aeFileEvent* events;		//注册的文件事件
		aeFilEvent* fired;			//就绪的文件事件
		aeTimeEvent* timeEventHead;	//时间事件
		int stop;
		void* apindata;				//多路复用库的私有数据
		aeBeforeSleepProc* beforesleep;
	} aeEventLoop;
其中，events是aeFileEvent结构的数组，每个aeFileEvent结构表示一个注册的文件事件。setsize表示能处理的文件描述符的最大个数。beforesleep函数指针表示在监控事件触发之前，需要调用的函数。apindata表示底层多路复用的私有数据，比如对于select来说，该结构保存了读写描述符数组；对于epoll来说，该结构中保存了epoll描述符和epoll\_event数组。       

### Redis事件的调度与执行
Redis的事件处理主循环在aeMain函数中进行：
	
	void aeMain(aeEventLoop* eventLoop){
		eventLoop->stop = 0;
		while(!eventLoop->stop){
			if(eventLoop->beforesleep != NULL){
				eventLoop->beforesleep(eventLoop);
			}
			aeProessEvents(eventLoop, AE_ALL_EVENTS);
		}
	}
该事件循环的流程图如下：
![](http://i.imgur.com/5bUY0j1.png)  
aeProcessEvents函数进行具体的事件处理，定义在ae.c文件下，其关键代码如下：

	int aeProcessEvents(aeEventLoop* eventLoop, int flags) {
		...
		if(eventLoop->maxfd!=-1 || ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
			shortest = aeSearchNearestTimer(eventLoop);  //获取最近的时间事件
			if(shortest){
				aeGetTime(&now_sec, &now_ms);
				//计算最近的时间事件距离到达还有多少毫秒
				tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
            	tvp->tv_sec --;
				...
			}else{		//没有时间事件，根据AE_DONT_WAIT来设置是否阻塞，以及阻塞的事件长度
				if(flags & AE_DONT_WAIT) {		
					tv.tv_sec = tv.tv_usec = 0;
					tvp = &tv;
				}else{
					tvp = NULL;
				}
			}
			numevents = aeApiPoll(eventLoop, tvp);			//select,epoll...
			for(j=0; j<numevents;j++){
				aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
				int mask = eventLoop->fired[j].mask;
				int fd = eventLoop->fired[j].fd;
				int rfired = 0;
				if(fe->mask & mask & AE_READABLE){
					rfired = 1;
					fe->rfileProc(eventLoop, fd, fe->clientData, mask);		//处理读事件
				}
			...
			}
		}
		if(flags & AE_TIME_EVENTS)
			processed += processTimeEvents(eventLoop);		//处理时间事件
		return processed;
	}

可以看出，aeApiPoll函数的最大阻塞时间(tvp结构)由到达时间最接近当前时间的时间事件(shortest)决定，这样既可以避免服务器对时间事件进行频繁的轮询（忙等待），也可以确保aeApiPoll函数不会阻塞过长时间。      
因为文件事件是随机出现的，如果等待并处理完一次文件事件之后，仍未有任何时间事件到来，那么服务器将再次等待并处理文件事件。随着文件事件的不断执行，时间逐渐向时间事件所设置的到达时间逼近，并最终来到到达时间，这时服务器就可以开始处理到达的时间事件了。     
对文件事件和时间事件的处理都是同步、有序、原子地执行的，服务器不会中途中断事件处理，也不会对事件进行抢占，因此，不管是文件事件的处理器，还是时间事件的处理器，它们都会尽可能地减少程序的阻塞时间，并在需要时让出执行权，从而降低造成事件饥饿的可能性。比如说，在命令回复处理器将一个命令回复写入到客户端套接字，如果写入的字节数超过了一个预设常量的话，命令回复处理器就会主动用break跳出写入循环，将余下的数据留到下次再写，另外，时间事件也会将非常耗时的持久化操作放到子线程或子进程中执行。     
此外，由于时间事件放在文件事件之后执行，并且事件之间不会出现抢占，所以时间事件的实际处理时间，通常会比时间事件设定的到达时间稍晚一些。   
### Redis IO复用函数选择
最后提到的一点是，Redis提供了在多个IO库之间选择最佳的策略。

	#ifdef HAVE_EVPORT
	#include "ae_evport.c"
	#else
    	#ifdef HAVE_EPOLL
    	#include "ae_epoll.c"
    	#else
        	#ifdef HAVE_KQUEUE
        	#include "ae_kqueue.c"
        	#else
        	#include "ae_select.c"
        	#endif
    	#endif
	#endif
这样，在编译时，Redis可以根据平台自上而下选择最快的IO多路复用函数。
