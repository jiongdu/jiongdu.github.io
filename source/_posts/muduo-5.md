---
title: muduo源码阅读之线程同步   
date: 2016-07-08 15:07:13
tags: muduo
---
前面的四篇“muduo源码阅读”文章都是从总体上（Reactor模式）出发把握muduo库的一些设计思想和架构，从这篇开始，将更深入细节、更贴近代码地阅读muduo网络库。      
首先，是多线程编程中重中之重的线程同步问题。     

<!--more-->

### 互斥锁

提到线程同步原语，相信绝大多数人首先想到的就是互斥锁(mutex)了，互斥锁保护了临界区，保证任何时刻最多只有一个线程在由mutex保护的临界区内活动。所以，一般使用mutex来保护共享数据。muduo中使用了C++ RAII的手法封装了mutex的创建、销毁、加锁和解锁四个操作，并最终封装成MutexLockGuard类，这样，一切都交给栈上的MutexLockGuard类的对象的构造和析构函数负责，对象的生命期正好等于临界区。        

下面，首先来看MutexLock类的主要代码。

	class MutexLock : boost noncopyable
	{
		public:
			MutexLock() : holder_(0)
			{
				MCHECK(pthread_mutex_init(&mutex_,NULL)); //初始化mutex
			}
			~MutexLock()
			{
				assert(holder_=0);
				MCHECK(pthread_mutex_destroy(&mutex_));	//销毁mutex
			}
			void lock()
			{
				MCHECK(pthread_mutex_lock(&mutex_));	//加锁
				assignHolder();	
			}
			void unlock()
			{
				unassignHolder();
				MCHECK(pthread_mutex_unlock(&mutex_));	//解锁
			}
		private:
			friend class Condition;
			void assignHolder()
			{
				holder_=CurrentThread::tid();
			}
			void unassignHolder()
			{
				holder_=0;
			}
			pthread_mutex_t mutex_;
			pid_t holder_;
	};

   从代码中，很容易看出MutexLock对mutex的操作的封装。但是，为了防止忘记加锁、解锁操作，muduo做了更进一步的封装，于是有了MutexLockGuard类。
	
	class MutexLockGuard : boost::noncopyable
	{
		public:
			explicit MutexLockGuard(MutexLock& mutex) : mutex_(mutex)
			{
				mutex_.lock();		//在构造函数内进行加锁操作
			}	
			~MutexLockGuard()
			{
				mutex_.unlock();	//在析构函数内进行解锁操作
			}
		private:
			MutexLock& mutex_;
	};

这样一来，用户只需创建一个栈上的MutexLockGuard对象，便可以完成初始化、加锁操作，MutexLockGuard对象的生命期等于临界区，栈上对象析构的时候，自动完成解锁、销毁等操作。

### 条件变量 

互斥锁是加锁原语，用来排他性地访问共享数据，它不是等待原语。如果需要等待某个条件成立，就应该使用条件变量。条件变量是一个或多个线程等待某个条件为真，即等待别的“线程”唤醒它。    
所以，条件变量涉及等待端和通知端，所以双方都要按照一定的方式保证正常使用。    
对于wait端：       
（1）必须与mutex一起使用，条件（布尔表达式）的读写受mutex的保护       
（2）在mutex已上锁的时候才能调用wait()     
（3）把判断布尔表达式和wait()放入while()循环中     
对于signal/broadcast端：     
（1）在signal之前一般要修改布尔表达式           
（2）修改布尔表达式要加锁保护       
（3）signal（通知）通常用于资源可用，broadcast（广播）通常用于表明状态变化       
写成代码表示：     
 	
	muduo::MutexLock mutex;
	muduo::Condition cond(mutex);
	std::deque<int> queue;     
	
	int dequeue()
	{
		MutexLockGuard lock(mutex);
		while(queue.empty())
		{
			cond.wait();   		//等待条件变量
		}
		assert(!queue.empty());
		int top = queue.front();
		queue.pop_front();
		return top;
	}
	void enqueue(int x)
	{
		MutexLockGuard lock(mutex);
		queue.push_back(x);
		cond.notify();			//通知资源可用       
	}
以上，可以看成生产者--消费者模型，这里有一个经典的问题，即消费者中使用while()循环等待，可否使用if呢？答案是否定的，因为在多线程中，由于莫名其妙的原因，即使没有线程调用signal或broadcast，原来wait的线程可能也会返回，即被唤醒了。所以，通过while()循环判断来避免虚假唤醒造成的错误，而不能使用if。        
muduo中使用Condition类提供了对条件变量的封装。     

	class Condition : boost::noncopyable
	{
		public:
			explicit Condition(MutexLock& mutex) : mutex_(mutex)
			{
				MCHECK(pthread_cond_init(&pcond_,NULL));
			}
			~Condition()
			{	
				MCHECK(pthread_cond_destroy(&pcond_));
			}
			void wait()
			{
				MutexLock::UnassginGuard ug(mutex_);
				MCHECK(pthread_cond_wait(&pcond_,mutex_.getPthreadMutex()));
			}
			void notify()
			{
				MCHECK(pthread_cond_signal(&pcond_));
			}
			void notifyAll()
			{
				MCHECK(pthread_cond_broadcast(&pcond_));
			}
		private:
			MutexLock& mutex_;	
			pthread_cond_t pcond_;
	};    


### CountDownLatch
条件变量是非常底层的同步原语，很少直接使用，一般都是用它来实现高层的同步措施。muduo中封装的CountDownLatch（倒计时）就是一种常用且易用的同步手段。主要有两方面的用途：     
（1）主线程发起多个子线程，等这些子线程各自都完成一定的任务之后，主线程才继续执行。通常用于主线程等待多个子线程完成初始化。   
（2）主线程发起多个子线程，子线程都等待主线程，主线程完成其他一些任务之后通知所有子线程开始执行。通常用于多个子线程等待主线程发起“起跑”命令。     
下面是muduo中的倒计时CountDownLatch类。
	
	class CountDownLatch : boost::noncopyable
	{
		public: 
			explicit CountDownLatch(int count);		//倒数几次
			void wait();							//等待计数值变为0
			void countDown();						//计数减1
			void getCount() const;
		private:
			mutable MutexLock mutex_;
			Condition condition_;
			int count_;
	}

### 总结

互斥锁和条件变量构成了多线程编程的几乎全部必备的同步原语，正确并熟练地使用它们，即可完成任何多线程同步任务。在此基础上，如果特定的场景下实在还需要性能上的提升，再考虑更高级的同步手段。