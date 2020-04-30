---
title: muduo源码阅读之Channel与Poller    
date: 2016-05-06 18:39:19
tags: muduo
---
如前所述，Reactor模式包括四个部分的组件:Handle, Synchronous Event Demultiplexer，Initiation Dispatcher和Event Handler。上一篇已经学习了muduo中的Initiation Dispatcher---EventLoop类，接下来分别讲述Handle---Channel类和Synchronous Event Demultiplexer---Poller类。 
<!--more-->           
### Channel
Channel类的功能类似于JAVA NIO的SelectableChannel和SelectionKey的组合。每个Channel对象自始至终只属于一个EventLoop，而根据muduo采用的one loop per thread模型，每个Channel对象都只属于某一个IO线程。每个Channel对象只负责一个文件描述符(fd)的IO事件分发，但它不拥有这个fd，也不会在析构的时候关闭这个fd。    
Channel在Event Handler与Dispatcher之间起到中介的作用。一方面可与Event Handler交互，Event Hanlder设置Channel的：    
1. 与I/O相关的文件描述符，即成员变量fd\_；     
2. 在当前Channel上感兴趣的事件(events\_)，在Event Handler中通过调用Channel的enableReading()、enableWriting()函数设置；   
3. 处理各种类型事件的回调函数，相应成员变量包括readCallback\_、writeCallback\_、errorCallback\_等，在Event Handler中通过调用Channel的接口函数setXXXCallback()等完成。       
相应的代码如下(部分)。

	
	class Channel : boost::noncopyable
	{
		public:
			...
			void handleEvent();
			void setReadCallback(const EventCallback& cb)
			{ readCallback_ = cb; }
			...
			void enableReading() 
			{ events_ |= kReadEvent; update(); }
			...
			int index() { return index_; }
			EventLoop* ownerLoop() { return loop_; }
		
		private:	
			void update();
			static const int kReadEvent;
			...
			EventLoop* loop_;
			int fd_;
			int events_;
			int revents_;
			int index_;
			EventCallback readCallback_;
			...
	}

另一方面，EventLoop与Channel通信，包括以下内容:      
1. Channel包含EventLoop类型的成员变量loop\_，Channel在成员函数update()函数中调用loop\_的成员函数updateChannel()，后者转而调用Poller::updateChannel()。   
2. 通知在当前Channel中就绪的事件。EventLoop通过Demultiplexer得到就绪Channel中发生的事件，并将其保存在成员变量revents\_中。    
3. Channel事件处理函数handleEvent()的调用。Channel::handleEvent()是Channel的核心，由EventLoop::loop()调用，Channel根据revents_的值调用Event Handler传入的相应回调函数。  
所以，Channel是Event Handler与Dispatcher之间的桥梁，二者不直接通信，而是分别与Channel交互。    
Channel::handleEvent()会调用Channel::handleEventWithGuard()，后者完成具体的事件处理过程。
	

	void Channel::handleEventWithGuard(Timestamp receiveTime)
	{
		...
		if((revents_ & POLLHUP) && !(revents_ & POLLIN))
		{
			if(closeCallback_) closeCallback_();
		}
		if(revents_ & (POLLIN | POLLPRI | POLLRDHUP))
		{
			if(readCallback_) readCallback_();
		}
		if(revents_ & POLLOUT)
		{
			if(writeCallback_) writeCallback_();
		}
		...
	}

函数中的POLLIN，POLLNVAL，POLLERR都是poll函数的测试条件，handleEventWithGuard()根据这些测试条件调用相应的回调函数。     
最后来看一下对感兴趣事件的设置，具体来说，设置当前Channel是否关系读、写事件的发生。通过update()函数或者将Channel插入Demultiplexer，或者更新Demultiplexer中相应Channel的状态。     
	

	const int Channel::kNoneEvent = 0;
	const int Channel::kReadEvent = POLLIN | POLLPRI;
	const int Chennel::kWriteEvent = POLLOUT;
	void enableReading() { events_ |= kReadEvent; update(); }
	...
	void diableAll() { events_ |= kNoneEvent; update(); }

### Poller
Poller类是IO多路复用的封装，在muduo中是个抽象基类，因为muduo同时支持poll()和epoll()两种IO复用机制，是muduo中少有的使用到继承的模块。Poller是EventLoop的间接成员，只供其owner EventLoop在IO线程调用，其生命期与EventLoop相等。Poller并不拥有Channel，Channel在析构之前必须自己unregister(EventLoop::removeChannel()),避免空悬指针。
Poller接口很简单，主要包括三个成员函数：      

 `1. virtual Timestamp poll(int timeoutMs, ChannelList* activeChannels) = 0；` 
   
poll()是Poller的核心功能，调用poll(), epoll()获得当前活动的IO事件，然后填充调用方传入的activeChannes，并返回poll()返回的时刻。


	Timestamp EPollPoller::poll(int timeoutMs, ChannelList* activeChannels)
	{
		int numEvents = ::epoll_wait(epollfd_,
							   &*events_.begin(),
                               static_cast<int>(events_.size()),
                               timeoutMs);
		...
		if(numEvents>0)
		{
			fillActiveChannels(numEvents, activeChannels);
		}
		...
	}
以epoll()实现为例，fillActiveChannels()遍历找出有活动事件的fd\_，把其对应的Channel填入activeChannels。 
      
`2. virtual void updateChannel(Channel* channel) = 0；`

Poller::updateChannel()的主要功能是负责维护和更新pollfds\_数组(以poll()实现为例)。添加新Channel的复杂度是O(logN)，原因是channels_对应的数据结构： `typedef std::map<int,Channel*> ChannelMap;`，更新已有的Channel的复杂度是O(1)，因为Channel记住了自己在pollfds\_数组中的下标，可以快速定位。  

`3. virtual void removeChannel(Channel* channel) = 0；` 

Channel的删除。同样，需要更改pollfds\_、channels\_数据成员。


### 总结
至此，Reactor模式中最核心的事件分发机制中的关键结构已讲述完毕，对应muduo中的EventLoop，Channel，Poller三个class。