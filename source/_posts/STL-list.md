---
title: C++ STL之list
date: 2016-10-28 19:33:14
tags: C++
---
上一篇文章总结了vector的结构和实现原理，今天来看看STL中另一个重要的容器：list。
<!--more-->
相比于vector的连续线性空间，list显得复杂得多，它的好处是每次插入或删除一个元素，就配置或释放一个元素空间。因此，list对于空间的运用有绝对的精准，一点都不浪费。    
### list的节点
我们知道，list本身和list的节点是不同的结构，需分开设计。以下是STL list的节点结构：	
	
	template <class T>
	struct __list_node {
		typedef void* void_pointer;
		void_pointer prev;
		void_pointer next;
		T data;	
	};
可以看出，STL list是一个双向链表，有两个指针，分别指向前向节点和后向节点。  
### list的迭代器
list不能够像vector一样以普通指针作为迭代器，因为其节点不保证在存储空间连续存在。并且，由于list提供的是一个双向链表，迭都代器必须具备前移、后移的能力，所以list提供的是Bidirectional Iterators.    
list有一个重要的性质：插入操作和结合操作都不会造成原有的list迭代器失效。这在vector是不成立的，因为vector的插入操作可能造成记忆体重新配置，导致原有的迭代器全部失效。甚至list的元素删除操作，也只有“指向被删除元素”的那个迭代器失效，其他迭代器不受任何影响。   
   
	template<class T, class Ref, class Ptr>
	struct __list_iterator {
		typedef __list_iterator<T, &T, T*> iterator;
		typedef __list_iterator<T, Ref, Ptr> self;

		typedef __list_node<T>* link_type;
		link_type node;						//list的节点:node 
		
		//迭代器的构造函数
		__list_iterator(link_type x) : node(x) {}
  		__list_iterator() {}
  		__list_iterator(const iterator& x) : node(x.node) {}	
		
		reference operator*() const { return (*node).data; }
		pointer operator->() const { return &(operator*()); }

		self& operator++() {					//先加1，再返回，类似++i
			node = (link_type)((*node).next);
			return *this;
		}
		self operator++(int) {					//类似于i++
			self tmp = *this;
			++*this;
			return tmp;
		}
		...
	};
以上是list的迭代器的设计，只实现了迭代器的++，--，取值，成员调用四个基本操作，没有像vector迭代器那样的+n操作，主要是因为地址不连续。       
### list的结构
list不仅是一个双向链表，而且还是一个环状双向链表。所以，只需要一个指针，便可以完整表现整个链表：   
	
	template <class T, class Alloc = alloc>
	class list {
	protected:
		typedef __list_node<T> list_node;
	public:
		typedef list_node* link_type;
	protected:
		link_type node;		
	};
如下图所示，如果让指针node指向刻意置于尾端的一个空白节点，node便能符合STL对于“前闭后开”区间的要求，成为last迭代器。    
![](http://i.imgur.com/rClntKM.png)
#### list的构造与内存管理
list提供了多种constructors，主要包括四种：默认构造函数、使用n个值来初始化list、将另一个list的部分数据来初始化以及复制构造函数。   
以下是默认构造函数，就是创建一个节点，然后前后指向自己。    

	list() { empty_initialize(); }
	void empty_initialize() {
		node = get_node();
		node->next = node;
		node->prev = node;
	}
list缺省使用alloc作为空间配置器，并据此另外定义了一个list_node_allocator，方便以节点大小为配置单位。
	
	typedef simple_alloc<list_node, Alloc> list_node_allocator;
于是，list\_node\_allocator(n)表示配置n个节点空间，list提供四个函数分别用来配置、释放、构造和销毁一个节点。

	link_type get_node() { return list_node_allocator::allocate(); }
	void put_node(link_type p) { list_node_allocator::deallocate(p); }
	
	link_type create_node(const T& x) {	//配置并构造节点
		link_type p = get_node;
		construct(&p->data, x);
		return p;
	}  
	void destroy_node(link_type p){		//析构并释放节点
		destroy(&p->data);
		put_node(p);
	}
另外，当我们以push\_back()将新元素插入于list尾端时，此函数内部调用insert():
	
	void push_back(const T& x) { insert(end(), x); }
insert()是一个重载函数，有多种形式，其中最简单的一种如下，符合以上所需： 首先配置并构造一个节点，然后进行适当的指针操作，将新节点插入进去：
	
	iterator insert(iterator position, const T& x){
		link_type tmp = create_node(x);
		tmp->next = position.node;
		tmp->prev = position.node->prev;
		(link_type(position.node->prev))->next = tmp;
		position.node->prev = tmp;
		return tmp;
	}
#### list的析构

	~list() {
		clear();
		put_node(node);
	}
clear()清空链表所有数据，即只剩下node节点，然后put\_node将最后的node节点回收。

### list常用操作函数

#### 在链表头尾插入节点

	void push_front(const T& x) { insert(begin(), x); }
	void push_back(const T& x) { insert(end(), x); }

#### erase

	iterator erase(iterator position){
		link_type next_node = link_type(position.node->next);
		link_type prev_node = link_type(position.node->prev);
		prev_node->next = next_node;
		next_node->prev = prev_node;
		destroy_node(position.node);
		return iterator(next_node);
	}
修改前后节点的指针指向，然后销毁position处的节点。

#### remove
将数值为value的所有元素移除：
	
	template <class T, class Alloc>
	void list<T, Alloc>::remove(const T& value){
		iterator first = begin();
		iterator last = end();
		while(first != last){
			iterator next = first;
			++next;
			if(*first == value) erase(first);
			first = next;
		}
	}	
这里的remove是真的移除了元素，而不像vector那样。

#### transfer
list内部提供一个迁移操作transfer：将某连续范围的元素迁移到某个特定位置之前。技术上很简单，节点间的指针移动。

	void transfer(iterator position, iterator first, iterator last) {
  		if (position != last) {
    		(*(link_type((*last.node).prev))).next = position.node;	　　　　// (1)
    		(*(link_type((*first.node).prev))).next = last.node;		   // (2)
    		(*(link_type((*position.node).prev))).next = first.node;  	   // (3)
    		link_type tmp = link_type((*position.node).prev);			   // (4)
    		(*position.node).prev = (*last.node).prev;			           // (5)
    		(*last.node).prev = (*first.node).prev; 				       // (6)
    		(*first.node).prev = tmp;						               // (7)
  		}
	}
示意图如下所示。
	![](http://i.imgur.com/nq2Qm5c.png)
transfer并非公开接口。list公开提供的是接合操作splice：将某连续范围的元素从一个list移动到另一个（或同一个）list的某个定点。
	
	//将x链表的所有元素插入到当前list的position处
	void splice(iterator position, list& x) {
		if(!x.empty()){
			transfer(position, x.begin(), x.end());
		}
	}
	//将i处节点插到position之前，i和position可能来自同一个链表
	void splice(iterator position, list&, iterator i) {
		iterator j = i;
		++j;
		if(position == i || position == j) return;
		transfer(position, i, j);
	}
	//将[first, last)内的所有元素结合与position所指位置之前,position不能位于[first,last）之内
	void splice(iterator position, list& iterator first, iterator last) {
		if(first!=last) {
			transfer(position, first, last);
		}
	}

#### merge(), reverse(), sort()
基于transfer操作，list提供了三个很有用的函数：     
merge()：合并两个链表，要求两个链表必须有序，合并之后的链表也是有序的。merge之后，传入的list被清空。    
reverse()：将链表数据反转。      
sort()：由于list的迭代器为Bidirectional iterator，而STL的sort算法必须接受RamdonAccessIterator，所以list不能使用STL的sort算法。于是，基于quick sort，list实现了自身的sort。     
三个函数的具体源码这里就不再详细列出。
### 总结
list是一种很常用的数据结构，我们不仅要熟练使用它，还要弄清其中的设计与实现。