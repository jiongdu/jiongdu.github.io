---
title: Redis之发布与订阅    
date: 2016-11-13 18:13:33   
tags: Redis
---
本文分析下Redis的发布与订阅功能，该功能有点类似网络中的多播或者广播的感觉，即当一个客户端向某个频道发布消息时，频道的所有订阅者和与这个频道相匹配的模式的订阅者都会收到该消息。     
Redis的发布与订阅功能分为频道的订阅和模式的订阅。
<!--more-->   
### 相关的数据结构
首先看一下服务器和客户端与发布/订阅相关的属性。   
	
	struct redisServer {
		...
		dict* pubsub_channels;	//字典，键为频道，值为链表，记录所有订阅这个频道的客户端
		list* pubsub_patterns;	//订阅模式列表，为pubsubPattern结构
		...
	}
	typedef struct redisClient {
		...
		dict* pubsub_channels;	//该客户端感兴趣的频道列表
		list* pubsub_patterns;	//该客户端感兴趣的模式
		...
	}redisClient;
	
	typedef struct pubsubPattern {
		redisClient* client;			//订阅模式的客户端
		robj* pattern;					//被订阅的模式
	}pubsubPattern;
	
### 频道的订阅与退订
频道的订阅与退订在Redis中对应的命令分别为SUBSCRIBE和UNSUBSCRIBE。     
首先学习SUBSCRIBE相关的源码。

	void subscribeCommand(redisClient* c){
		int j;
		for(j=1; j<c->argc; j++){
			pubsubSubscribeChannel(c, c->argv[j]);	//第二个参数开始的都是要订阅的频道
		}
	}
	  
	int pubsubSubscribeChannel(redisClient* c, robj* channel){
		dictEntry* de;
		list* client=NULL;
		int retval=0;
		//将channels放入客户端c->pubsub_channels的集合中
		if(dictAdd(c->pubsub_channels, channel, NULL) == DICT_OK){
			retval=1;
			incrRefCount(channel);		//与该channel关联的引用计数加1
			//在server中查找这个频道的客户端链表是否存在
			de = dictFind(server.pubsub_channels, channel);
			if(de==NULL){	//不存在创建一个
				clients = listCreate();
				dictAdd(server.pubsub_channels, channel, clients);	//添加进字典
				increRefCount(channel);
			}else{
				clients = dictGetVal(de);	//如果已经存在，则直接获取频道所对应的客户端链表
			}
			
			listaddNodeTail(clients, c);	//将客户端添加到链表的末尾
		}
		//回复客户端
		addReply(c, shared.mbulkhdr[3]);
		addReply(c, shared.subscribebulk);
		addReply(c, channel);	
		addReplyLongLong(c,dictSize(c->pubsub_channels)+listLength(c->pubsub_patterns));
		
		return retval; 
	}
这样，客户端就订阅了频道。    
UNSUBSCRIBE命令的行为和SUBSCRIBE的行为正好相反，当一个客户端退订某个或某些频道的时候，服务器将从pubsub\_channels中解除客户端与被退订频道之间的关联。

### 模式的订阅与退订

如前文所述，服务器将所有模式的订阅关系都保存在服务器状态的pubsub\_patterns属性里面。  
模式的订阅与退订在Redis中对应的命令分别为PSUBSCRIBE和PUNSUBSCRIBE。   
下面是PSUBSCRIBE相关的源代码。    

	int pubsubSubscribePattern(redisClient* c, robj* pattern){
		int retval=0;
		
		//看客户端是否已经订阅了这个模式
		if(listSearchKey(c->pubsub_patterns, pattern)==NULL){	//没有
			retval=1;
			pubsubPattern* pat;
			listAddNodeTail(c->pubsub_patterns, pattern);
			incrRefCount(pattern);
			
			//创建并设置新的pubsubPattern结构
			pat = zmalloc(sizeof(*pat));
			pat->pattern = getDecodedObject(pattern);
			pat->client = c;

			listAddNodeTail(server.pubsub_patterns, pat);
		}
		//回复客户端
		...
	}
同频道订阅操作类似，只是在数据结构上有些许差异。    
模式的退订命令PUNSUBSCRIBE是PSUBSCRIBE命令的反操作：当一个客户端退订某个或某些模式的时候，服务器将在pubsub\_patterns链表查找并删除那些pattern属性为被退订模式，并且client属性为执行退订命令的客户端的pubsubPattern结构。

### 发送消息

当一个Redis客户端执行PUBLISH <channel> <message>命令将消息message发送给频道channel的时候，服务器需要执行以下两个动作：    
（1）将消息message发送给channel频道的所有订阅者；     
（2）如果有一个或多个模式pattern与频道channel相匹配，那么将消息message发送给pattern模式的订阅者。     
下面是相关的源码。
	
	int pubsubPublishMessage(robj* channel, robj* message) {
		int receives=0;
		dictEntry* de;
		listNode* ln;
		listIter li;

		de = dictFind(server.pubsub_channels, channel);	//查找channel
		if(de){
			list* list = dictGetVal(de);	//得到客户端链表
			listNode* ln;
			listIter* li;
		
			listRewind(list, &li);
			while(ln = listNext(&li)!=NULL){	//遍历客户端链表，将消息发送给他们
				redisClient* c = ln->value;
				addReply(c,shared.mbulkhdr[3]);
				addReply(c,shared.messagebulk);
				addReplyBulk(c,channel);
				addReplyBulk(c,message);
				receives++;		//接收客户端计数
			}
		}

		//将消息发送给与频道匹配的模式
		if(listLength(server.pubsub_patterns)){
			listRewind(server.pubsub_patterns,&li);
			channel = getDecodedObject(channel);
			while ((ln = listNext(&li)) != NULL) {
				if (stringmatchlen((char*)pat->pattern->ptr,
                                sdslen(pat->pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)){	//如果模式和channel匹配
					//给所以订阅该模式的客户端发送消息
					addReply(pat->client,shared.mbulkhdr[4]);
                	addReply(pat->client,shared.pmessagebulk);
                	addReplyBulk(pat->client,pat->pattern);
                	addReplyBulk(pat->client,channel);
                	addReplyBulk(pat->client,message);
					
					receives++;		//接收客户端计数
				}
			}
		}
		return receives;
	}

### 查看订阅信息
此外，Redis还支持PUBSUB命令，客户端用以查看频道或者模式的相关信息，比如某个频道目前有多少订阅者、某个模式目前有多少订阅者等。