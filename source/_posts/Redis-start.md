---
title: Redis之启动过程分析
date: 2016-11-10 09:49:28
tags: Redis
---

Redis服务器是典型的一对多服务器程序：一个服务器可以与多个客户端建立网络连接，每个客户端可以向服务器发送命令，而服务器接收并处理客户端发送的命令请求，并向客户端返回命令回复。通过使用I/O多路复用技术实现的文件事件处理器，Redis服务器使用单进程单线程的方式来处理命令请求。      
本文对Redis服务器的启动过程做简单分析。
<!--more-->

一个Redis服务器从启动到能够接受客户端的命令请求，需要经过一系列的初始化和设置过程，比如初始化服务器状态，接受用户指定的服务器配置，创建相应的数据结构和网络连接等。
### 初始化服务器状态结构
初始化服务器的第一步是创建一个struct redisServer类型的实例变量作为服务器的状态，并为结构中的各个属性设置默认值，在Redis中，服务器所有属性都保存在struct redisServer类型全局变量server中。该部分工作由redis.c/initServerConfig函数完成。     

	void initServerConfig()
	{
		...
		getRandomHexChars(server.runid, REDIS_RUN_ID_SIZE);	//设置服务器的运行ID
		server.configfile = NULL;
		server.port = REDIS_SERVERPORT;
		server.tcp_backlog = REDIS_TCP_BACKLOG;
		...
		server.lruclock = getLRUclock();	//
		server.commands = dictCreate(&commandTableDictType, NULL);
		server.orig_command = dictCreate(&commandTableDictType, NULL);
		populateCommandTable();
		...
	}   
其中，`server.lruclock = getLRUclock();`用于保存服务器的LRU时钟，表示服务器最近一次使用时钟的时间。精度为秒。
	
	//mstime()返回Unix时间毫秒数
	//#define REDIS_LRU_BITS 24
	//#define REDIS_LRU_CLOCK_MAX ((1<<REDIS_LRU_BITS)-1)    //Max value of obj->lru
	//#define REDIS_LRU_CLOCK_RESOLUTION 1000 		//LRU clock resolution in ms
	unsigned int getLRUClock(void){
		return (mstime()/REDIS_LRU_CLOCK_RESOLLUTION) & REDIS_LRU_CLOCK_MAX;
	}

此外，每个Redis对象都会有一个LRU属性，保存了对象最后一次被命令访问的时间。    

### 载入配置选项
在启动服务器时，用户可以通过给定配置参数（终端命令输入）或者指定配置文件（redis.conf）来修改服务器的默认配置。

	if(argc >= 2){
		int j=1;
		sds options = sdsempty();
		char* configfile = NULL;
		if(argv[j][0]!='-' || argv[j][1] != '-')	   //配置文件
			configfile = argv[j++];
		while(j != argc){	//除了配置文件还有其余选项，由options保存,比如--port 6380会被分析为"port 6380\n"
			if(argv[j][0]=='-' && argv[j][1]=='-'){
				if(sdslen(options)) options = sdscat(options, "\n");
				options = sdscat(options, argv[j]+2);
				options = sdscat(options, " ");
			}else{
				options = sdscatrepr(options,argv[j],strlen(argv[j]));
				options = sdscat(options, " ");
			}
			j++;
		}	
		if(configfile) server.configfile = getAbsolutePath(configfile);
		resetServerSaveParams();
		loadServerConfig(configfile, options);	//载入配置文件，还有options
		sdsfree(options);
	}

配置文件只能再第二个参数（第一个为程序名称），然后跟着配置文件后面可以是各个配置选项，都存储在option字符串中。最后调用函数`loadServerConfig(configfile, options);`将文件读进内存，并给server赋值。

	void loadServerConfig(char* filename, char* options)
	{
		sds config = sdsempty();
		char buf[REDIS_CONFIGFILE_MAX+1];
		
		if(filename){
			FILE* fp;
			
			if(filename[0]=='-' && finename[1]=='\0'){
				fp = stdin;
			}else{
				if((fp=fopen(filename,"r"))==NULL){
					redisLog(REDIS_WARNING, "Fatal error, can't open config file '%s'", filename);
					exit(1);
				}
			}
			while(fgets(buf, REDIS_CONFIGFILE_MAX+1, fp)!=NULL)
				config = sdscat(config, buf);
			if(fp != stdin) fclose(fp);
		}
		if(options) {
			config = sdscat(config, "\n");
			config = sdscat(config, options);
		}
		loadServerConfigFromString(config);
		sdsfree(config);
	}
将配置文件和options的配置存入config中，然后调用loadServerConfigFromString(config)根据配置选项名称给server结构体赋值。

### 初始化服务器数据结构
在之前执行initServerConfig函数初始化server状态时，程序只创建了命令表一个数据结构，不过除了命令表之外，服务器状态还包含其他数据结构。比如：   
(1) server.clients链表。该链表记录了所有与服务器相连的客户端的状态结构，链表的每一个节点都包含一个redisClient结构实例。    
(2) server.db数组。该数组包含了服务器中的所有数组。     
(3) 用于保存频道订阅信息的server.pubsub\_channels字典，以及用于保存模式订阅信息的server.pubsub\_patterns链表。     
(4)...   
当初始化服务器进行到这一步，服务器将调用initServer函数，为以上提到的数据结构分配内存，并在有需要时，为这些数据结构设置或关联初始化值。      
因此，前面的initServerConfig函数中主要负责初始化一般属性，而initServer函数主要负责初始化数据结构。

	void initServer()
	{
		int j;
		
		server.current_client = NULL;
    	server.clients = listCreate();
    	server.clients_to_close = listCreate();
		server.slaves = listCreate();
    	server.monitors = listCreate();

		createSharedObjects();	//创建共享对象
		server.el = aeCreateEventLoop(server.maxclients+REDIS_EVENTLOOP_FDSET_INCR);	//创建EventLoop实例
		server.db = zmalloc(sizeof(redisDb)*server.dbnum);

		if(server.port!=0 && 
				listenToPort(server.port, server.ipfd, &server.ipfd_count)==REDIS_ERR)
			exit(1);
		...
		if(acCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR){
			redisPanic("can't create the serverCron time event.");
			exit(1);
		}
		
		for(j=0; j<server.ipfd_count; j++){
			//为TCP连接关联连接处理器
			if(aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE, acceptTcpHandler, NULL)==AE_ERR){
				redisPanic("...");
			}	
		}	
		//为本地套接字关联处理器
		if(server.sofd>0 && aeCreateFileEvent(server.el, server.sofd, AE_READABLE, acceptUnixHandler, NULL)==AE_ERR){
			redisPanic("...");
		}
		//AOF文件
		//初始化脚本系统
		//初始化慢查询功能
		//...
	}    
其中，listenToPort函数先判断用户是否提供监听地址，如果没有，则监听INADDR\_ANY(0.0.0.0)地址，即所有地址，包括IPV4和IPV6，并设置为非阻塞。如果有设置监听地址，可以是一个地址队列，则只监听用户自己设置的ip地址队列。最后server.ipfd为监听套接字数组，server.ipfd_count为套接字数组个数。    
然后创建了时间和文件描述符事件，主要是设置处理事件的回调函数。   

### serverCron函数
Redis服务器中的serverCron函数默认每隔100（1000/server.hz）毫秒执行一次（第一次是1ms执行，后面是100ms），该函数负责管理服务器的资源，并保持服务器自身的良好运转。    
(1)更新服务器时间缓存  
Redis服务器中有不少功能需要获取系统的当前时间，而每次获取系统的当前时间都需要执行一次时间调用，为了减少系统调用的执行次数，服务器状态中的unixtime属性和mstime属性被当作当前时间的缓存，serverCron函数更新该域，这样就可以从这里获取时间。

	//time_t unixtime;	//秒级精度的系统当前UNIX时间戳
	//long long mstime;	//毫秒级精度的系统当前UNIX时间戳
	void updateCachedTime(void){
		server.unixtime = time(NULL);
		server.mstime = mstime();
	}
  服务器只会在打印日志、更新服务器的LRU时钟、决定是否执行初始化任务、计算服务器上线时间这类对时间精度要求不高的功能上“使用unixtime和mstime属性”。而对于为键设置过期时间、添加慢查询日志这种需要高精度时间的功能来说，服务器还是会再次执行系统调用，从而获得最准确的系统当前时间。     
(2)更新LRU时钟；     
(3)更新内存使用峰值   
服务器状态中的stat\_peak\_memory属性记录了服务器的内存峰值大小。每次serverCron函数执行时，程序都会查看服务器当前使用的内存数量，并与stat\_peak\_memory保存的数值进行比较，如果当前使用的内存数量比stat\_peak\_memory属性记录的值要打，那么程序就将当前使用的内存数量记录到stat\_peak\_memory。     
(4)处理SIGTERM信号     
在启动服务器时，Redis会为服务器进程的SIGTERM信号关联处理器sigtermHandler函数，该信号处理器接到SIGTERM信号时，打开服务器状态的shutdown\_asap标识。每次serverCron函数运行时，程序都会对服务器状态的shutdown\_asap属性进行检查，以决定是否关闭服务器。    
(5)管理客户端资源     
serverCron函数每次执行都会调用clientsCron函数，clientsCron函数会对一定数量的客户端进行一下两个检查：    
a. 如果客户端与服务器之间连接已经超时（很长时间没有互动），那么程序释放这个客户端，关闭连接。    
b. 如果客户端在上一次执行命令请求之后，输入缓冲区的大小超过了一定的长度，那么程序会释放客户端当前的输入缓冲区，并重新创建一个默认大小的输入缓冲区，从而防止客户端的输入缓冲区耗费了过多的内存。    
(6)管理数据库资源
serverCron函数每次执行都会调用databasesCron函数，该函数会对服务器中的一部分数据库进行检查，删除其中的过期键，并在有需要时，对字典进行收缩操作。    
(7)调度aof或rdb读写子进程，复制同步，集群同步，sentinel定时器等；   
总之，定时事件做的事情很多，可以说Redis充分利用了定时器，这样就少了很多线程。
