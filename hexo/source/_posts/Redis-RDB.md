---
title: Redis之RDB持久化    
date: 2016-11-27 14:22:16
tags: Redis
---

Redis是内存数据库，它将自己的数据库状态储存在内存里面，所以，如果不想办法将储存在内存中的数据库状态保存到磁盘里面，那么一旦服务器进程退出，服务器中的数据库状态也会消失不见。
<!--more--> 
Redis提供了RDB持久化功能，该功能可以将Redis在内存中的数据库状态保存到磁盘里面，避免数据意外丢失。RDB持久化可以手动执行，也可以根据服务器配置选项定期执行。RDB文件是一个压缩的二进制文件，通过该文件可以还原生成RDB文件时的数据库状态。      

### RDB文件的创建与载入     
有两个命令Redis命令可以用于生成RDB文件，一个是SAVE（rdb.c/rdbSave），另一个是BGSAVE(rdb.c/rdbSaveBackground)。     
SAVE命令会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在服务器进程阻塞期间，服务器不能处理任何命令请求。而BGSAVE命令会派生一个子进程，然后由子进程负责创建RDB文件，服务器进程（父进程）继续处理命令请求。    
创建RDB文件的实际工作由rdb.c/rdbSave函数完成，SAVE命令和BGSAVE命令以不同的方式调用这个函数，关键代码如下：

	int rdbSave(char* filename) {
		dictIterator *di = NULL;
		dictEntry *de;
		char tmpfile[256];
		char magic[10];
		int j;
		long long now = mstime();
		FILE *fp;
		rio rdb;
		uint64_t cksum;

		snprintf(tmpfile, 256, "temp-%d.rdb", (int)getpid());	//创建临时文件，pid为标识
		fp = fopen(tmpfile, "w");
		if(!fp) {
			redisLog(REDIS_WARNING, "Failed opening .rdb for saving : %s", strerror(errno));
			return REDIS_ERR;
		}

		rioInitWithFile(&rdb, fp);		//初始化I/O

		if(server.rdb_checksum)			//设置校验和函数
			rdb.update_cksum = rioGenericUpdateChecksum;
		snprintf(magic, sizeof(magic), "REDIS%04d", REDIS_RDB_VERSION);
		if(rdbWriteRaw(&rdb, magic, 0)==-1) goto err;	//写入RDB版本号
		
		for(j=0; j<server.dbnum; j++){
			redisDb *db = server.db+j;	//指向数据库
			dict *d = db->dict;			//数据库键空间
			if(dictSize(d) == 0) continue;	//跳过空数据库
			di = dictGetSafeIterator(d);
			if(!di){
				fclose(fp);			
				return REDIS_ERR;
			}
			//写入SELECTDB
			if (rdbSaveType(&rdb,REDIS_RDB_OPCODE_SELECTDB) == -1) goto werr;
			//对数据库ID编码后写入
			if (rdbSaveLen(&rdb,j) == -1) goto werr;
			//遍历数据库，写入每个键值对数据
			while((de = dictNext(di)) != NULL){
				sds keystr = dictGetKey(de);
				robj key, *o = dictGetValue(de);
				long long expire;
				
				initStaticStringObject(key, keystr);
				expire = getExpire(db, &key);
				if(rdbSaveKeyValuePair(&rdb, &key, o, expire, now)==-1) goto err;
			}
			dictReleaseIterator(di);
		}
		di = NULL;
		if (rdbSaveType(&rdb,REDIS_RDB_OPCODE_EOF) == -1) goto werr;		//写入EOF
		//校验和
		cksum = rdb.cksum;
    	memrev64ifbe(&cksum);
    	rioWrite(&rdb,&cksum,8);
		//冲洗缓存
		if (fflush(fp) == EOF) goto werr;
    	if (fsync(fileno(fp)) == -1) goto werr;
    	if (fclose(fp) == EOF) goto werr;
		//命名
		if (rename(tmpfile,filename) == -1) {
        	redisLog(REDIS_WARNING,"Error moving temp DB file on the final destination: %s", strerror(errno));
        	unlink(tmpfile);
        	return REDIS_ERR;
    	}
		//清零数据库脏状态
    	server.dirty = 0;
    	//记录最后一次完成 SAVE 的时间
    	server.lastsave = time(NULL);
    	//记录最后一次执行 SAVE 的状态
    	server.lastbgsave_status = REDIS_OK;
		
		return REDIS_OK;

	werr:
		fclose(fp);
		unlink(tmpfile);
		...
	}

### RDB文件的结构
从上文的代码分析中可以看出RDB文件的结构。    
![](http://i.imgur.com/fnagdgd.png)    
RDB文件的最开头部分是REDIS，通过这五个字符，程序可以在载入文件时，快速检查所载入的文件是否是RDB文件。        
db_version长度为4字节，它的值是一个字符串表示的整数，该整数记录了RDB文件的版本号，比如“0006”代表RDB文件的版本为第六版。  
#### databases部分
一个RDB文件的databases部分可以保存任意多个非空数据库。每个非空数据库在RDB文件中都可以保存为SELECTDB、db\_number、key\_value\_pairs三个部分，如下图所示。
![](http://i.imgur.com/vjF6XaN.png)       
![](http://i.imgur.com/8NLANJY.png)     
SELECTDB常量的长度为1字节，当读入程序遇到这个值的时候，它知道接下来要读入的将是一个数据库号码。db_number保存着一个数据库号码，根据号码的大小不同，该部分的长度可以是1、2或者5字节。key\_value\_pairs部分保存了数据库中的所有键值对数据，如果键值对带有过期时间，那么过期时间也会和键值对保存在一起。

##### key\_value\_pairs部分
RDB文件中的每个key\_value\_pairs部分都保存了一个或以上数量的键值对，如果键值对带有过期时间的话，那么键值对的过期时间也会被保存在内。       
不带过期时间的键值对在RDB文件中由TYPE、key、value三部分组成。     
![](http://i.imgur.com/NYbtrsP.png)    
TYPE记录了value的类型，长度为1字节。可能的取值包括REDIS\_RDB\_TYPE\_STRING、REDIS\_RDB\_TYPE\_LIST、REDIS\_RDB\_TYPE\_SET、REDIS\_RDB\_TYPE\_HASH等，每个TYPE常量代表了一种对象类型或者底层编码，当服务器读入RDB文件中的键值对数据时，程序会根据TYPE的值来决定如何读入和解释value的数据。        
带有过期时间的键值对在RDB文件中的结构如图10-17所示。     
![](http://i.imgur.com/vI9BEm5.png)      
带有过期时间的键值对中的TYPE、key、value三部分的意义和前面不带过期时间键值对三部分的意义完全相同，新增的EXPIRETIME_\MS常量的长度为1字节，它的作用是告知程序，接下来要读入的将是一个以毫秒为单位的过期时间；ms是一个8字节长的带符号整数，记录着以ms为单位的时间戳，即过期时间。  

### 自动间隔性保存

因为BGSAVE命令可以在不阻塞服务器进程的情况下执行，所以Redis允许用户通过设置服务器配置的save选项，让服务器每隔一段时间自动执行一次BGSAVE命令。用户可以通过save选项设置多个保存条件，但只要其中任意一个条件满足，服务器就会执行BGSAVE命令。     
当Redis服务器启动时，用户可以通过指定配置文件或者传入启动参数的方式设置save选项，如果用户没有主动设置save选项，那么服务器会为save选项设置默认条件：     
	
	save 900 1
	save 300 10
	save 60 10000

接着，服务器程序会根据save选项所设置的保存条件，设置服务器状态redisServer结构的saveparams属性。      

	struct redisServer {
		struct saveparam *saveparams;
		...
	}
	struct saveparams {
		time_t seconds;		//秒数
		int changes;		//修改数
	}

### RDB文件的载入

RDB文件的载入工作(rdb.c/rdbLoad)是在服务器启动时自动执行的，所以Redis并没有专门用于载入RDB文件的命令，只要Redis服务器在启动时监测到RDB文件存在，它就会自动载入RDB文件。    
另外，因为AOF文件（见下文分析）的更新频率通常比RDB文件的更新频率高，所以：    
（1）如果服务器开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态；  
（2）只有在AOF持久化功能处于关闭状态时，服务器才会使用RDB文件来还原数据库状态。