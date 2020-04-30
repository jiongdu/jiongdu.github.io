---
title: C++ STL空间配置器
date: 2016-10-9 13:54:15
tags: C++
---
最近准备结合《STL源码剖析》把学习的STL知识整理一遍，加深自己的理解的同时，也方便以后需要的时候查看。    
<!--more-->
首先总结的是STL中的内存分配---空间配置器，从STL的运用角度而言，空间配置器是最不需要介绍的东西，因为用户使用过程中不会直接与其打交道，但是，从STL的实现角度看，空间配置器又是最重要的知识点之一，因为整个STL的操作对象都存放在容器之内，而容器一定需要配置空间以放置数据。      
C++ STL配置器分为两层配置器，当请求的内存大于128bytes时，视为“足够大”，用第一层配置器分配内存，当请求的内存小于128bytes时，视为“过小”，调用第二级配置器，第二级配置器采用复杂的内存池整理方式，而不再求助于第一级配置器。
### 第一级配置器
首先来看下主要的源代码：     
	
	template <int inst>
	class __malloc_alloc_template{
	private:
		static void *oom_malloc(size_t);	//malloc调用内存不足
		static void *oom_realloc(void*, size_t);	//realloc调用内存不足
		static void (*__malloc_alloc_oom_handler)();	//错误处理函数
	public:
		static void* allocate(size_t){
			void *result = malloc(n);		//第一级配置器直接调用malloc()
			if(0 == result) result = oom_malloc(n);
			return result;
		}
		static void* deallocate(void *p, size_t){
			free(p);			
		}
		static void *reallocate(void *p, size_t/* old_sz */, size_t new_sz){
			void *result = realloc(p, new_sz);	//第一级配置器直接调用realloc()
			if(0 == result) result = oom_realloc(p, new_sz);
			return result;	
		}
		static void (*set_malloc_handler(void (*f)()))(){	//设置错误处理函数
			void (* old)() = __malloc_alloc_oom_handler;
			__malloc_alloc_oom_handler = f;
			return(old);	
		}
	};
	
	template <int inst>
	void* __malloc_alloc_template<inst>::oom_alloc(size_t n){
		void (* my_alloc_handler)();     //声明函数指针
		void *result;					 //返回的内存指针
		for(;;){ 						 //循环，直到成功
			my_alloc_handler = __malloc_alloc_oom_handler;	
			if(0 == my_alloc_handler){ __THROW_BAD_ALLOC; }		//抛出异常
			(*my_alloc_handler)();
			result = malloc(n);			//再重新分配内存
			if(result) return(result);		
		}
	}

第一级配置器相对简单，其使用malloc(), free(), realloc()等C函数执行实际的内存分配，释放，重新配置等操作。此外，这个配置器提供了当内存配置错误时的处理函数oom_malloc，这个函数会调用\_\_malloc\_alloc\_oom\_handler()这个错误处理函数，去企图释放内存，然后重新调用malloc分配内存。如此循环，直到分配成功，返回指针。
### 第二级配置器
STL第二级配置器的做法是，如果区块够大，超过128bytes时，就移交第一级配置器处理。当区块小于128bytes时，则以内存池管理，即每次配置一大块内存，并维护对应之自由链表，下次若再有相同大小的内存需求，就直接从free lists中拔出，而如果客户端释还小额区块，就由配置器回收到free lists中。       
为了方便管理，SGI第二级配置器会主动将任何小额区块的内存需求量上调至8的倍数（例如客端要求30bytes，就会自动调整为32bytes），并维护16个free lists，各自管理大小分别为8,16,24,32,40,48,56,64,72,80,88,96,104,112,120,128的小额区块。      
free-lists的节点结构定义如下：
	
	union obj {
		union obj *free_list_link;     //指向下一个内存的地址
		char client_data[1];		   //内存的首地址 
	};
根据union的特性，obj的内存布局应该如下所示：     
![](http://i.imgur.com/IE7fZAU.png)
从起第一字段观之，obj被视为一个指针，指向相同形式的另一个obj，从其第二字段观之，obj可被视为一个指针，指向实际区块。这样一物二用的结果是，不会为了维护链表所必须的指针而造成内存的另一种浪费。      
下面来看一下相关的源代码。   

	enum {__ALIGN = 8}；
	enum {__MAX_BYTES = 128};
	enum {__NFREELISTS = __MAX_BYTES/__ALIGN};

	template <bool threads, int inst>
	class __default_alloc_template{
		...
		static size_t ROUND_UP(size_t bytes){		//将bytes上调至8的倍数
			return (((bytes)+__ALIGN-1) & ~(__ALIGN-1));
		}
		static obj* volatile free_list[__NFREELISTS];	//内存池链表
		static size_t FREELIST_INDEX(size_t bytes){	
			return (((bytes)+__ALIGN-1)/__ALIGN-1);		//根据区块大小，决定使用第n号free-list
		}
		...	
	};
	
	static void* allocate(size_t n){
		obj * volatile * my_free_list;
		obj * result;
		if(n > (size_t)__MAX_BYTES){
			return (malloc_alloc::allocate(n));
		}
		my_free_list = free_list + FREELIST_INDEX(n);	//寻找合适的那个free_list
		result = *my_free_list;
		if(result == 0){		//没找到可用的free_list,准备重新填充free_list
			void* r = refill(ROUND_UP(n));
			return r;
		}
		*my_free_list = result->free_list_link;		//调整free_list
		return (result);
	}
具体操作如下图所示（摘自《STL源码剖析》）。当有内存请求到达时（第二级配置器），先找到负责这个内存大小的数据元素指向的内存链表，取出第一块内存，然后把数据元素(obj指针)指向第二块内存的首地址。
![](http://i.imgur.com/VrO5VMA.png)
此外，当程序释放这块内存时，第二级配置器还负责回收这块内存，等下次有请求时，可以直接使用这块内存。示意图（摘自《STL源码剖析》）如下：
![](http://i.imgur.com/um5jnaS.png)
先计算这块内存属于哪个数组元素负责，然后将这块回收的内存放置链表的第一个位置，这块内存的下一块内存为这个链表原先的第一块内存。源代码如下；
	
	static void deallocate(void *p, size_t n){
		obj *q = (obj *)p;
		obj * volatile * my_free_list;
		
		if(n > (size_t)__MAX_BYTES) {    //大于128bytes,调用第一级配置器回收
			malloc_alloc::deallocate(p, n);
			return;
		}
		my_free_list = free_list + FREELIST_INDEX(n);	//找到负责这块内存的数据元素
		q ->free_list_link = *my_free_list;	 	
		*my_free_list = q;
	}
### 重新填充free lists
在上面的allocate()中，当发现free list中没有可用区块了时，就调用refill()，准备为free list重现填充空间。新的空间将取自内存池（经由chunk_alloc()完成）。缺省取得20个新节点（新区块），但万一内存池空间不足，获得的区块数可能小于20。例如，如果请求内存为32bytes，此时内存链表中没有足够的内存了，那么refill会分配20块32bytes的内存块，然后把第一块返回给程序，其他19块由数组相应链表管理。相应的源代码为：

	template <bool threads, int inst>
	void* __default_alloc_template<threads, inst>::refill(size_t n){
		int nobjs = 20;
		char *chunk = chunk_alloc(n, nobjs);   //nobjs: pass by reference
		obj * volatile * my_free_list;
		obj * result;
		obj * current_obj, next_obj;
		int i;
		
		if(1 == nodejs) return chunk;	//如果只返回一块内存，直接返回
		my_free_list = free_list + FREELIST_INDEX(n);
		
		result = (obj*)chunk;		//不止一块内存，取出第一块内存
		*my_free_list = next_obj = (obj *)(chunk + n );   //数组元素链表指针指向第二块内存
		for(i=1; ;i++){
			curent_obj = next_obj;
			next_obj = (*obj)((char*)next_obj+n);
			if(nobjs - 1 ==i){
				current_obj->free_list_link = 0;
				break;
			}else{
				current_obj->free_list_link = next_obj;
			}
		}
		return(result);
	}            
接下来分析真正从内存池获取内存的函数chunk_alloc。
	
	template <bool threads, int inst>
	char* __default_alloc_template<threads, inst>::chunk_alloc(size_t size, int& nobjs){
		char* result;
		size_t total_bytes = size * nobjs;
		size_t bytes_left = end_free - start_free;		//内存池剩余的内存
		
		if(bytes_left >= total_bytes){		//当内存持内存足够时
			result = start_free;
			start_free += total_free;
			return(result);
		}else if(bytes_left >= size){		//内存池不能满足total，但是可以满足一块以上
			nobjs = bytes_left/size;
			total_bytes = size * nobjs;
			result = start_free;
			start_free += total_bytes;
			return(result);
		}else{				//试着让内存池的残余零头还有利用价值
			size_t bytes_to_get = 2*total_bytes + ROUND_UP(heap_size >> 4);
			if(bytes_left > 0){
				obj * volatile * my_free_list = free_list + FREELIST_INDEX(bytes_left);
				((obj *)start_free) -> free_list_link = *my_free_list;	//调整free list，将内存池中的残余空间编入
				*my_free_list = (*obj)start_free;
			}
			//调用malloc从内存分配
			start_free = (char*)malloc(bytes_to_get);
			if(0 == start_free){				//当系统内存不足时
				int i;
				obj * volatile * my_free_list, *p;
				for(i=size;i<=__MAX_BYTES;i+=__ALIGN){
					my_free_list = free_list + FREELIST_INDEX(i);
					p = *my_free_list;
					if(0 != p){
						*my_free_list = p->free_list_link;
						start_free = (char*)p;
						end_free = start_free + i;
						return(chunk_alloc(size, nobjs));
					}
				}
				end_free = 0;		//从其他链表也没获取到内存，到处都没内存可用了
				start_free = (char*)malloc_alloc::allocate(byte_to_get);		//调用第一级配置器，因为有错误处理函数，最后的补救办法了
			}
			heap_size += bytes_to_get;
			end_free = start_free + bytes_to_get;
			return(chunk_alloc(size, nobjs));
		}
	}
chunk\_alloc()函数以end\_free-start\_free来判断内存池的水量，如果水量充足，就直接调出20个区块返回给free list，如果水量不足以提供20个区块，但还足够供应一个以上的区块，就拨出这不足20个区块的空间出去，这时候pass by reference的nobjs参数被修改为实际能够供应的区块数。如果内存池连一个区块空间都无法供应，此时便利用malloc()从heap中配置内存，为内存注入源头活水以应付需求。其过程如下图所示：
![](http://i.imgur.com/SCDG2Gu.png)

### 对象内容的构造与析构
我们知道，C++中的new操作符包含两阶段的操作：（1）调用::operator new配置内存；（2）调用构造函数构造对象内容，所以，为了精密分工，STL allocator决定将这两阶段 操作区分开来，内存配置操作由alloc::allocate()负责，对象构造函数由::construct()负责。同理，对象的析构与内存释放也是由两部分操作组成。

### 使用配置器
最后，来看下配置器是如何使用的。      

	template<class T, class Alloc>
	class simple_alloc {
	public:
		static T *allocate(size_t n)
			{ return 0 == n ? 0 : (T*)Alloc::allocate(n * sizeof(T)); }
		static T *allocate(void)
			{ return 0 == n ? 0 : (T*)Alloc::allocate(sizeof(T)); }
		static void deallocate(T* p, size_t n)
			{ if(0 != n) Alloc::deallocate(p, n * sizeof(T)); }
		static void deallocate(T* p)
			{ Alloc::deallocate(p, sizeof(T)); }
	};        
simple\_alloc类封装了Alloc的分配和回收内存函数，并提供了四个用于内存操作的函数接口。
	
	template <class T, class Alloc = alloc>		//alloc被默认为第二级配置器
	class vector{
	public:
		typedef T value_type;
		...
	protected:
		typedef simple_alloc<value_type, Alloc> data_allocator;	
	}
可以看出，vector内嵌了data_allocator类型，当需要分配内存时，调用simple_alloc的成员方法即可。