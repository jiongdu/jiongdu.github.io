---
title: C++ STL之vector
date: 2016-10-25 18:42:24
tags: C++
---
“用C++开发，尽量用容器类+迭代器来代替数组+指针，因为数组+指针容易越界，或者内存泄露，而容器+迭代器已将底层封装好，使用安全简单”。       
<!--more-->
vector是序列式容器里使用最广泛的容器之一，该容器被用来改善数组的缺点，vector是一个动态空间，随着元素的加入，它的内部机制会自行扩充以容纳新元素，因此，vector的运用对于内存的合理利用和灵活性都有很大的帮助，再也不必因为害怕空间不足一开始就配置一个大容量数组了。        
所以，vector的实现技术，关键在于其对大小的控制以及重新配置时的数据移动效率。下面就一起详细地看看vetcor的设计。     
### vector成员变量
vector的成员变量比较简单，主要由空间配置器和三个迭代器组成，三个迭代器分别指向目前使用空间的头、尾合目前可用空间的尾。其源码如下：
	
    template <class T, class Alloc = alloc>   
	class vector{
	public:
		typedef T value_type;
		typedef value_type* pointer;
		typedef value_type* iterator;	//vector的迭代器是普通指针
		...
	protected:
		typedef simple_alloc<value_type, Alloc> data_allocator;
		
		iterator start;
		iterator finish;
		iterator end_of_storage;
	...
	};
vector提供以下函数用于获取相关成员变量。
	
	iterator begin() { return start; }
	iterator end() { return finish; }
	
	//返回vector当前对象的个数
	size_type size() const { return size_type(end()-begin()); }
	//返回目前可用空间的大小
	size_type capacity() const { return size_type(end_of_storage-end()); }

### vector的构造函数

	vector() : start(0), finish(0), end_of_storage(0) { }
	vector(size_type n, const T& value) { fill_initialize(n, value); }
	vector(int n, const T& value) { fill_initialize(n, value); }
	vector(long n, const T& value) { fill_initialize(n, value); }
	explicit vector(size_type n) { fill_initialize(n, T()); }
除了默认构造函数，四个带参构造函数都调用`fill_initialize(size_type n, const T& value)`，这个函数调用`iterator allocate_and_fill(size_type n, const T& x)`配置空间，并填充数据(`uninitialized_fill_n(size_type n, const T& x)`)，并设置vector的三个迭代器。

	void fill_initialize(size_type n, const T& value){
		start = allocate+_and_fill(n, value);
		finish =  start + n;
		end_of_storage = finish;
	}

	iterator allocate_and_fill(size_type n, const T& x){
		iterator result = data_allocator::allocate(n);	//分配n个sizeof(T)大小的内存
		uninitialized_fill_n(result, n, x);		//全局函数，填充数据
		return result;
	}
uninitialized\_fill\_n是STL的内存基本处理函数，用于在申请的内存上填充数据，存放在stl\_uninitialized.h中。该函数的逻辑是，首先萃取出迭代器result的value type，然后判断该型别是否为POD型别，POD指Plain Old Data，也就是标量型别或传统的C struct型别。POD型别必然拥有trival constructor/default constructor/copy/assignment函数，因此，可以对POD型别采用最有效率的初值填写手法，而对non-POD型别采取最保险的做法。
相关代码如下：     

	template <class ForwardIterator, class Size, class T>
	inline ForwardIterator uninitialized_fill_n(ForwardIterator first, 
													Size n, const T& x) {
  		return __uninitialized_fill_n(first, n, x, value_type(first));
	}

	template <class ForwardIterator, class Size, class T, class T1>
	inline ForwardIterator __uninitialized_fill_n(ForwardIterator first, 
													Size n, const T& x, T1*) {
	    typedef typename __type_traits<T1>::is_POD_type is_POD;
  		return __uninitialized_fill_n_aux(first, n, x, is_POD());                                
	}
	
	template <class ForwardIterator, class Size, class T>	
	inline ForwardIterator __uninitialized_fill_n_aux(ForwardIterator first, Size n, 
													const T& x, __true_type) {
  		return fill_n(first, n, x);			//POD类型，直接用fill_n填充即可
	}
		
	template <class ForwardIterator, class Size, class T>
	ForwardIterator __uninitialized_fill_n_aux(ForwardIterator first, Size n,
                           							const T& x, __false_type) {
  		ForwardIterator cur = first;
  		__STL_TRY {
    		for ( ; n > 0; --n, ++cur)
      			construct(&*cur, x);	//调用构造函数，这里用的是placement new来构造
    		return cur;
  		}
  		__STL_UNWIND(destroy(first, cur));
	}
	     
同样的，在复制构造函数中，也是先分配内存，然后根据是否为POD型别调用不同的复制函数。    
### vector的析构函数	
vector的析构函数比较简单，先析构[start, finish)，然后归还内存。

	~vector() {
		destory(start, finish);
		deallocate();
	}
同构造函数类似，destroy函数根据数据类型，如果是POD，则什么都不做，如果不是POD型别，则调用析构函数一个一个的析构。然后调用deallocate函数归还内存,如果大于128bytes，则归还给系统，小于128bytes，则还给内存池，后面还可使用。

	void deallocate() {
		if(start) data_allocator::deallocate(start, end_of_storage-start);
	}
### vector常用操作函数
#### push_back	
当我们以push_back()将新元素插入于vector尾端时，该函数首先检查是否还有备用空间，如果有，就直接在备用空间上构造函数，并调整迭代器finish，使vetcor变大，如果没有，就扩充空间（重新配置、移动数据、释放原空间）。   

	void push_back(const T& x){
		if(finish != end_of_storage){
			construct(finish, x);
			++finish;		
		}else{
			insert_aux(end(), x);			//已无备用空间
		}
	}

	template <class T, class Alloc>
	void vector<T, Alloc>::insert_aux(iterator position, const T&x){
		if(finish != end_of_storage){
			construct(finish, *(finish-1));	
			++finish;
			T x_copy = x;
			copy_backward(position, finish-2, finish-1);
			*position = x_copy;
		}else{
			const size_type old_size = size();
			const size_type len = old_size!=0 ? 2 * old_size : 1;
			iterator new_start = data_Allocator::allocate(len);
			iterator new_finish = new_start;
			
			try{
				new_finish = uninitialized_copy(start, position, new_start);
				construct(new_finish, x);
				++new_finish;
				new_finish = uninitialized_copy(position, finish, new_finish);
			}catch(...){
				destroy(new_start, new_finish);
				data_allocator::deallocate(new_start, len);
				throw;
			}
			
			destroy(begin(), end());		//析构
			deallocate();					//释放原来的内存
			//调整三个迭代器
			start = new_start;
			finish = new_finish;
			end_of_storage = new_start + len;
		}
	}
#### erase

	iterator erase(iterator first, iterator last){
		iterator i = copy(last, finish, first);
		destroy(i, finish);
		finish = finish-(last-first);
		return first;
	}
	
	iterator erase(iterator position){
		if(position+1 != end()){
			copy(position+1, finish, position);
		}
		--finish;
		destroy(finish);
		return position;
	}
所以，调用erase()函数时，size()会变化，而capacity()不会发生变化。可以看出，erase()函数会涉及到元素的移动，可能会导致原有的迭代器失效，使用时需要注意。此外，remove()和erase()的区别：remove只是将待删除元素移动到vector的尾端，不是删除，erase才是真正的删除元素。一定不要误用。
  
#### 其他
另外，还有一些常用的函数操作，如pop\_back(), clear(), insert()等，这里就不再逐一介绍，方法和原理都和上述差不多，在日常编程中多使用，自然就熟了。