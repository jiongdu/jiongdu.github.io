---
title: 从BlockingQueue再探线程同步
date: 2016-08-26 13:18:11
tags: [muduo,进程/线程/并发]
---
BlockingQueue（阻塞队列）在程序中被大量使用，特别是在多线程程序中，譬如需要提高吞吐量时采用将请求直接扔到阻塞队列中，然后返回，由其他线程从阻塞队列中获取请求再进一步处理。而经典的生产者-消费者模型，正是阻塞队列大放光彩之处。   
<!--more-->   
Java中java.util.concurrent包便提供了BlockingQueue接口和其几种实现方式。muduo中也实现了无大小限制的BlockingQueue和固定大小的BoundedBlockingQueue。下面以BlockingQueue为例进行说明。     

### BlockingQueue的实现
muduo中BlockingQueue的实现非常简单，队列采用C++标准库中的deque，无大小限制，唯一需要注意的是使用互斥锁和条件变量做好线程同步。   
BlockingQueue实现时运用了模板。

	template<typename T>
	class BlockingQueue : boost::noncopyable
	{
		public:
			BlockingQueue() : mutex_(), notEmpty_(mutex_), queue_() {}
		
			void put(const T& x)
			{
				MutexLockGuard lock(mutex_);
				queue_.push_back(x);
				notEmpty_.notift();
			}	
		
			T take()
			{
				MutexLockGuard lock(mutex_);
				while(queue_.empty())	
				{
					notEmpty_.wait();
				}
				assert(!queue_.empty());
				T front(queue_.front());
				queue_.pop_front();
				return front;
			}
	
			size_t size() const
			{
				MutexLockGuard lock(mutex_);
				return queue_.size();
			}
		private:
			mutable MutexLock mutex_;
			Condition notEmpty_;
			std::deque<T> queue_;
	};

BlockingQueue提供了取元素(take)、放入元素(put)和取得队列大小(size)三个public成员函数完成对阻塞队列的操作。

### 线程同步的一些细究
细究BlockingQueue的实现，可以总结并学习一些问题。

#### spurious wakeup
首先是大名鼎鼎的spurious wakeup（虚假唤醒）。即对应代码中`while(queue_.empty())`循环，可否改为`if(queue_.empty())`？       
答案是不能，否则会导致虚假唤醒，因为`notEmpty_.wait()`封装的`pthread_cond_wait()`不仅能被`pthread_cond_signal()/pthread_cond_broadcast()`唤醒，而且还可能会被其他的信号唤醒，后者便是虚假唤醒，这时条件并不满足。所以，不仅要在`pthread_cond_wait()`前检查条件是否成立，在`pthread_cond_wait()`之后也要检查。 

#### unlock和signal的顺序

muduo中采用RAII手法封装了互斥器，所以，在进行put操作时，是先notify其他线程，再释放锁。那么反过来呢，先释放锁，再notify其他线程？这二者有什么差别呢？       
先notify，再释放锁，在某些平台下存在性能问题，原因是，假设线程1阻塞在条件变量上wait，线程2将其唤醒（notify），系统执行上下文切换，但是线程2仍持有锁，导致线程1不能从`pthread_cond_wait()`(需要持有锁)返回，所以线程1又阻塞在互斥锁上，直到线程2释放锁，线程1才可以运行。      
福音是，对于此问题，Glibc中使用的线程库NPTL对此进行了优化。           
另一种情况是先释放锁，再notify，这样可以避免上述问题，但是，这样可能会唤醒其他阻塞在此mutex上的线程，而不是处于wait条件变量的线程，所以，这种情况下，在唤醒之后，最好还得再check下predicate（类似于spurious wakeup）。  
所以，Pthreads实现平台下，建议使用第一种方法，可以避免一些[obscure bugs](http://www.domaigne.com/blog/computing/condvars-signal-with-mutex-locked-or-not/),除非采用第二种有较明显的性能的提升。

#### Linux快速同步机制（Futex）

Futex（Fast Userspace mutexes，快速用户空间互斥体），是在Linux上实现锁定和构建高级抽象锁如信号量和POSIX互斥的基本工具，首先出现在Linux 2.5.7版。             
在传统的Unix系统中，System V IPC，如消息队列，信号量，socket等进程间同步机制都是对一个内核对象操作来完成的，这个内核对象对于要同步的进程都是可见的，其提供了共享的状态信息和原子操作。当进程间要同步的时候必须要通过系统调用在内核中完成。可是研究发现，很多同步状态是无竞争的，即某个进程进入互斥去，到从互斥区出来这段时间，常常是没有进程进也要进入这个互斥区或者请求同一个同步变量的。而这种情况下，进程也要陷入到内核去看有没有竞争者，退出的时候还要陷入内核去看有没有进程等待在同一变量上。这些不必要的系统调用（陷入内核）造成了大量的性能开销。       
而Futex就是在这样的背景下被提出的，以减少不必要的系统调用（内核陷入）。Funtex是一种用户态和内核态混合的同步机制。首先，同步的进程间通过mmap共享一段内存，Futex变量就位于这段共享的内存中且操作是原子的，当进程尝试进入互斥区或者退出互斥区的时候，先去查看共享内存中的futex变量，如果没有竞争发生，则只修改futex，而不用再执行系统调用了。当通过访问futex变量告诉进程有竞争发生，再去执行系统调用以完成相应的处理。futex通过这样的机制，大大提高了low-contention时的效率。     
    
##### Futex机制
所有的Futex同步操作都从用户空间开始，首先创建一个futex同步变量，也就是位于共享内存的一个整型计数器。   
当进程尝试持有锁或者要进入互斥区的时候，对futex执行"down"操作，即原子性的给futex同步变量减1。如果同步变量变为0，则没有竞争发生，进程照常执行。如果同步变量是个负数，则意味着有竞争发生，需要调用futex系统调用的futex\_wait操作休眠当前进程。      
当进程释放锁或者要离开互斥区的时候，对futex进行"up"操作，即原子性的给futex同步变量加1。如果同步变量由0变成1，则没有竞争发生，进程照常执行。如果加之前同步变量是负数，则意味着有竞争发生，需要调用futex系统调用的futex\_wake操作唤醒一个或者多个等待进程。     
这里的原子性加减通常是用CAS(Compare and Swap)完成的，与平台相关。CAS的基本形式是：CAS(addr,old,new),当addr中存放的值等于old时，用new对其替换。在x86平台上有专门的一条指令来完成它: cmpxchg。    

##### Linux下使用glibc开发

上面讲了很多关于Futex的机制，重点是弄清楚其中的原理和思想，我们在实际的多线程程序开发中却没有必要去实现自己的futex同步原语。       
因为NPTL库实现了POSIX标准定义的线程同步机制，他们都构造与futex之上，而glibc又使用NPTL作为自己的线程库。所以，在日常程序开发中，只需要正确地使用glibc所提供的同步方式，并在使用它们的过程中，意识到它们是利用futex机制和linux配合完成同步操作，重点是思想。
   



      



