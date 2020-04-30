---
title: C++ STL之deque
date: 2016-10-31 17:49:42
tags: C++
---
C++标准模板库序列式容器中，使用最少的估计就是deque了。但是，了解下deque这种双向开口的连续线性空间容器的底层设计与实现仍是很有帮助的。此外，stack和queue这两种适配器容器（容器底层是调用其他容器）底层实现默认就是使用deque。
<!--more-->
### 与vector比较
deque和vector的最大差异，一是deque允许常熟时间内对起头端进行元素的插入或移除操作，二在于deque没有所谓容量观念，因为它是动态地以分段连续空间组合而成，随时可以增加一段新的空间并链接起来。即，像vector那样“因旧空间不足而重新配置一块更大空间，然后复制元素，再释放旧空间”这样的事情在deque是不会发生的。也因此，deque没有必要提供所谓的空间保留功能。        
更具体的，array无法增长，vector虽可增长，却只能向尾端增长，而且所谓增长是个假象，事实上是（1）另觅更大空间；（2）将原数据复制过去；（3）释放原空间 三部曲。   
deque是由一段一段的定量连续空间构成。一旦有必要在deque的前端或尾端增加新空间，便配置一段定量连续空间，串接在整个deque的头端或尾端。deque的最大任务，便是在这些分段的定量连续空间上，维护其整体连续的假象，并提供随机存取的接口。这样避开了“重新配置、复制、释放”的轮回，代价则是复杂的迭代器架构。    


### deque的中控器和迭代器
deque底层实现一个中控器，即一个指针数组，其中的每一个元素都是指针，指向另一段连续线性空间，称为缓冲区。缓冲区才是deque的存储空间主体。       
迭代器由以下四个部分组成：    
（1）cur：指向缓冲区当前元素。           
（2）first：指向缓冲区的第一个位置。        
（3）last：指向末节点，即最后一个元素的下一个位置。      
（4）node：指向主控器相应的索引位置。   
中控器、缓冲区、迭代器之间的关系如下图所示。    
![](http://i.imgur.com/SsE0h85.png)
以下是中控器和迭代器的相关源代码。     

	template <class T, class Alloc = alloc, size_t BufSiz = 0>
	class deque {
	public:
		typedef T value_type;
		typedef value_type* pointer;		
		...
	protected:
		typedef pointer* map_pointer;		//指针的指针
		map_pointer map;		//map是指针数组，中控器
		size_type map_size;		//map内可容纳多少指针
	};
可以看出，中控器map其实是一个T**，即是指向指针的指针，T为元素型别。      

	template <class T, class Ref, class Ptr, size_t BufSiz>
	struct __deque_iterator {
		typedef __deque_iterator<T, T&, T*, BufSiz> iterator;
		typedef __deque_iterator<T, const T&, const T*, BufSiz> const_iterator;
		static size_t buffer_size() { return __deque_buf_size(BufSiz, sizeof(T)); }
		typedef random_access_iterator_tag iterator_category;	//RandomAccess Iterator
		typedef T value_type; 
		typedef Ptr pointer;
		typedef Ref reference;
		typedef size_t size_type;
		typedef ptrdiff_t difference_type;
		typedef T** map_pointer;				//指向主控器的指针

		T* cur;
		T* first;
		T* last;
		map_pointer node;
	};
迭代器的四个主要组成：cur、first、last、node的定义。    
接下来是构造函数和迭代器的重载运算因子。          
	
	__deque_iterator(T* x, map_pointer y)	//用T类型数据和指向中控器的指针初始化一个迭代器
		: cur(x), first(*y), last(*y+buffer_size()), node(y) {}
	__deque_iterator() : cur(0), first(0), last(0), node(0) {}	//创建一个空的迭代器
	__deque_iterator(const iterator& x) : cur(x.cur), first(x.first), last(x.last), node(x.node) {}	//复制构造函数
	
	reference operator*() const { return *cur; }
	pointer operator->() const { return &(operator*()); }
	difference_type operator-(const self& x) const {
		return difference_type(buffer_size())*(node-x.node-1)+(cur-first)+(x.last-x.cur);
	}
	...
	self& operator+=(difference_type n){
		difference_type offset=n+(cur-first);
		if(offset>=0 && offset<difference_type(buffer_size())){	//目标在同一个缓冲区内
			cur+=n;
		}else{			//目标位置不在同一个缓冲区内
			difference_type node_offset = offset>0 ? offset/difference_type(buffer_size()) : -difference_type((-offset-1) / buffer_size())-1;
			set_node(node + node_offset);
			cur = first+(offset-node_offset*difference_type(buffer_size()));
		}
		return *this;
	}
	
在进行+=n运算时，要判断+n之后的索引是否还在当前缓冲区内，如果在，cur简单+n即可，如果不在，则要计算正确的缓存节点和正确的元素位置。其他+n，-=n，-运算都可以通过+=运算得出。

### deque的结构
deque除了维护一个指向map的指针外，还维护start、finish两个迭代器，分别指向第一缓冲区的第一个元素和最后缓冲区的最后一个元素（的下一个位置）。    
	
	template <class T, class Alloc = alloc, size_t BufSiz = 0>
	class deque {
	public:
		typedef T value_type;
		typedef value_type* pointer;
		typedef size_t size_type;
		typedef __deque_iterator<T, T&, T*, BufSiz> iterator;
		typedef simple_alloc<value_type, Alloc> data_allocator;
		typedef simple_alloc<pointer, Alloc> map_allocator;
		...
	protected:
		typedef pointer* map_pointer;
		iterator start;
		iterator finish;
		map_pointer map;
		size_type map_size;		//map容纳指针的个数	
		...
	};

#### 默认构造函数

	deque()
		: start(), finish(), map(0), map_size(0) {
		create_map_nodes(0);
	} 
默认构造函数先初始化两个迭代器为空迭代器，然后设置map和map\_size，接着调用create\_map\_and\_nodes(size_type num\_elements)来初始化一块内存。

	template<class T, class Alloc, size_t BufSize>
	void deque<T, Alloc, BufSize>::create_map_and_nodes(size_type num_elements) {
		size_type num_nodes = num_elements/buffer_size() + 1;	//	需要节点数
		map_size = max(initial_map_size(), num_nodes+2);	//一个map管理的节点数，最少8个，最多是“所需节点加2”
		map = map_allocator::allocate(map_size);

		map_pointer nstart = map+(map_size-num_nodes)/2;
		map_pointer nfinish = nstart+num_nodes-1;
		map_pointer cur;

		__STL_TRY {
			for(cur=nstart; cur<=nfinish; ++cur){
				*cur = allocate_node();
			}
		}catch(...){
			...
		}
		//为deque内的两个迭代器设定正确的内容
		start.set_node(nstart);
		finish.set_node(nfinish);
		start.cur = start.first;
		finish.cur = finish.first + num_elements%buffer_size();
	}

#### 由n个value初始化的构造函数

	deque(size_type n, const value_type& value)
		: start(), finish(), map(0), map_size(0) {
		fill_initialize(n, value);
	}
该构造函数调用fill\_initialize函数来初始化deque，fill\_initialize函数负责产生并安排好deque的结构，并将元素的初值设定妥当。     

	template <class T, class Alloc, size_t BufSize>
	void deque<T, Alloc, BufSize>::fill_initialize(
		size_type n, const value_type& value) {
		create_map_and_nodes(n);
		map_pointer cur;
		__STL_TRY {
			//为每个节点的缓冲区设定初值
			for(cur=start.node; cur<finish.node;++cur){
				uninitialized_fill(*cur, *cur + buffer_size(), value);
			}
			uninitialized_fill(finish.first, finish.cur, value);	//最后一个节点单独对待
		}
	}

#### 两个迭代器初始化的构造函数
	
	template <class InputIterator>
	deque(InputIterator first, InputIterator last)
		: start(), finish(), map(0), map_size(0) {
		range_initialize(first, last, iterator_category(first));
	}
该构造函数调用range\_initialize来用迭代器范围内的元素初始化deque.range\_initialize，根据迭代器类型，调用对应的版本。

#### 析构函数
deque的析构函数先是调用destroy析构缓冲区内的数据，然后调用destroy\_map\_and\_nodes函数析构缓冲区和中控器内存。    

	~deque() {
		destroy(start, finish);
		destroy_map_and_nodes();
	}

### deque的一些常用操作

deque所提供的元素操作很多，如pop\_back、pop\_front、clear、erase、insert等，这里不再一一列举，在理解时，只需把握“deque是多个连续缓冲区组合在一起的分段线性空间”即可。