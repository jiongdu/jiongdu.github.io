---
title: goroutine和golang调度器
date: 2018-04-28 21:07:00
tags: [golang]
---

关于golang，自己听到的第一句话就是“原生支持并发”，而goroutine就是其具体实现。

<!--more-->

### golang调度器
本节围绕“**存在内核线程调度器的情况下，golang为什么要设计一个自己的调度器？**”这一问题进行讨论。
我们知道，操作系统的内核调度器会将系统中的多个线程按照一定的算法调度到物理CPU上运行。最典型的例子如C/C++语言的并发实际上就是基于内核调度的，应用程序负责创建线程（比如linux下通过pthread库调用），内核负责调度。但是，这种传统方式存在一些不足：        
1、**代价较高**：虽然线程的代价已经比进程小了很多，但仍占用不小的资源，并且线程切换也有不小的消耗，因此我们依然不能创建大量的线程。     
2、常见的线程模型---N:1和1:1模型**不能较好的兼顾**1）充分利用多核CPU的优势；2）快速的上下文切换。    
3、**线程之间的通信不易掌握**。虽然多线程之间的通信有多种方法，但使用起来都比较复杂、易错。比如常见的共享内存，各种加锁操作会使得程序开发和问题定位相当困难，甚至会出现死锁和挂掉等问题。    
4、**大量使用异步**。比如很多网络服务程序，如上文所述，由于系统中不能创建大量线程，所以通常需要在线程中使用IO多路复用（如epoll），然后通过注册回调函数的机制实现各事件的异步调用。但是，异步调用的方式和我们通常的思维习惯不同，当系统中存在大量的回调函数时，会给程序的开发和维护带来较大困难。  

**基于以上问题，go采用了用户态轻量级线程（“goroutine”）的方式来解决**。其主要特点可以归纳为：    
1、**用户态**：goroutine在用户空间实现，其对于操作系统来说只是一个应用程序，操作系统甚至不知道有goroutine的存在，goroutine的调度需要自己完成。     
2、**轻量级**：goroutine占用的资源很小，因此一个go程序可以创建成千上万个并发的goroutine。同时，goroutine之间的调度切换不需要陷入内核中，代价较低。将多个goroutine按照一定的算法调度到CPU上执行需要goroutine调度器。
3、倡导“**通过通信来共享内存，而不是通过共享内存来通信**”。go通过语法关键词提供了channel机制，用于goroutine之间的通信，实现了CSP并发模型。该并发范式逻辑简单清楚，系统有高正确性。

### golang调度模型
golang试图通过M:N模型来兼顾N:1和1:1模型的优点，具体来说，每个用户线程可以对应多个内核线程，同时一个内核线程也可以对应多个用户线程。M:N模型的缺点是调度复杂，需要设计良好的调度器。  

如下图所示，goroutine调度器中使用了三种实体：（1）M：系统线程；（2）P：逻辑处理器（核）；（3）G：goroutine。图中，每个线程运行了一个goroutine，所以必须得维持一个上下文P。P的数量由启动时环境变量GOMAXPROCS决定，即在程序执行的过程中都只有GOMAXPROCS个gouroutine在同时运行。而灰色的goroutine在等待被调度，它们被维护在一个队列（runqueue）里。当一个go语句执行（即创建一个goroutine）时，就有一个新的goroutine被添加到runqueue队尾；当运行当前goroutine到调度点时，runqueue队头弹出一个goroutine进入运行。  

![](https://i.imgur.com/TewfErj.png)

因此，当一个上下文（P）运行完要被调度的所有goroutine时，系统的稳定状态将被改变。此时如果各上下文的运行队列里的goroutine数目不均衡，将会导致一个上下文在执行完它的运行队列后就结束了，而系统中仍然有许多goroutine要执行。所以，调度系统需要维护一个全局队列，每个上下文能够从全局队列中获取goroutine，但如果全局队列中也没有goroutine了，上下文还能从其他上下文获取goroutine。该过程被称为“stealing”，goroutine通过“stealing”，确保每个上下文总是处于运行，同时保障所有线程都处于最大负载。
goroutine调度器模型践行了计算机界的一句名言：“**计算机科学领域的绝大多数问题都可以采用增加一个间接的中间层解决**”。每个G（goroutine）要想真正运行起来，首先需要被分配一个P（进入到P的local runqueue中），在G看来，P就是它所运行的“CPU”。但在go调度器看来，真正的“CPU”却是M，只有将P和M绑定才能让P的runqueue中的G真正运行起来。因此，goroutine调度器通过向G-M模型中增加一个中间层P，实现了高效、可扩展的调度。

### golang调度器跟踪
下面通过一个例子来查看go调度器的一些跟踪信息。 

![](https://i.imgur.com/rKoVNN1.png)

在main函数中，通过一个for循环创建了10个goroutine，然后在第16行等待所有的goroutine完成任务。每个goroutine执行的work函数中先sleep 1秒，然后循环计数1e9次。完成后调用waitGroup的Done方法返回。
然后在linux环境下运行该程序，并设置GODEBUG环境变量和schedtrace参数观察调度器的输出信息，其结果如下图所示。

![](https://i.imgur.com/03sGFDQ.png)

其中，xxxms代表自程序开始运行时的毫秒数，gomaxprocx代表配置的处理器数，即上述golang调度器模型中的P，threads代表运行时管理的线程数，idleprocs代表空闲的处理器数，idlethreads代表空闲的线程数，spinningthreads处于自旋状态的线程数（M正在寻找可运行的G）。runqueue=xxx代表在全局队列中的goroutine数。[xxx]代表本地run队列中的goroutine数。
而如果还想更深入的了解goroutine调度的细节，可以添加scheddetail参数，该参数下将打印输出处理器P、线程M和goroutine的细节，如下图所示。

![](https://i.imgur.com/MhoDjYV.png)

可以看出，与上图相比，添加scheddetail参数后P、M、G实体的状态和摘要信息更加清晰，这里就不再一一描述，有兴趣的同学可以参考：https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs

### 总结
golang入门不久，对goroutine的设计及调度，高并发CSP模型等问题有较强的兴趣，本文从初学者的角度，简单学习、总结了goroutine和对应调度器的一些设计思路，希望借此能更好的理解go语言，后续还将对channel以及对应的CSP模型进行较深入的学习。

### 附

本文参考：
https://morsmachine.dk/netpoller
https://software.intel.com/en-us/blogs/2014/05/10/debugging-performance-issues-in-go-programs   




























