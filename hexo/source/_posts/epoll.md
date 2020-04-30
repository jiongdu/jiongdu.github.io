---
title: epoll总结
date: 2016-07-29 19:03:21
tags: network/linux
---

在现如今的并发网络服务器程序开发中，我们都会见到IO多路复用（Linux下的epoll、Windows下的IOCP等）的影子，它们是各种服务器并发方案（[常见并发网络服务程序设计方案](http://blog.dujiong.net/2016/05/28/concurrent-server-conclusion/)）的基础，而我们知道，绝大部分服务器都是运行在Linux系统下，所以，本文就现如今Linux下用的最多的epoll的原理、用法进行一个简单总结。   

<!--more-->

### 什么是epoll？
epoll是在Linux 2.6内核中提出的，是之前select/poll的增强版本，相比如后两种IO多路复用版本，它能显著提高程序在大量并发连接中只有少量活跃的情况下的系统CPU利用率，因为它会复用文件描述符集合来传递结果而不用迫使开发者每次等待事件之前都必须重新准备要被侦听的文件描述符集合，另一点提升是获取事件的时候，epoll无须遍历整个被侦听的描述符，只要遍历那些被内核IO事件异步唤醒而加入就绪队列中的描述符集合就行了。此外，epoll除了提供select/poll那种IO事件的电平触发（LT）模式外，还提供边沿触发（ET），这样使得用户空间程序可以缓存IO状态，减少epoll_wait的调用，提高应用程序效率。        

### epoll的优点
上面笼统的说了一下epoll的一些优点，下面分条做一个总结。       
（1）支持一个大数目的socket描述符（FD）      
epoll没有文件描述符的限制，它所支持的FD上限是最大可以打开文件的数目，这个数字远远大于select所支持的1024/2048。可以通过`cat /proc/sys/fs/file-max`查看这个值，下面是我笔记本上的显示结果。     

	pt@Ubuntu:~$ cat /proc/sys/fs/file-max   
	6815744

（2）IO效率不随FD数目增加而线性下降     
传统select/poll的另一个致命弱点是当拥有一个很大的socket集合，由于网络的时延，使得任一时间只有部分的socket是“活跃”的，而select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。epoll对其进行了优化，它只会对“活跃”的socket进行操作：这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。所以，只有“活跃”的socket才会主动去调用callback函数，其他idle状态的socket则不会。在这点上，epoll实现了一个”伪”AIO”，因为这时候推动力在os内核。在一些 benchmark中，如果所有的socket基本上都是活跃的，比如一个高速LAN环境，epoll也不比select/poll效率低，但若过多使用的调用epoll_ctl，效率稍微有些下降。然而一旦使用idle connections模拟WAN环境，那么epoll的效率就远在select/poll之上了。     
（3）使用mmap加速内核与用户空间的消息传递     
无论是select,poll还是epoll都需要内核把FD消息通知给用户空间，如何避免不必要的内存拷贝就显得很重要。在这点上，epoll是通过内核于用户空间mmap同一块内存实现。

### epoll的两种工作模式     

#### LT模式
LT（Level Triggered，水平触发）模式，这是缺省的工作模式，同时支持block和non-block socket，在这种模式中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的，所以，这种模式编程出错误可能性要小一点。传统的select/poll都是这种模型的代表。 

#### ET模式   
ET(Edge Triggered，边沿触发)模式，这是高速工作模式，只支持non-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核就通过epoll告诉你，然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作而导致那个文件描述符不再是就绪状态(比如你在发送、接收或是接受请求，或者发送接收的数据少于一定量时导致了一个EWOULDBLOCK 错误)。但是请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核就不会发送更多的通知(only once)。不过在TCP协议中，ET模式的加速效用仍需要更多的benchmark确认。所以，这种方式下，出错率比较高，需要增加一些检测程序。

#### 使用ET和LT模式
上面已经说过了二者分别的特点，ET会更高效，但是只支持non-block socket，LT更加易用，且不易出错。个人觉得，一般的场景下，使用LT模式足以解决问题，如果碰到一些特殊场景要使用ET模式，一定要增加出错监测程序。        

### 使用epoll
epoll用到的所有数据结构和函数都是在头文件sys/epoll.h中声明的，下面逐一介绍。    

#### 相关数据结构        
	
	typedef union epoll_data{
		void* ptr;
		int fd;
		__uint32_t u32;
		__uint43_t u64;
	}epoll_data_t;
	
	struct epoll_event{
		__uint32_t events;    //Epoll events
		epoll_data_t data;    //User data variable	
	};
结构体epoll\_event被用于注册所感兴趣的事件和回传发生待处理的事件。epoll\_event 结构体的events字段是表示感兴趣的事件和被触发的事件，可能的取值为：    
EPOLLIN： 表示对应的文件描述符可以读；      
EPOLLOUT： 表示对应的文件描述符可以写；       
EPOLLPRI： 表示对应的文件描述符有紧急的数据可读；     
EPOLLERR： 表示对应的文件描述符发生错误；     
EPOLLHUP： 表示对应的文件描述符被挂断；     
EPOLLET： 表示对应的文件描述符有事件发生；     
联合体epoll_data用来保存触发事件的某个文件描述符相关的数据。例如一个client连接到服务器，服务器通过调用accept函数可以得到于这个client对应的socket文件描述符，可以把这文件描述符赋给epoll\_data的fd字段，以便后面的读写操作在这个文件描述符上进行。

#### epoll函数
1.创建函数     
`int epoll_create(int size)`    
该函数创建一个epoll实例，通知内核需要监听size个fd。size指的并不是最大的后备存储设备，而是衡量内核内部结构大小的一个提示。当创建成功后，会占用一个fd，所以记得在使用完之后调用close()，否则fd可能会被耗尽。不过，自Linux 2.6.8版本之后，size值没什么作用，只需大于0，因为内核可以动态的分配大小，不需要size这个提示了。     
2.注册函数     
`int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event)`     
第一个参数epfd，为上一步epoll\_create()返回的epoll fd。     
第二个参数op表示操作值，有三个类型：      
EPOLL\_CTL\_ADD ：注册目标到epfd中，同时关联内部event到fd上     
EPOLL\_CTL\_MOD ：修改已经注册到fd的监听事件      
EPOLL\_CTL\_DEL ：从epfd中删除/移除已注册的fd    
第三个参数fd表示需要监听的fd。
第四个参数event表示需要监听的事件。    
3.等待函数
`int epoll_wait(int epfd, structepoll_event * events, int maxevents, int timeout) `      
该函数用于轮询IO事件的发生。函数如果等待成功，则返回fd的数字；0表示等待fd超时，其他错误返回errno值。       

#### 使用综述
首先通过`int create\_epoll(int maxfds)`来创建一个epoll的句柄fd，其中maxfds为你的epoll所支持的最大句柄数（现只需设置大于0）。这个函数会返回一个新的epoll句柄，之后的所有操作都将通过这个句柄来进行操作。在用完之后，记得用close()来关闭这个创建出来的fd。                    
然后在你的网络主循环里面，调用epoll\_wait(int epfd, epoll\_event events, int max\_events,int timeout)来查询所有的网络接口，看哪一个可以读，哪一个可以写。基本的语法为： `nfds = epoll_wait(kdpfd, events, maxevents, -1);` 其中kdpfd为用epoll\_create创建之后的句柄，events是一个epoll\_event*的指针，当epoll\_wait函数操作成功之后，events里面将储存所有的读写事件。max\_events是当前需要监听的所有socket句柄数。最后一个timeout参数指示 epoll\_wait的超时条件，为0时表示马上返回；为-1时表示函数会一直等下去直到有事件返回；为任意正整数时表示等这么长的时间，如果一直没有事件，则会返回。一般情况下如果网络主循环是单线程的话，可以用-1来等待，这样可以保证一些效率，如果是和主循环在同一个线程的话，则可以用0来保证主循环的效率。epoll_wait返回之后，应该进入一个循环，以便遍历所有的事件。
