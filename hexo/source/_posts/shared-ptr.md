---
title: shared_ptr的注意事项
date: 2016-08-08 21:38:44
tags: C++
---
前面在[浅谈C++值语义和对象语义](http://blog.dujiong.net/2016/05/15/cplusplus-sematics/)一文中简要的阐述了一下智能指针shared\_ptr的作用和用法，作为C++11中最为广泛使用的智能指针，shared\_ptr虽然解决了裸指针的诸多问题，但是它并非完美、无懈可击，所以，本文就shared\_ptr使用过程中需要注意的一些问题做一下总结。
<!--more-->  
### 循环引用

首当其冲的便是shared_ptr造成的循环引用。     

	#include <iostream>
	#include <memory>

	using namespace std;
	
	class BaseClass;
	class DerivedClass;

	typedef std::shared_ptr<BaseClass> BaseClassPtr;
	typedef std::shared_ptr<DerivedClass> DerivedClassPtr;
	
	class BaseClass
	{
		public:
			DerivedClassPtr derivedPtr;
			~BaseClass()
			{
				cout << "~BaseClass()" << endl;
			}
	};

	class DerivedClass
	{
		public:
			BaseClassPtr basePtr;
			~DerivedClass()
			{
				cout << "~DerivedClass()" << endl;
			}
	};

	int main()
	{
		BaseClassPtr base(new BaseClass());
		DerivedClassPtr derived(new DerivedClass());

		base->derivedPtr = derived;
		derived->basePtr = base;

		cout << base.use_count() << endl;
		cout << derived.use_count() << endl;
	}

程序运行结果如下。    
![](http://i.imgur.com/dqHRq8F.png)     
从结果可以看出，堆上的BaseClass和DerivedClass都没有被析构，因为二者的引用都为1，并形成了循环引用，类似于“放开我的引用”，“你先放开我的我就放你的”的恶性循环，所以，造成了内存泄露。   
解决方法是使用弱智能指针weak\_ptr，让BaseClass和DerivedClass分别持有对方的weak\_ptr，而不是shared\_ptr，weak\_ptr也是一个引用计数型智能指针，但是它不增加对象的引用计数。这样，就可以保证内存正常释放。     
![](http://i.imgur.com/cG2fmZx.png)    

### 线程安全


shared\_ptr是引用计数型智能指针，几乎所有的实现都采用在堆上存放计数值的方法。具体来说，shared\_ptr<Foo>包含两个成员，一个是指向Foo的指针ptr，另一个是ref\_count指针（不一定是原始指针），指向堆上的ref\_count对象。ref\_count对象有多个成员。
   
![](http://i.imgur.com/WZNNAPg.png)

所以，如果执行：
	
	shared_ptr<Foo> x(new Foo);
	shared_ptr<Foo> y = x;

y=x将会涉及两个成员的拷贝,ptr和ref\_count指针。这两步不是原子的，所以在多线程中会有race condition，需要加锁保护。     

### 内存多次释放

shared_ptr多次引用同一数据会导致内存多次释放。
	
	#include <iostream>
	#include <memory>

	using namespace std;
	
	int main()
	{
		int* ptr = new int(100);
		std::shared_ptr sptr1(ptr);
		...
		std::shared_ptr sptr2(ptr);

		return 0;
	}
只分配了一次内存，却两次释放。

### enable\_shared\_from\_this

当使用智能指针时，类的某些成员函数需要返回的是管理自身的shared\_ptr，这时就需要使A继承自enable\_shared\_from\_this（顾名思义，将this指针变身为shared\_ptr），然后通过其成员函数shared\_from\_this()返回指向自身的shared\_ptr。      

首先看没有使用它会发生什么？
	
	#include <iostream>
	#include <memory>

	using namespace std;
	
	class A
	{
		public:
			~A()
			{
				cout << "~A()" << endl;
			}
			shared_ptr<A> get_sptr()
			{
				return shared_ptr<A>(this);
			}
	};
		
	int main()
	{
		shared_ptr<A> a(new A());
		shared_ptr<A> b = a->get_sptr();
		
		return 0;
	}  
运行结果为：      
![](http://i.imgur.com/21rnuTE.png)          
可以看出，只new了一个A对象，但是却调用了两次析构函数。产生这个错误的原因get\_sptr()成员函数中将shared\_ptr中的引用计数器的值加了1。所以，需要解决的是，怎样通过一个类的成员函数获取当前对象的shared\_ptr？     
方法就是使类继承enable\_shared\_from\_this，然后在需要shared_ptr的地方调用其成员函数shared\_from\_this()即可。

	#include <iostream>
	#include <memory>

	using namespace std;
	
	class A : public enable_shared_from_this<A>
	{
		public:
			~A()
			{
				cout << "~A()" << endl;
			}
			shared_ptr<A> get_sptr()
			{
				return shared_from_this();
			}
	};
		
	int main()
	{
		shared_ptr<A> a(new A());
		shared_ptr<A> b = a->get_sptr();
		
		return 0;
	}  
运行结果为：     
![](http://i.imgur.com/ZuA23y9.png)    
需要注意的是，不能在构造函数中使用`shared_from_this()`，因为虽然对象的基类`enable_shared_from_this`的构造函数已经调用，但是`shared_ptr`的构造函数并没有调用，所以这个时候调用`shared_from_this()`是错的。

### 总结

智能指针虽好，但也并非完美，在使用的过程中还是需要注意。