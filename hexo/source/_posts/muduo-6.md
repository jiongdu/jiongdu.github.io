---
title: muduo源码阅读之Thread和ThreadPool
date: 2016-07-17 19:14:12
tags: muduo
---

在muduo的one loop per thread + thread pool模型中，线程和线程池应该是其中最基础也是最重要的两个组件了。所以，本文深入代码，学习Thread和ThreadPool两个类的结构和实现。

<!--more-->

### Thread类

#### __thread关键字
学习Thread class之前，先了解一个关键字的用法：\_\_thread。    
\_\_thread是GCC内置的线程局部存储设施。它的实现非常高效，比Pthread库中的pthread\_key\_t（muduo中ThreadLocal）快很多。\_\_thread变量是表示每个线程有一份独立实体，各个线程的变量值互不干扰。\_\_thread只能修饰POD类型，不能修饰class类型，因为无法自动调用构造函数和析构函数。    
Thread类的封装用到了命名空间CurrentThread，这个空间中定义了和线程相关的一些独立属性。   

	namespace CurrentThread
	{
		__thread int t_cachedTid = 0;
		__thread char t_tidString[32];
		__thread int t_tidStringLength = 6;
		__thread const char* t_threadName = "unknown";
	}
其中，t\_cachedTid表示线程的真实id，Pthread库中提供了pthread\_self()获取当前线程的标识，类型为pthread\_t。但是，pthread\_t不一定是数值类型，也可能是一个结构体，这带来了一些问题。    
（1）无法打印输出pthread\_t，因为不知道其确切类型。     
（2）无法比较pthread\_t大小或计算hash值。     
（3）pthread\_t值只在进程内有意义，与操作系统的任务调度之间无法建立有效关系，Pthread库只能保证在同一进程之内，同一时刻的各个线程的id不同。    
所以，muduo采用gettid()系统调用的返回值作为线程id，muduo中将操作封装为gettid()函数。但是，我们知道，调用系统调用开销比较大，所以，muduo中采用\_\_thread变量t\_cachedTid来存储，在线程第一次使用tid时通过系统调用获得，存储在t\_cachedTid中，以后使用时不再需要系统调用了。     
t\_tidString[32]：用string类型表示tid，以便输出日志。    
t\_tidStringLength：string类型tid的长度。     
t_threadName：线程的名字。     

#### 相关的数据结构

在Thread的实现中，还用到了两个数据结构，一个位ThreadData，用来辅助调用线程执行的函数。另一个为ThreadNameInitializer，为线程的创建做环境准备。其中用到了`pthread_atfork(NULL,NULL,&afterfork)`。该函数的原型为
	`int pthread_atfork(void (*prepare)(void), void (*parent)(void), void (*child)(void));`    
这是一个跟进程创建有关的函数，为fork的调用做准备和调用后子进程父进程的初始化。prepare函数在调用fork前执行，parent在调用fork后的父进程中执行，child在调用fork后的子进程中执行。         
当然，在实际应用中，多线程不要调用fork()，否则会出现一些问题。因为fork智能克隆当前线程的thread of control，却不克隆其他线程。fork()之后，除了当前线程之外，其他线程都消失了。这样，会出现很多问题。比如，如果复制了一个lock的mutex，却没有复制unlock的线程，那么在给mutex加锁时就会出现死锁。


#### Thread类分析
首先来看Thread类的数据成员和构造函数。
	
	class Thread : boost::noncopyable
	{
		typedef boost::function<void ()> ThreadFunc;
		...
		private:
			bool started_;
			bool joined_;
			pthread_t pthreadId_;
			boost::shared_ptr<pid_t> tid_;
			ThreadFunc func_;
			string name_;

			static AtomicInt32 numCreated_;
	};
	Thread::Thread(ThreadFunc&& func, const string& n)
		: started_(false), 
		  joined_(false),
    	  pthreadId_(0),
    	  tid_(new pid_t(0)),
    	  func_(func),
    	  name_(n)
	{
  		setDefaultName();
	}
其中有两处需要说明一下，首先是`shared_ptr<pid> tid_`，可能有人会有疑问：为什么这里要用shared\_ptr包装pid？        
原因是tid\_所属的对象Thread在主线程（A）中创建，而tid\_需要在新创建的线程B中进行赋值操作，如果tid使用裸指针的方式传递给线程（B），那么线程A中Thread对象析构（下文）销毁后，线程B持有的就是一个野指针，所以，在Thread对象中将以shared\_ptr包装。      
然后是numCreated\_，是一个静态变量，类型为AtomicInt32，原子类型，用来表示第几次创建线程实例，在记录日志时可用记录为：“线程名+numCreated\_”。

接下来看Thread的一些接口函数。       

	void Thread::start()
	{
		started = true;
		detail::ThreadData* data = new detail::ThreadData(func_, name_, tid_);
		if(pthread_create(&pthreadId_, NULL, &detail::startThread, data));
		{
			started_ = false;
			delete data;
			LOG_SYSFATAL << "Failed in pthread_create";
		}
	}

Thread::start()将调用pthread\_create()创建新线程，detail::startThread()是新线程的入口函数，data是新线程执行的辅助结构体。detail::startThread()调用data->runInThread()执行线程逻辑(func_)。
	
	int Thread::join()
	{
		assert(started_);
		assert(!joined_);
		joined_ = true;
		return pthread_join(pthreadId_, NULL);
	}

	Thread::~Thread()
	{
		if(started_ && !joined_)
		{
			pthread_detach(pthreadId_);
		}
	}

Thread析构的时候没有销毁持有的Pthreads句柄(pthread\_t)，也就是说Thread的析构不会等待线程结束。如果Thread对象的生命期长于线程，然后通过Thread::join()来等待线程结束并释放线程资源。如果Thread对象的生命期短于线程，那么析构时会自动detach线程，避免了资源泄露。

### ThreadPool类
ThreadPool（线程池）本质上是一个生产者-消费者的模型，在实际中主要完成计算任务。在muduo线程池中有一个存放工作线程的容器ptr_vector，相当于消费者；有一个存放任务的队列deque。      
任务队列是有界的，类似于BoundedBlockingQueue，实现时需要两个条件变量。
以下是ThreadPool的数据成员：

	class ThreadPool : boost::noncopyable
	{
		typedef boost::function<void ()> Task;
		private:
			MutexLock mutex_;
			Condition notEmpty_;
			Condition notFull_;
			string name_;
			Task threadInitCallback_;
			boost::ptr_vector<muduo::Thread> threads_;
			std::deque<Task> queue_;
			size_t maxQueueSize_;
			bool running_;
	}
其中threadInitCallback\_可由setThreadInitCallback(const Task& cb)设置，设置回调函数，每次在执行任务前先调用。在线程池开始运行之前，需要先设置任务队列的大小（调用setMaxQueueSize()），因为运行线程池时，线程会从任务队列取任务。     
接下来是ThreadPool的一些接口函数。

	void ThreadPool::start(int numThreads)
	{
		assert(threads_.empty());
		running_ = true;
		threads_.reserve(numThreads);
		for (int i = 0; i < numThreads; ++i)
  		{
    		char id[32];
    		snprintf(id, sizeof id, "%d", i+1);
    		threads_.push_back(new muduo::Thread(
          		boost::bind(&ThreadPool::runInThread, this), name_+id));
    		threads_[i].start();
  		}
		if(numThreads == 0 && threadInitCallback_)
		{
			threadInitCallback_();
		}
	}
`void ThreadPool::start(int numThreads)`开启线程池，按照线程数量numThreads_创建工作线程，线程函数为ThreadPool::runInThread（）。    

	void ThreadPool::runInThread()
	{
		try
		{
			if(threadInitCallback_)
			{
				threadInitCallback_();
			}
			while(running_)
			{
				Task task(take());
				if(task)
				{
					task();
				}
			}
		}
		catch(const Exception& ex)
		{
			...
		}
		
	}
如果设置了threadInitCallback\_，则进行执行任务前的一些初始化操作。然后从任务队列中取任务执行，有可能阻塞，当任务队列为空时。

	
	ThreadPool::Task ThreadPool::take()
	{
		MutexLockGuard lock(mutex_);
		while(queue_.empty() && running_)
		{
			notEmpty_.wait();
		}
		Task task;
		if(!queue_.empty())
		{
			task = queue_.front();
			queue_.pop_front();
			if(maxQueueSize_ > 0)
			{
				notFull_.notify();
			}
		}
		return task;
	}
多线程从消息队列中取任务的时候，需要加锁保护。等到队列非空信号，就取任务。取出之后，便告知任务队列已经非满，可以继续添加任务。
	
	void ThreadPool::run(const Task& task)
	{
		if(threads_.empty())
		{
			task();
		}
		else
		{
			MutexLockGuard lock(mutex_);
			while(isFull())
			{
				notFull_.wait();
			}
			assert(!isFull());
			queue_.push_back(task);
			notEmpty_.notfy();
		}
	}
如果ThreadPool没有子线程（set和start操作在run之前），就在主线程中执行该task，否则，将任务加入到队列，并通知线程从中取task，如果队列已满，便等待。

	ThreadPool::~ThreadPool()
	{
		if(running_)
		{
			stop();
		}
	}
	void ThreadPool::stop()
	{
		{
			MutexLockGuard lock(mutex_);
			running_ = false;
			notEmpty_.notifyAll();
		}
		for_each(threads_.begin(),threads_.end(),boost::bind(&muduo::Thread::join, _1));
	}

最后是ThreadPool的析构函数，在其中调用stop()，唤醒所有等待的线程，然后对线程池中的每一个线程执行join()。     

### 总结
以上就是muduo中Thread和ThreadPool类的学习，有很多源码，有点啰嗦。但是，在muduo的one loop per thread + thread pool模型中，Thread和ThreadPool是很重要的组件，所以需要深入地掌握。
	
