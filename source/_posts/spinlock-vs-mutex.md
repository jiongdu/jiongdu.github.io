---
title: Spin lock与Mutex     
date: 2016-09-18 21:34:32
tags: 进程/线程/并发
---
线程同步是多线程编程中最重要的问题之一，Linux下的POSIX threads库(Pthreads)是在Linux平台下进行多线程编程的一套API，其中线程同步最典型的应用就是用Pthreads提供的锁机制(lock)来对多个线程共享的临界区进行保护。     
<!--more--> 
Pthreads提供了多种线程同步锁机制：      
（1）Mutex（互斥量）：pthread\_mutex\_xxx     
（2）Spin lock(自旋锁)：pthread\_spin\_xxx    
（3）Condition Variable（条件变量）：pthread\_cond\_xxx   
（4）Read/Write lock（读写锁）：pthread\_rwlock\_xxx    
本文主要讲解Spin lock和其与Mutex（Mutex的用法很常见、也很简单，若还不熟悉，可参考[muduo中的封装](http://blog.dujiong.net/2016/07/08/muduo-5/)）之间的区别。     

### Spin lock
Spin lock又称自旋锁，线程通过busy-wait-loop的方式来获取锁，任何时刻都只有一个线程能够获得锁，其他线程忙等待直到获得锁。Spin lock在多处理器多线程环境的场景中有很广泛的使用。   
#### Spin lock和Mutex
Spin lock有如下特点：    
（1）Spin lock是一种死等的锁机制。当发生访问资源冲突的时候，可以有两种机制：一个是死等，一个是挂起当前进程，调度其他线程执行（Mutex）。线程会一直进行忙等待而不停的进行锁清秋，直到得到这个锁为止。       
（2）执行时间短。由于Spin lock死等这种特性，因此它使用在那些代码不是非常复杂的临界区（当然也不能太简单，否则使用原子操作或者其他适用简单场景的同步机制就可以了），如果临界区执行时间太长，那么不断在临界区门口进行死等的线程十分浪费CPU资源。      
（3）可以在中断上下文执行。由于不睡眠，因此Spin lock可以在中断上下文中使用。      
从上述总结的Spin lock的特点，已经可以看出其与Mutex的不同之处了，下面再以一个实例进行说明。   
例如在一个双核的机器上有两个线程(线程A和线程B)，它们分别运行在Core 0和Core 1上，假设线程A想要通过pthread\_mutex\_lock操作去得到一个临界区的锁，而此时这个锁正被线程B所持有，那么线程A就会被阻塞，Core 0会在此时进行上下文切换将线程A置于等待队列中，此时Core 0就可以运行其他的任务(例如另一个线程C)而不必进行忙等待。而Spin lock则不然，它属于busy-waiting类型的锁，如果线程A是使用pthread\_spin\_lock操作去请求锁，那么线程A就会一直在Core 0上进行忙等待并不停的进行锁请求，直到得到这个锁为止。

#### 使用Spin lock和Mutex        

下面通过实际的代码来进一步比较说明Spin lock和Mutex。

	#include <cstdio>
	#include <unistd.h>
	#include <sys/syscall.h>
	#include <errno.h>
	#include <sys/time.h>
	#include <pthread.h>
	#include <list>
	
	using namespace std;
		
	const int LOOPS = 50000000;

	list<int> ilist;

	#ifdef USE_SPINLOCK
		pthread_spinlock_t spinlock;
	#else
		pthread_mutex_t mutex;
	#endif
	
	pid_t gettid() 
	{
		return syscall( __NR_gettid );
	}

	void *consumer(void *ptr)
	{
    	int i;
 
    	printf("Consumer TID %lun", (unsigned long)gettid());
 
    	while (1)
    	{
	#ifdef USE_SPINLOCK
        	pthread_spin_lock(&spinlock);
	#else
        	pthread_mutex_lock(&mutex);
	#endif
 
        	if (ilist.empty())
        	{
	#ifdef USE_SPINLOCK
            	pthread_spin_unlock(&spinlock);
	#else
            	pthread_mutex_unlock(&mutex);
	#endif
            	break;
        	}
 
        	i = ilist.front();
        	ilist.pop_front();
 
	#ifdef USE_SPINLOCK
        	pthread_spin_unlock(&spinlock);
	#else
        	pthread_mutex_unlock(&mutex);
	#endif
  		}

    	return NULL;
	}
	
	int main()
	{
		int i;
   		pthread_t thr1, thr2;
    	struct timeval tv1, tv2;
 
	#ifdef USE_SPINLOCK
    	pthread_spin_init(&spinlock, 0);
	#else
    	pthread_mutex_init(&mutex, NULL);
	#endif
 
    	for (i = 0; i < LOOPS; i++)
        	ilist.push_back(i);

    	gettimeofday(&tv1, NULL);
 
    	pthread_create(&thr1, NULL, consumer, NULL);
    	pthread_create(&thr2, NULL, consumer, NULL);
 
    	pthread_join(thr1, NULL);
    	pthread_join(thr2, NULL);

    	gettimeofday(&tv2, NULL);
 
    	if (tv1.tv_usec > tv2.tv_usec)
    	{
        	tv2.tv_sec--;
        	tv2.tv_usec += 1000000;
    	}
    	printf("Result - %ld.%ld\n", tv2.tv_sec - tv1.tv_sec,
    	tv2.tv_usec - tv1.tv_usec);
 
	#ifdef USE_SPINLOCK
    	pthread_spin_destroy(&spinlock);
	#else
    	pthread_mutex_destroy(&mutex);
	#endif
 
    	return 0;
	}
该程序的逻辑是：主线程先初始化一个list结构，并根据LOOPS的值将对应数量的i插入该list，之后创建两个新线程，它们都执行consumer()这个任务。两个被创建的新线程同时对这个list进行pop()操作。主线程会计算从创建两个新线程到新线程结束之间所用的时间。    
代码执行平台参数：    
Ubuntu14.04 X86        
Inter i5-2430M @ 2.40GHz, Dual Core    
4.0GB Memory    
下面是代码执行结果：     
![](http://i.imgur.com/7dmwgzS.png)    
从结果中可以看出Spin lock表现出的性能更好，另外，sys时间是所花费的系统掉头时间，可以看出，Mutex将消耗更多的系统调用时间，这是因为Mutex会在锁冲突时调用System Wait造成的。        
但是，当临界区很大时，两个线程的锁进程会非常的激烈。这时，Spin lock的“死等”策略效率将会急剧下降。       

	#include <cstdio>
	#include <stdlib.h>
	#include <pthread.h>
	#include <unistd.h>
	#include <sys/syscall.h>

	using namespace std;

	const int THREAD_NUM = 2;
	
	pthread_t g_thread[THREAD_NUM];
	#ifdef USE_SPINLOCK
	pthread_spinlock_t g_spin;
	#else
	pthread_mutex_t g_mutex;
	#endif
	
	__uint64_t g_count;

	pid_t gettid()
	{
		return syscall(SYS_gettid);
	}

	void* run_amuck(void* arg)
	{
       int i, j;
 
       printf("Thread %lu started.n", (unsigned long)gettid());
 
       for (i = 0; i < 10000; i++) 
	   {
	#ifdef USE_SPINLOCK
           pthread_spin_lock(&g_spin);
	#else
           pthread_mutex_lock(&g_mutex);
	#endif
           for (j = 0; j < 100000; j++) 
		   {
               if (g_count++ == 123456789)
                  printf("Thread %lu wins!n", (unsigned long)gettid());
           }
	#ifdef USE_SPINLOCK
           pthread_spin_unlock(&g_spin);
	#else
           pthread_mutex_unlock(&g_mutex);
	#endif
       }
       printf("Thread %lu finished!n", (unsigned long)gettid());
 
       return NULL;
	}

	int main(int argc, char *argv[])
	{
       int i, threads = THREAD_NUM;
       printf("Creating %d threads...n", threads);
	#ifdef USE_SPINLOCK
       pthread_spin_init(&g_spin, 0);
	#else
       pthread_mutex_init(&g_mutex, NULL);
	#endif
       for (i = 0; i < threads; i++)
               pthread_create(&g_thread[i], NULL, run_amuck, (void *) i);
       for (i = 0; i < threads; i++)
               pthread_join(g_thread[i], NULL);
 
       return 0;
}

代码执行结果：     
![](http://i.imgur.com/mZQYF49.png)
果然，在我们选择使临界区变得很大时，锁竞争也变得很激烈。这样，Spin lock的性能急剧下降，Mutex的性能更好。同样地，可以看出，这种情况下，Spin lock消耗了很多的user time。原因是两个线程分别运行在两个核上，大部分时间只有一个线程能拿到锁，所以另一个线程就一直在它运行的core上进行忙等待，CPU占有率一直是100%；Mutex则不同，当对锁的请求失败后，上下文切换就会发生，这样就能空出一个核来运行别的计算任务。

### 总结

根据以上的分析和测试：    
（1）Mutex适合对锁操作非常频繁的场景，并且具有更好的适应性。尽管相比Spin lock会花费更多的开销（上下文切换），但是它能适合实际开发中复杂的应用场景，在保证一定性能的前提下提供更大的灵活度。     
（2）spin lock的lock/unlock性能更好(花费更少的cpu指令)，但是它只适应用于临界区运行时间很短的场景。而在实际软件开发中，除非程序员对自己的程序的锁操作行为非常的了解，否则使用spin lock不是一个好主意，实际中的多线程程序对锁的操作一般会很多。     
（3）最好的方式是先使用Mutex，然后如果对性能还有进一步的需求，可以尝试使用spin lock进行调优。毕竟，需要先保证程序的正确性，再考虑提升性能。    

参考：      
http://www.parallellabs.com/2010/01/31/pthreads-programming-spin-lock-vs-mutex-performance-analysis/  