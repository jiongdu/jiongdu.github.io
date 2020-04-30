---
title: muduo源码阅读之EventLoop类
date: 2016-05-03 21:13:47
tags: muduo
---
EventLoop类似于前面所述的Reactor模式中的Initiation Dispatcher，是用于驱动的主模块，完成与其他模块的调用和交互。EventLoop类提供的主要是一个框架，事件的分发由Demultiplexer提供，事件的处理由Event Handler提供，但事件的循环、怎样将事件的分发与调用结合起来则是由EventLoop类决定的。
<!--more-->  
### EventLoop继承自boost::noncopyale
EventLoop是不可拷贝的，所以源码中继承了boost::noncopyable。boost::noncopyable的思想是把构造函数和析构函数设置成protected权限，这样子类可以调用，但是外面的类不能调用，那么当子类需要定义构造函数的时候不至于通不过编译。但是更关键的是noncopyable把复制构造函数和复制赋值函数都做成了private，这就意味着除非子类定义自己的复制构造和赋值函数，否则在子类没有定义的情况下，外面的调用者是不能够通过复制构造和赋值等手段来产生一个新的子类对象的。   
one loop per thread模型意味着每个线程只能有一个EventLoop对象，因此EventLoop的构造函数会检查当前线程是否已经创建了其他EventLoop对象。EventLoop的构造函数会记住本对象所属的线程。

	EventLoop::EventLoop()	
		:looping(false),
		 threadId_(CurrentThread::tid())
		 ...
	{
		if(t_loopInthisThread)
			LOG_FATAL << ...
	}
		
### Event Hander的注册与删除     
在初始状态，一个EventLoop只是一个框架，其中并没有包含具体需要处理的事件及相应的资源。所以，要处理一个事件，必须要将相应的Handler信息注册到EventLoop中，muduo中，这个用于注册的函数就是EventLoop::updateChannel()。

    void EventLoop::updateChannel(Channel* channel)
	{
		assert(channel_->ownerLoop() == this);
		asserInLoopThread();
		poller_->updateChannel(channel);
	}
函数形参为Channel*类型，如前所述，Channel类似Reactor模式中的Handle。通过Channel类型的变量，Event Handler的相关信息被注册到了EventLoop中。具体的注册过程由形参为poller_成员变量完成，这是muduo的多路复用机制。Poller在muduo中是个抽象基类，因为muduo同时支持poll()和epoll()两种IO复用机制。Poller::updateChannel()的主要功能就是负责维护和更新pollfds数组。                                                                                                                                

    void EventLoop::removeChannel(Channel* channel)
	{
		...
		poller_->removeChannel(channel);
	}
同样，Event Handler的删除与注册很相似，具体过程也通过poller_完成。

### 事件循环中的事件分发
loop函数是EventLoop类的核心，用于事件循环的运行。在循环返回时，通过Demultiplexer获得已注册到EventLoop中Event Handler的就绪通知。然后Demultiplexer将所有就绪的事件Channel保存到容器activeChannels_中，接着一一处理其中所有activeChannel的callback。EventLoop只提供框架，对要处理的事件一无所知，Channel中包含感兴趣的事件和对事件的处理方法(注册的回调函数)。    


    void EventLoop::loop()
	{
		while(!quit)
		{
			pollReturnTime=poller_->poll(kPollTimeMs, &activeChannels_);
			for(ChannelList::iterator it=activeChannels_.begin();
					it!=activeChannels_.end();++it)
			{
				currentActiveChannel_ = *it;
				currentActiveChannel_->handleEvent(pollReturnTime_);
			}
			doPendingFunctors();
		}
	}

EventLoop在它的IO线程里执行某个用户任务回调，即EventLoop::runInLoop(const Functor& cb),如果用户在当前IO线程调用这个函数，回调会同步进行，如果用户在其他线程调用runInLoop()，cb会被加入队列(queueInLoop(std::move(cb)))，IO线程会被唤醒来调用Functor。唤醒IO线程传统的方法是用pipe()，IO线程始终监视此管道的readable事件，在需要唤醒的时候，其他线程往管道里写一个字节，这样IO线程就从IO复用阻塞调用中返回。而muudo中使用了Linux新增的eventfd，可以更高效的唤醒。

    void EventLoop::runInLoop(Functor&& cb)
	{
		if(isInLoopThread())
		{
			cb();
		}
		else
		{
			queueInLoop(std::move(cb));
		}
	}
    void EventLoop::queueInLoop(Functor&& cb)
	{
		{
			MutexLockGuard lock(mutex_);
			pendingFunctors_.push_back(std::move(cb));
		}
		if(!isInLoopThread() || callingPendingFunctors_)
		{
			wakeup();
		}
	}

### 几个定时运行函数
muduo EventLoop中有三个定时函数。runAt(...)指在指定的时间调用TimerCallback, runAfter(...)指等一段时间调用TimerCallback, runEvery(...)指以固定时间间隔反复调用TimerCallbck。

	TimerId EventLoop::runAt(const Timestamp& time, const TimerCallback& cb)
	{
 		 return timerQueue_->addTimer(cb, time, 0.0);
	}

	TimerId EventLoop::runAfter(double delay, const TimerCallback& cb)
	{
 	 	 Timestamp time(addTime(Timestamp::now(), delay));
  		 return runAt(time, cb);
	}

	TimerId EventLoop::runEvery(double interval, const TimerCallback& cb)
	{
  		 Timestamp time(addTime(Timestamp::now(), interval));
  		 return timerQueue_->addTimer(cb, time, interval);
	}
muduo中的TimerQueue的公有接口很简单，只有两个函数addTimer()和cancel()，TimerQueue使用二叉搜索树，以pair<Timestamp,Timer*>为key(即使两个Timer到期时间相同，地址也必定不同)，能快速根据当前时间找到已到期的定时器，也要高效的添加和删除Timer。TimerQueue关注最早的定时器，getExpired返回所有的超时定时器列表，使用stl算法lower_bound()返回第一个值>=超时定时器的迭代器。

### 在事件循环中运行回调函数
前面已经叙述过，EventLoop::runInLoop()用于在一个事件循环中运行回调函数。如果事件循环在当前线程中，那么可以直接运行回调函数。但如果不在当前线程，那么将回调函数放入待完成事件集合中，在loop()函数中的doPendingFunctors()完成处理。在上文中可以看到loop函数的每次while()循环在最后都会运行doPendingFunctors()。   
pendingFunctors保存了待完成的事件，doPendingFunctors()函数很简单，就是顺序调用pendingFunctors各个回调函数。