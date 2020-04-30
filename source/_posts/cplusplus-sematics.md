---
title: 浅谈C++值语义和对象语义   
date: 2016-05-15 19:29:20
tags: C++
---


### 值语义和对象语义
值语义(value sematics)指的是对象的拷贝与原对象无关，拷贝之后就与原对象脱离关系。C++的内置类型都是值语义，如bool/int/double等，标准库里的pair<>,vector<>,map<>，string等类型也都是值语义。Java语言的primitive types(原生数据类型)也是值语义。             
与值语义(object sematics)对应的是"对象语义"，或者也叫做"引用语义"。对象语义指的是面向对象意义下的对象，对象是禁止拷贝的或拷贝后与原来的对象存在关联。比如，拷贝一个Employee对象是没有意义的，一个雇员不会变成两个雇员。同样，muduo库中的TcpConnection，显然也是不可复制的，因为牵涉到系统的资源。Java里面的class对象都是对象语义/引用语义。
<!--more-->      
下面举Java和C++几个简单的例子看值语义和对象语义:

#### Java
	
	int a = 8;
	int b = a;

这里的a，b都是int， 属于Java中原生数据类型，他们的特点是复制后二者再无关联。它们表现出来是两个value。    
与他们形成对比的是用户自定义的class，如下:
	
	Student a1 = new Student("dujiong", 23);
	Student a2 = a1;
	a2.setAge(25);	
	System.out.println(a1.getAge());

代码输出为25，因为第二行语句，导致a和b指向的是同一个对象，修改a2也会影响a1，所以表现的行为是一种reference。
#### C++
如前所述，C++的内置类型、标准库容器如vector<>，map<>等都是值语义，C++要求凡是可以放入容器的元素都必须具备值语义，即必须具备拷贝的能力，放入标准容器的元素和之前的元素没有任何的关联。     
而C++的普通类呢?
	
	class Book
	{
		private:
			string isbn_;
			double price_;
			int amount_;
	};

如果以下面的代码:
	
	Book a1;
	Book a2 = a1;

那么a1和a2复制后将没有任何关联，他们的表现属于值语义。

### 值语义与C++语言
C++的class本质上是值语义的。C++的设计初衷是让用户定义的类型(class)能像内置类型一样工作，具有同等的地位。为此，C++做了妥协:    
(1) class的layout与C struct一样，没有额外的开销，定义一个"只包含一个int成员的class"的对象开销和一个int一样。   
(2) class可以在stack上创建，也可以在heap上创建。。    
(3) class的data member默认都是uninitialized。    
(4) class的数组是一个个class对象挨着，没有额外的indirection。    
(5) 编译器为class默认生成copy constructor和assignment operator。其他语言没有copy constructor，也不允许重载assignment operator。C++的对象默认是可以拷贝的。    
(6) 当class类型传入函数时，默认是make a copy(除非参数声明为reference)。    
(7) 当函数返回一个class type时，只能通过make a copy(C++不得不定义RVO来解决性能问题)。    
(8) 当class type为成员时，数据成员时嵌入的。
C++这样的设计带来了性能上的好处。比如在C++里定义复数类(complex<double>class)和其数组(array)、向量(vector)，它们的layout分别是:

   ![](http://i.imgur.com/xN0fihh.png)

而如果在Java中定义同样的结构，就不一样。

![](http://i.imgur.com/ptaBium.png)

Java中每个object都有header。对比Java，C++的对象模型更紧凑。


### 值语义与生命期
值语义的一个巨大好处是生命期管理很简单，就像你不需要操心基本数据类型的生命期一样。值语义的对象要么是栈上对象，要么作为其他对象的成员(一个成员函数使用自己的数据成员对象)。而，对象语义由于不能拷贝，只能通过指针或引用来使用它。而一旦使用指针或引用来操作对象，那么就要担心所指的对象是否已被释放，而这也是C++ bug的主要来源。此外，C++只能通过指针或引用来获得多态，所以，在C++中进行继承和多态的面向对象编程中，资源管理成为了一大难题。            
#### 模型1
模型1: a Parent has a Child, a Child knows his/her Parent    
Java中很好些，不用担心内存泄露，也不用担心空悬指针。    
	
	public class Parent{
		private Child myChild;	
	}
	public class Child{
		private Parent myParent;	
	}
只要正确初始化myChild和myParent即可。
而在C++里就要为资源管理费一番脑筋:Parent 和 Child 都代表的是真人，肯定是不能拷贝的，因此具有对象语义。Parent 是直接持有 Child 吗？抑或 Parent 和 Child 通过指针互指？Child 的生命期由 Parent 控制吗？       
直接但是易错的写法:
	
	class Child;
	class Parent : boost::noncopyable
	{
		private:
			Child* myChild;
	};
	
	class Child : boodt::noncopyable
	{
		private:
			Parent* myParent;
	};

那么，直接使用指针作为成员，那么如何确保指针的有效性？如何防止出现空悬指针？Child 和 Parent 由谁负责释放？在释放某个 Parent 对象的时候，如何确保程序中没有指向它的指针？在释放某个 Child 对象的时候，如何确保程序中没有指向它的指针？     
这一系列问题都是C++面向对象编程令人头疼的问题。现在可以使用boost智能指针解决(C++11引入,用栈上对象管理堆上对象的生命期），借助于智能指针把对象语义转换为值语义，从而解决对象生命期问题：让Parent持有Child的智能指针，同时让Child持有Parent的智能指针，这样始终引用对方的时候就不用担心出现空悬指针。当然，其中一个智能应该是weak reference，否则会出现循环引用，导致内存泄露。
所以，如果Parent拥有Child，Child的生命期由其Parent控制，Child的生命期小于Parent，那么代码：     
	
	class Parent;
	class Child : boost::noncopyable
	{
		public:
			explicit Child(Parent* myParent) : myParent_(myParent)
			{ }
		private:
			Parent* myParent_;
	};
	class Parent : boost::noncopyable
	{
		public:
			Parent() : myChild(new Child(this))
			{ }
		private:
			boost::scoped_ptr<Child> myChild;
	}; 

#### 模型2
模型2: a Child has parents:mom and dad; a Parent has one or more Child; a Parent knows his/her spouser    
Java描述还是很简单，垃圾收集会搞定对象生命期。
	
	public class Parent{
		private Parent mySpouser;
		private ArrayList<Child> myChildren;	
	}
	public class Child{
		private Parent myMom;
		private Parent myDad;
	}

而如果用C++实现，还是需要借助于智能指针将裸指针转化为值语义。
	
	class Parent;
	typedef boost::shared_ptr<Parent> ParentPtr;

	class Child : boost::noncopyable
	{
		public:
			explicit Child(const ParentPtr& myMom, const ParentPtr& myDad)
			: myMom_(myMom), myDad_(myDad)
			{ }
		private:
			boost::weak_ptr<Parent> myMom_;
			boost::weak_ptr<Parent> myDad_;
	};
		
	typedef boost::shared_ptr<Child> ChildPtr;
	
	class Parent :　boost::noncopyable
	{
		public:
			Parent(){}
			void setSpouser(const ParentPtr& spouser)
			{
				mySpouser_ = spouser;
			}
			void addChild(const ChildPtr& child)
			{
				myChildren_.push_back(child);	
			}
		private:
			boost::weak_ptr<Parent> mySpouser_;
			std::vector<ChildPtr> myChildren_;
	}

可以看出，如果不使用智能指针，使用C++面向对象编程，有效的资源管理将会很困难。

### 待续
对于智能指针，这里只是有一些简单的介绍，后面还将深入研究。

参考: http://www.cnblogs.com/Solstice/archive/2011/08/16/2141515.html