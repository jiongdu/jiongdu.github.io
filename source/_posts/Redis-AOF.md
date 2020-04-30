---
title: Redis之AOF持久化
date: 2016-11-30 20:17:07
tags: Redis
---
除了RDB持久化功能之外，Redis还提供了AOF（Append Only File）持久化功能。与RDB持久化通过保存数据库中的键值对来记录数据库状态不同，AOF持久化是通过保存Redis服务器所执行的写命令来记录数据库状态的。
<!--more-->
### AOF持久化的实现
AOF持久化功能的实现可以分为命令追加、文件写入、文件同步三个步骤。
#### 命令追加
当AOF持久化功能处于打开状态时，服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的aof\_buf缓冲区的末尾：

	struct redisServer {
		sds aof_buf;		//AOF缓冲区
		...
	};    

#### AOF文件的写入与同步

Redis的服务器进程是一个事件循环，这个循环中的文件事件负责接收客户端的命令请求，以及向客户端发送命令回复，而时间事件则负责执行像serverCron函数这样需要定时运行的函数。     
因为服务器在处理文件事件时可能会执行写命令，使得一些内容被追加到aof\_buf缓冲区，所以在服务器每次结束一个事件循环之前，它都会调用flushAppendOnlyFile函数，考虑是否需要将aof\_buf缓冲区中的内容写入和保存到AOF文件中。      
因为程序需要在回复客户端之前对AOF执行写操作，而客户端能执行写操作的唯一机会就是在事件loop中，因此，程序将所有AOF写累计到缓存中，并在重新事件之前，将缓存写入到文件中。下面是flushAppendOnlyFile函数的关键代码。 

	void flushAppendOnlyFile(int force) {
		ssize_t nwritten;
		int sync_int_progress = 0;
		if(sdslen(server.aof_buf) == 0) return;		//缓冲区没有内容

		//当fsync策略为每秒一次时，如果后台线程仍然有fsync在执行，那么可能会延迟执行冲洗（flush）操作，因为Linux上的write会被后台的fsync阻塞。
		//当这种情况下，说明需要尽快冲洗aof缓存，程序会尝试在serverCron函数中对缓存进行冲洗。
		//但是，当force为1时，不管后台是否在fsync，程序都直接进行写入
		if(server.aof_fsync == AOF_FSYNC_EVERYSEC)	//每秒同步
			sync_in_progress = bioPendingJobsOfType(REDIS_BIO_AOF_FSYNC) != 0;
		if(server.aof_fsync == AOF_FSYNC_EVERYSEC && !force){
			if(sync_in_progress){	//后台正在执行FSYNC
				if(server.aof_flush_postponed_start == 0){
					server.aof_flush_postponed_start = server.unixtime;	 //前面没有推迟过write操作，记录推迟写操作的时间
					return；
				}
				else if(server.unixtime-server.aof_flush_postponed_start < 2){	//推迟时间小于2秒
					return；
				}
				server.aof_delayed_fsync++;
				redisLog(REDIS_NOTICE, "...");
			}
		}
		//程序对AOF文件进行写入
		server.aof_flush_postponed_start = 0;
		//如果写入设备是物理的话，这个操作应该是原子的
		//若出现电源中断这样的不可抗现象，AOF文件也是可能出现问题的，这时就需要使用redis-check-aof程序来修复
		nwritten = write(server.aof_fd, server.aof_buf, sdslen(server.aof_buf));
		if(nwritten != (signed)sdslen(server.aof_buf)){
			//不成功处理
			...
		}else{
			if(server.aof_last_write_status == REDIS_ERR){
				redisLog(REDIS_WARNING, "...");
				server.aof_last_write_status = REDIS_OK;
			}
		}
		server.aof_current_size += nwritten;	//更新写入后的AOF文件大小
		if(sdslen(server.aof_buf)+sdsavail(server.aof_buf) < 4000){
			sdsclear(server.aof_buf);	//清除缓存内容，等待重用
		}else{
			//释放缓存
			sdsfree(server.aof_buf);
			server.aof_buf = sdsempty();
		}
		
		if(server.aof_fsync == AOF_FSYNC_ALWAYS) {		//策略为总是执行fsync
			aof_fsync(server.aof_fd);
			server.aof_last_fsync = server.unixtime;
		}else if(server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync){		//策略为每秒fsync, 并且距离上次fsync已经超过1秒
			if (!sync_in_progress) aof_background_fsync(server.aof_fd);
			server.aof_last_fsync = server.unixtime;
		}
	}   	

服务器配置appendfsync选项的值直接决定AOF持久化功能的效率和安全性。    
当appendfsync的值为always时，服务器在每个事件循环都要将aof\_buf缓冲区中的所有内容写入到AOF文件，并且同步AOF文件，所有always的效率是appendfsync三个选项值中最慢的一个，但从安全性来看，always也是最安全的，因为即使出现故障停机，AOF持久化也只会丢失一个事件循环中所产生的命令数据。       
当appendfsync的值为everysec时，服务器在每个事件循环都要将aof\_buf缓冲区中的所有内容写入到AOF文件，并且每隔一秒就要在子线程中对AOF文件进行一次同步。从效率上来讲，everysec模式足够快，并且即使出现故障停机，也只丢失一秒钟的命令数据。   
当appendfsync的值为no时，服务器在每个事件循环都要将aof\_buf缓冲区中的所有内容写入到AOF文件，至于何时对AOF文件进行同步，则由操作系统决定。所以该模式的单次同步时长通常是三种模式中时间最长的，并且在出现故障时，将丢失上次同步AOF文件之后的所有写命令数据。     

### AOF文件的载入与数据还原
	
因为AOF文件里面包含了重建数据库状态所需的所有写命令，所有服务器只要读入并重新执行一遍AOF文件里面保存的命令，就可以还原服务器关闭之前的数据库状态。Redis读取AOF文件并还原数据库状态的详细步骤如下：     
（1）创建一个不带网络连接的伪客户端：因为Redis的命令只能在客户端上下文中执行，而载入AOF文件时所使用的命令直接来源于AOF文件而不是网络连接，所以服务器使用了一个没有网络连接的伪客户端来执行AOF文件保存的写命令，伪客户端执行命令的效果和带网络连接的客户端执行命令的效果完全一样。        
（2）从AOF文件中分析并读出一条写命令。     
（3）使用伪客户端执行被读出的写命令。     
（4）重复执行步骤2和步骤3，直到AOF文件中的所有写命令都被处理完毕为止。      

#### AOF重写
因为AOF持久化是通过保存被执行的写命令来记录数据库状态的，所以随着服务器运行时间的流逝，AOF文件中的内容会越来越多，文件的体积也会越来越大，如果不加以控制的话，体积过大的AOF文件很可能对Redis服务器、甚至整个宿主计算机造成影响，并且AOF文件的体积越大，使用AOF文件来进行数据还原所需的时间就越多。      
为了解决AOF文件体积膨胀的问题，Redis提供了AOF文件重写功能。通过该功能，Redis服务器可以创建一个新的AOF文件来替代现有的AOF文件，新旧两个AOF文件所保存的数据库状态相同，但新AOF文件不会包含任何浪费空间的冗余命令，所以新AOF文件的体积通常会比旧AOF文件的体积要小得多。
#### 实现
AOF文件重写是通过读取服务器当前的数据库状态来实现的，并不需要对现有的AOF文件进行任何读取、分析或者写入操作。即首先从数据库状态中读取键现在的值，然后用一条命令去记录键值对，代替之前记录这个键值对的多条命令。           
下面是相关的源代码实现。       

	int rewriteAppendOnlyFile(char* filename)
	{
		dictIterator *di = NULL;
		dictEntry *de;
		rio aof;
		FILE *fp;
		char tmpfile[256];
		int j;
		long long now = mstime();
		
		snprintf(tmpfile, 256, "temp-rewriteaof-%d.aof", (int)getpid());
		fp = fopen(tmpfile, "w");
		
		rioInitWithFile(&aof, fp);
		//设置每写入REDIS_AOF_AUTOSYNC_BYTES字节，就执行一次FSYNC
		//防止缓存中积累太多命令内容，造成I/O阻塞时间过长
		if(server.aof_rewrite_incremental_fsync){
			rioSetAutoSync(&aof, REDIS_AOF_AUTOSYNC_BYTES);
		}
		//遍历所有数据库
		for(j=0; j<server.dbnum; j++){
			char selectcmd[] = "*2\r\n$6\r\nSELECT\r\n";
			redisDb *db = server.db+j;
			dict *d = db->dict;
        	if (dictSize(d) == 0) continue;
			di = dictGetSafeIterator(d);		//键空间迭代器
			if(!di) {
				fclose(fp);	
				return REDIS_ERR;
			}
			//写入SELECT命令
			if(rioWrite(&aof, selectcmd, sizeof(selectcmd)-1) ==0 ) goto werr;
			if (rioWriteBulkLongLong(&aof,j) == 0) goto werr;
			
			//遍历数据库所有键
			while((de = dictNext(di)) != NULL) {
				robj key, *o;
				long long expiretime;	
				sds keystr = dictGetKey(de);
				
				o = dictGetVal(de);		//取出键
				initStaticStringObject(key, keystr);
				expiretime = getExpire(db, &key);
				if(expiretime != -1 && expiretime < now) continue;
				//根据值的类型，选择适当的命令来保存
				if(o->type == REDIS_STRING){
					char cmd[]="*3\r\n$3\r\nSET\r\n";
					if (rioWrite(&aof,cmd,sizeof(cmd)-1) == 0) goto werr;
					if (rioWriteBulkObject(&aof,&key) == 0) goto werr;
					if (rioWriteBulkObject(&aof,o) == 0) goto werr;
				} else if(o->type == REDIS_LIST){
					if (rewriteListObject(&aof,&key,o) == 0) goto werr;
				} else if(o->type == REDIS_SET){
					if (rewriteSetObject(&aof,&key,o) == 0) goto werr;
				}
				...
				//写入键的过期时间
				if(expiretime != -1) {
					char cmd[]="*3\r\n$9\r\nPEXPIREAT\r\n";
					if (rioWrite(&aof,cmd,sizeof(cmd)-1) == 0) goto werr;
					if (rioWriteBulkObject(&aof,&key) == 0) goto werr;
					if (rioWriteBulkLongLong(&aof,expiretime) == 0) goto werr;
				}
			}
			dictReleaseIterator(di);	//释放迭代器
		}
		if(fflush(fp)==EOF) goto werr;	
		if(aof_fsync(fileno(fp))==-1) goto werr;
		if(fclose(fp)==EOF) goto werr;
		//重命名
		if (rename(tmpfile,filename)==-1){
			...
		}
		
		return REDIS_OK;
	werr:
		fclose(fp);
		unlink(tmpfile);
		if(di) dictReleaseIterator(di);
		return REDIS_ERR;
	}

因为rewriteAppendOnlyFile函数生成的新AOF文件只包含还原当前数据库状态所必须的命令，所以新AOF文件不会浪费任何硬盘空间。
       
#### 后台重写
rewriteAppendOnlyFile函数虽然可以很好地完成一个新的AOF文件的任务，但是，该函数会进行大量的写入操作，所以调用这个函数的线程将被长时间阻塞，因为Redis服务器使用单个线程来处理命令请求，所以如果服务器直接调用rewriteAppendOnlyFile函数的话，那么在重写AOF文件期间，服务器将无法处理客户端发来的命令请求。     
所以Redis将AOF重写程序放到子进程里执行，这样做可以同时满足：（1）子进程在处理AOF重写的同时，服务器进程（父进程）可以继续处理命令请求。 （2）子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全性。当然，使用子进程也有一个问题，即子进程在进行AOF重写期间，服务器进程还需要继续处理命令请求，而新的命令可能会对现有的数据库状态进行修改，从而使得服务器当前的数据状态和重写后的AOF文件所保存的数据库状态不一致。      
为此，Redis服务器设置了一个AOF重写缓冲区，该缓冲区在服务器创建子进程之后开始使用，当Redis服务器执行完一个写命令之后，它会同时将这个写命令发送给AOF缓冲区和AOF重写缓冲区。当子进程完成AOF重写工作之后，它会向父进程发送一个信号，父进程在接到信号后，会调用一个信号处理函数：（1）将AOF重写缓冲区中的所有内容写入到新AOF文件中，这时新AOF文件所保存的数据库状态将和服务器当前的数据库状态一致；（2）对新的AOF文件进行改名，原子地覆盖现有的AOF文件，完成新旧两个AOF文件的替换。      
因此，在整个AOF后台重写的过程中，只有信号处理函数执行时会对服务器进程（父进程）造成阻塞，在其他时候，AOF后台重写都不会阻塞父进程，这将AOF重写对服务器性能造成的影响降到了最低。          
下面是相关的源代码实现。  

	int rewriteAppendOnlyFileBackground(void) {
		pid_t childpid;
		long long start;
	
		if(server.aof_child_pid != -1) return REDIS_ERR;
	
		start = ustime();
		if(childpid=fork()==0) {	//子进程
			char tmpfile[256];
			closeListeningSockets(0);	//关闭网络连接fd
			redisSetProcTitle("redis-aof-rewrite");		//为进程设置名字
			//创建临时文件，并进行AOF重写
			snprintf(tmpfile, 256, "temp-rewriteaof-bg-%d.aof", (int) getpid());
			if(rewriteAppendOnlyFile(tmpfile) == REDIS_OK){
				exitFromChild(0);	//发送重写成功信号
			} else{
				exitFromChild(1);
			}
		} else{					//父进程
			server.stat_fork_time = ustime()-start;
			...
			server.aof_selected_db = -1;			//feedAppendOnlyFile()
        	replicationScriptCacheFlush();
			return REDIS_OK;
		}
		return REDIS_OK;
	}
### 待续
Redis提供了RDB和AOF两种持久化方式，对于二者各自的优缺点和使用的场景，还有待分析。