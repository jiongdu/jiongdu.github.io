---
title: 协程
date: 2016-12-23 18:41:42
tags: 进程/线程/并发
---

协程，顾名思义，是“协作的例程”。跟具有操作系统概念的线程不一样，协程是在用户空间利用程序语言的语法语义就能实现逻辑上类似多任务的编程技巧。协程可以在运行期间的某个点上暂停执行，并在恢复运行时从暂停的点上继续执行。协程已经被证明是一种非常有用的程序组件，不仅被Python、lua、ruby等脚本语言广泛语言，还被新一代面多核的编程语言如golang等作为并发的基本单位。  
<!--more-->   
### 协程的特点
协程的调度完全由用户控制，一个线程可以有多个协程，每个协程都是循环按照指定的任务清单顺序完成不同的任务，当任务被阻塞的时候执行下一个任务，当恢复的时候再回来执行这个任务，任务之间的切换只需要保存每个任务的上下文内容，就像直接操作栈一样，这样就完全没有内核切换的开销，可以不加锁的访问全局变量，所以上下文的切换非常快。     
因此，与传统的抢占式线程相比，协程主要具有以下两个优点：    
（1）与线程不同，线程是自己主动让出CPU，并交付它期望的下一个协程运行，而不是在任何时候都有可能被系统调度打断。因此协程的使用更加清晰易懂，并且大多数情况下不需要锁机制。    
（2）与线程相比，协程的切换由程序控制，发生在用户空间而非内核空间，因此切换的代价非常小。    

### 进程与线程
下面简单回顾下进程与线程的概念。   
#### 进程
进程是具有一定独立功能的程序关于某个数据集合上的一次活动，进程是系统进行资源分配和调度的单位。      
进程之间不共享任何状态，进程的调度由操作系统完成，每个进程都有自己的独立的内存空间，而进程间的通信主要是通过信号传递的方式来实现的，实现的方式有多种，如信号量、管道等，但是任何一种方式的通信都需要通过内核，因此效率比较低。同时，由于进程拥有的是独立的内存空间，所以在进行上下文切换的时候需要先保存调用栈的信息，CPU各寄存器的信息，虚拟内存以及打开的句柄等信息，所以导致进程间切换开销很大。
#### 线程
线程是进程的一个实体，是CPU调度的基本单位，它是比进程更小的能独立运行的基本单位。线程自己基本上不拥有系统资源，只拥有一点再运行中比不可少的资源（如程序计数器、一些寄存器和栈），它与同一个进程的其他线程共享所拥有的全部资源。    
线程之间共享变量，解决了通信麻烦的问题，但同时，多个线程对变量的访问需要进行同步与互斥操作。线程的调度也主要由操作系统完成，一个进程可以拥有多个线程，每个线程会共享父进程向操作系统申请的资源，包括虚拟内存，文件等，因此创建线程所需的资源要比进程小很多，相应的可创建的线程数量也变得多很多。线程之间的通信除了可以使用进程之间通信的方式之外还可以通过共享内存的方式，此外，在线程调度方面，由于线程间共享进程的资源，所以上下文切换的时候需要保存的东西相对少些，上下文也因此高效一些。     
          
### 构建C协程
C/C++不直接支持协程语义，但目前已有不少开源的协程库，本文以其中最常用的使用glibc的ucontext组件的实现方式进行说明。
#### ucontext组件
ucontext组件是GNU C库提供的一组用于创建、保存、切换用户态执行"上下文"的API。主要包括以下两个结构体和四个函数：
		
	//mcontest_t类型与机器相关，并且不透明
	typedef struct ucontext {
		struct ucontext* uc_link;	//链接下一个执行的上下文
		sigset_t uc_sigmask;	//阻塞信号集合
		stack_t uc_stack;		//该上下文中使用的栈
		mcontext_t uc_mcontest;
		...
	} ucontext_t;

	//初始化一个ucontext_t类型的结构，即用户执行上下文。
	//函数指针func指明了该context的入口函数，argc指入口参数个数	
	void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...);
	//"原子"地完成旧状态的保存和切换到新状态的工作
	int swapcontext(ucontext_t *oucp, ucontext_t *ucp);
	//将当前执行上下文保存到ucp中，若后续调用setcontext或swapcontext恢复状态，
	//则程序会沿着getcontext调用点之后继续执行，看起来好像刚从getcontext函数返回一样。
	int getcontext(ucontext_t *ucp);
	//将当前程序执行线索切换到ucp所指向的上下文状态，
	//在执行正确的情况下，该函数直接切入到新的执行状态，不再回返回
	int setcontext(const ucontext_t *ucp);
来看一个简单的实例：

	#include <stdio.h>
	#include <ucontext.h>
	#include <unistd.h>

	int main()
	{
		ucontext_t context;
		
		getcontext(&context);	
		puts("hello world");
		sleep(1);
		setcontext(&context);
		return 0;
	}		 
运行结果如下：   
![](http://i.imgur.com/SvtkRI7.png)               

程序通过getcontext保存了一个上下文，然后输出"hello world"，睡一秒后执行到setcontext，恢复上下文到getcontext之后，重新执行代码，所以程序不断输出“hello world”。   
#### 使用ucontext实现线程切换

虽然我们称协程是一个用户态的轻量级线程，但实际上多个协程同属于一个线程。任意一个时刻，同一个线程不可能同时运行两个协程。    
接下来通过一个实例说明协程与主函数的切换，即实现协程的调度。 

	#include <ucontext.h>
	#include <stdio.h>

	void func1(void *arg)
	{
		puts("1");
		puts("11");
		puts("111");
		puts("1111");
	}

	void context_test()
	{
		char stack[1024*128];
		ucontext_t child, main;
		getcontext(&child);
		child.uc_stack.ss_sp = stack;
		child.uc_stack.ss_size = sizeof(stack);
		child.uc_link = &main;
		makecontext(&child, (void (*)(void))func1, 0);
		swapcontext(&main, &child);
		puts("main");
	}
	
	int main()
	{
		context_test();
		return 0;
	}
运行结果如下图所示。    
![](http://i.imgur.com/FbFrKKW.png)
在context\_test中，创建了一个用户线程（协程）child，其运行的函数为func1，指定后继上下文为main，当func1返回后激活后继上下文，继续执行main函数。

### 开源C/C++协程库
最后，罗列几个比较出名的开源C/C++协程库，后面争取再深入学习下。     
libco： [腾讯的开源协程库](http://code.tencent.com/libco.html)    
coroutine: [云风大牛的作品](https://github.com/cloudwu/coroutine/)    
Protothreads: [一个"蝇量级"C语言协程库](http://coolshell.cn/articles/10975.html)
       

