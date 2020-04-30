---
title: Redis之数据库的实现
date: 2016-11-17 19:46:03
tags: Redis
---
本文对Redis服务器的数据库实现进行说明，包括服务器保存数据库的方法，数据库保存键值对的方法，以及针对数据库的添加、删除、查看等操作的实现方法，最后，会对服务器保存键的过期时间的方法和服务器自动删除过期键的方法进行分析。
<!--more-->
### 服务器中的数据库结构

Redis服务器将所有数据库都保存在服务器状态redis.h/redisServer结构的db数组中，db数组的每个项都是一个redis.h/redisDb结构，每个redisDb结构代表一个数据库。     

	struct redisServer {
		redisDb* db;	//	保存着服务器中的所有数据库
		int dbnum;		//服务器的数据库数量
		...	
	}
	
	typedef struct {
		dict *dict;			//保存着数据库中的所有键值对	
		dict *expires;		//键的过期时间，键为数据库键，值为过期事件
		dict *blocking_keys;	//正处于阻塞状态的键和相应的client
		dict *ready_keys;		//可以解除阻塞的键和相应的client
		dict *watched_keys;		//正在被WATCH命令监视的键和相应的client
		struct evictionPoolEntry *eviction_pool;
		int id;		//数据库ID
		long long avg_ttl;		//数据库键的平均TTL，统计信息
	} redisDb;

在初始化服务器时，程序会根据服务器状态的dbnum属性（配置文件database选项）来决定应该创建多少个数据库，默认情况下，该选项的值为16。
	
### 切换数据库

每个Redis客户端都有自己的目标数据库，默认情况下，Redis客户端的目标数据库为0号数据库，客户端可以通过执行SELECT命令来切换目标数据库。       
客户端redisClient结构的db属性记录了客户端当前的目标数据库，该属性是一个指向redisDb结构的指针，该指针指向redisServer.db数组的其中一个元素，即客户端的目标数据库。      
用户执行SELECT命令切换数据库，Redis便修改redisClient.db指针，让它指向服务器中的不同数据库。 

### 数据库键空间

如前文所述，redisDb结构化的dict字典保存了数据库中的所有键值对，该字典被称为键空间。    
键空间和用户所见的数据库是直接对应的：   
（1）键空间的键也就是数据库的键，每个键都是一个字符串对象；      
（2）键空间的值也就是数据库的值，每个值可以是字符串对象、列表对象、哈希表对象、集合对象和有序集合对象中的任何一种Redis对象。       
因为数据库的键空间是一个字典，所以所有针对数据库的操作，比如添加一个键值对，删除一个键值对，或者在数据库中获取某个键值对等，实际上都是通过对键空间字典进行操作来实现的。

#### 读写键空间的维护操作

当使用Redis命令对数据库进行读写时，服务器不仅会对键空间执行指定的读写操作，还会执行一些额外的维护操作，包括：      
（1）在读取一个键后，服务器会根据键是否存在来更新服务器的键空间命中(hit)次数或键空间不命中次数（miss）次数，这两个值可以在INFO status命令的keyspace\_hits属性和keyspace\_misses属性中查看；            
（2）在读取一个键后,服务器会更新键的LRU事件，这个值可以用来计算键的闲置时间；      
（3）如果服务器在读取一个键时发现该键已经过期，那么服务器会先删除这个过期键，再执行余下的其他操作；     
（4）如果有客户端使用WATCH命令监视了某个键，那么服务器在对被监视的键进行修改之后，会将这个键标记为脏，从而让食物程序注意到这个键已经被修改过；     
（5）...

### 设置键的生存时间或过期时间

客户端可以使用EXPIRE或PEXPIRE命令以秒或者毫秒精度为数据库中的某个键设置生存时间，在经过指定的秒数或者毫秒数之后，服务器就会自动删除生存时间为0的键。    
客户端可以使用EXPIREAT或PEXPIREAT命令以秒或者毫秒精度为数据库中的某个键设置过期时间，过期时间是一个UNIX时间戳，当键的过期时间来临时，服务器就会自动从数据库中删除这个键。  
下面是部分源代码(db.c)。
	
	//下面四个函数对应文中四个命令，最终都调用expireGenericCommand函数
	void expireCommand(redisClient *c) {
		expireGenericCommand(c,mstime(),UNIT_SECONDS);
	}    

	void expireatCommand(redisClient *c) {
		expireGenericCommand(c,0,UNIT_SECONDS);
	}

	void pexpireCommand(redisClient *c) {
		expireGenericCommand(c,mstime(),UNIT_MILLISECONDS);
	}

	void pexpireatCommand(redisClient *c) {
		expireGenericCommand(c,0,UNIT_MILLISECONDS);
	}

	void expireGenericCommand(redisClient *c, long long basetime, int unit) {
		robj *key=c->argv[1],*param=c->argv[2];
		long long when;
	
		if(getLongLongFromObjectOrReply(c, param, &when, NULL)!=REDIS_OK)		//取出when参数
			return;
		if(unit == UNIT_SECONDS) when *= 1000;	//传入的时间是以秒为单位的
		when += basetime;

		if(lookupKeyRead(c->db, key) == NULL) {	//取出键
			addReply(c, shared.czero);
			return;
		}
		
		// 在载入数据时，如果服务器为附属节点时，即使EXPIRE的TTL为负数，或者EXPIREAT提供的时间戳已经过期，服务器也不会主动删除这个键，而是等待主节点发来显示的DEL命令
		if(when <= mstime() && !server.loading && !server.masterhost) {
			robj *aux;
			redisAssertWithInfo(c, key, dbDelete(c->db, key));
			server.dirty++;
		
			aux = createStringObject("DEL", 3);
			rewriteClientCommandVector(c,2,aux,key);
       		decrRefCount(aux);

			signalModifiedKey(c->db,key);
        	notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC,"del",key,c->db->id);
			
			addReply(c, shared.cone);
			return;
		}else {			//设置键的过期事件
			setExpire(c->db, key, when);
			addReply(c, shared.cone);
			signalModifiedKey(c->db, key);
			notifyKeyspaceEvent(REDIS_NOTIFY_GENERIC, "expire", key, c->db->id);

			server.dirty++;
			return;
		}
	}

### 过期键删除策略

#### 三种删除策略

1. 定时删除：定时删除对内存是最友好的，在设置键的过期时间的同时，创建一个定时器，让定时器在键过期时间来临时，立即执行对键的删除操作。定时删除的问题是对CPU时间不友好，在过期键较多的情况下，删除过期键这一行为可能会占用相当一部分CPU时间，这样会对服务器的响应时间和吞吐量造成影响。     
2. 惰性删除：惰性删除策略对CPU时间来说是最友好的，因为程序只会在取出键时才会对键进行过期检查，这样确保删除过期键的操作只会在非做不可的情况下进行，不会在删除其他无关的过期键上花费任何时间。但惰性删除的缺点是对内存时最不友好的，如果数据库中有很多的过期键，而这些键又恰好没有访问到，那么它们也许永远也不会被删除，这样无用的垃圾数据占用了大量的内存。       
3. 定期删除：定期删除策略是前两种策略的一种折中，每隔一段时间执行一次删除过期键操作，并通过限制删除操作执行时长和频率来减少删除操作对CPU时间的影响。但是定期策略的难点也正是在于确定操作执行的时长和频率。   

#### Redis的过期键删除策略

Redis服务器采用的是惰性删除和定期删除两种策略，通过配合这两种删除策略，服务器可以很好地在合理使用CPU时间和避免内存浪费之间取得平衡。    
Redis过期键的惰性删除策略由db.c/expireIfNedded函数实现，所有读写数据库的Redis命令在执行之前都会调用expireIfNedded函数对输入键进行检查：如果输入键已经过期，那么该函数将输入键从数据库中删除。   
	
	int expireIfNeeded(redisDb *db, robj *key) {
		mstime_t when = getExpire(db, key);		//键的过期时间
		mstime_t now;
		if(when < 0) return 0;
		if(server.loading) return 0;	//服务器正在进行载入，返回

		now = server.lua_caller ? server.lua_time_start : mstime();
		//当服务器运行在replication模式时，附属节点并不主动删除 key，它只返回一个逻辑上正确的返回值，真正的删除操作要等待主节点发来删除命令时才执行 
		if (server.masterhost != NULL) return now > when;
		if(now <= when) return 0;		//未过期
		
		server.stat_expiredkeys++;
		propagateExpire(db, key);		//向AOF文件和附属节点传播过期信息
		notifyKeyspaceEvent(REDIS_NOTIFY_EXPIRED, "expired", key, db->id);

		return dbDelete(db, key);		//从数据库中删除键
	}


expireIfNeeded函数就像一个过滤器，它可以在命令真正执行之前，过期掉过期的输入键，从而避免命令接触到过期键。       
Redis过期键的定时删除策略由redis.c/activeExpireCycle函数实现，该函数在redis.c/beforesleep和redis.c/serverCron两个函数中调用，且分别对应ACTIVE\_EXPIRE\_CYCLE\_FAST（快速过期）和ACTIVE\_EXPIRE\_CYCLE\_SLOW(正常过期)两种处理模式。以下是关键部分源码。      

	void activeExpireCycle(int type) {
		//静态变量，累计函数连续执行时的数据
		static unsigned int current_db = 0;
		static int timelimit_exit = 0;
		static long long last_fast_cycle = 0;
		//默认每次处理的数据库数量
		unsigned int dbs_per_call = REDIS_DBCRON_DBS_PER_CALL;
		long long start=ustime(), timelimit;

		if(type == ACTIVE_EXPIRE_CYCLE_FAST) { 		//快速模式
			if (!timelimit_exit) return;
			if (start < last_fast_cycle + ACTIVE_EXPIRE_CYCLE_FAST_DURATION*2) return;
			last_fast_cycle = start;		//执行快速处理
		}
		if (dbs_per_call > server.dbnum || timelimit_exit)
        	dbs_per_call = server.dbnum;

		//处理的时间上限
		timelimit = 1000000*ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC/server.hz/100;	
    	timelimit_exit = 0;
    	if (timelimit <= 0) timelimit = 1;
		if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        	timelimit = ACTIVE_EXPIRE_CYCLE_FAST_DURATION;

		for(j=0; j<dbs_per_call, j++) {
			int expired;
			redisDb *db = server.db+(current_db % server.dbnum); //当前要处理的数据库
			current_db++;
			do {
				...
				if ((num = dictSize(db->expires)) == 0) {	//数据库中带过期时间的键的数量
                	db->avg_ttl = 0;
                	break;
            	}
				slots = dictSlots(db->expires);		//数据库键值对的数量
				if (num && slots > DICT_HT_INITIAL_SIZE && (num*100/slots < 1))  //数据库的使用率低于1%，跳过
					break;
				while(num--){
					dictEntry *de;
                	long long ttl;
					//随机取出一个带过期时间的键
					if ((de = dictGetRandomKey(db->expires)) == NULL) break;
					ttl = dictGetSignedIntegerVal(de)-now;		//计算TTL
					if (activeExpireCycleTryExpire(db,de,now)) //如果键过期，则删除它，并将expired计数器加1
						expired++;
					...
				}
				...
			} while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
	
		}
	}

### 其他

Redis的数据库功能很强大，也比较复杂，本文只是对其中的一些基本功能做了简单说明和分析，其他的一些功能和实现，比如持久化方式和复制功能对过期键的处理，将在后面的文章中继续学习。