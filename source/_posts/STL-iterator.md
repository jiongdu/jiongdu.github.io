---
title: C++ STL迭代器
date: 2016-10-14 17:49:12
tags: C++
---
我们知道，STL的中心思想在于：将数据容器和算法分开，彼此独立设计，最后再以一帖胶着剂将他们撮合在一起。容器和算法的泛型化，可分别通过C++的class templates和function templates达成目标，而如何设计出两者之间的良好胶着剂，是一个值得深究的问题。本文所要学习的迭代器（iterator）就是在STL中扮演粘胶的角色。
<!--more-->
### Traits编程手法
在迭代器的实现中，经常需要访问迭代器所指对象的类型，称之为该迭代器的value type。利用内嵌类型声明typedef可以轻松实现隐藏所指对象类型。如下迭代器的实现：

	template <class T>
	struct Iterator{
		typedef T value_type;
		...
	};

泛型算法就可以通过typename Iterator::value_type来获得value type。
	
	template <class Iterator>
	typename Iterator::value_type
	getValue(Iterator iter){
		return *iter;
	}
这里，关键字typename必不可少，因为T是一个一个template参数，在它被实例化之前，编译器不知道T是一个类型还是一个其他的对象，typename用于告诉编译器这是一个类型，这样才能通过编译。    
如此，可以解决“函数的template参数推导机制推导的只是参数，无法推导函数的返回值型别“的问题，但是有个问题，并不是所有迭代器都是class type，比如原生指针。如果不是class type，就无法为它定义内嵌型别。但STL（以及整个泛型思维）都必须接受原生指针作为一种迭代器，所以上面这样还不够。     
所以，这里需要多一层的封装，即萃取编译技术。

#### template partial specialization
template partial specialization的大致意思是：如果class template拥有一个以上的template参数，我们可以针对其中某个（或数个，但非全部）template参数进行特化工作。换句话说，我们可以在泛化设计中提供一个特化版本。所以，所谓的partial specialization就是“针对（任何）template参数更进一步的条件限制所设计出来的一个特化版本”。      
比如，面对以下的class template:
	
	template<typename T>
	class C { ... };

然后，有一个如下形式的partial specialization:

	template<typename T>
	class C<T*> { ... };
这个特化版本仅适用于“T为原生指针”的情况，即“T为原生指针”便是“T为任何型别”的一个更进一步的条件限制。      
如此，我们便可以使用partial specialization解决“内嵌型别”未能解决的问题。即对“迭代器之template参数为指针”者，设计特定版的迭代器。      
下面这个class template专门用来“萃取”迭代器的特性，而value_type正是迭代器的特性之一：
	
	template <class Iterator>
	struct iterator_traits {
		typedef typename Iterator::value_type value_type;
	};
这里所谓的traits，其意义是，如果Iterator定义有自己的value\_type，那么通过这个traits的作用，萃取出来的value\_type就是Iterator::value\_type。
因此，先前的func可以改写成这样：

	template <class T>
	typename iterator_traits<Iterator>::value_type
	getValue(Iterator iter){
		return *iter;
	}
如此，多了一层间接性，traits便可以拥有特化版本。令iterator\_traits拥有一个partial specialization：

	template <class T>
	struct iterator_traits<T*>{	  //模板是一个原生指针
		typedef T value_type;
	}
于是，原生指针虽然不是一种class type，亦可通过traits取其value type。
### STL的五种迭代器
根据移动特性与施行操作，迭代器被分为五类：    
（1）Input Iterator:这种迭代器所指的对象，不允许外界改变，客户只可读取它们所指的东西，且只能向前移动，一次一步。C++标准库中的istream\_iterator是这一类的代表。    
（2）Output Iterator：与Input Iterator类似，但一切只为输出，C++标准库中的ostream\_iterator是这类的代表。    
（3）Forward Iterator：这种迭代器可以做前述两种分类所做的每一件事，而且可以读或写所指物一次以上。   
（4）Bidirectional Iterator：比上一个分类威力更大，它除了可以向前移动，还可以向后移动。STL的list迭代器就属于这一分类，set、multiset、map和multimap的迭代器也都是这一分类。     
（5）最具威力的当属Random Access Iterator。这种迭代器比上一个分类威力更大的地方在于它可以执行“迭代器算术”，也就是可以在常量时间内向前或向后跳跃任意距离，这样的算术很类似指针算术，因为random access迭代器正是以内置（原始）指针为榜样，而内置指针也可被当做random access迭代器使用。 vector, deque和string提供的迭代器都是这一分类。       
这些迭代器的分类与从属关系，如下图所示。    
![](http://i.imgur.com/WSUJIqE.jpg)     
设计算法时，如果可能，我们尽量针对上图中的某种迭代器提供一个明确定义，并针对更强化的某种迭代器提供另一种定义，这样才能在不同情况下提供最大效率。假设有个算法可接受Forward Iterator，而传入Random Access Iterator，它当然也会接受，因为一个Random Access Iterator必然是一个Forward Iterator，但是并不是最佳。       
以advance()为例说明使用函数重载机制来选择最佳的迭代器版本。advance()函数有两个参数，迭代器p和数值n，函数内部将p累进n次。下面有分别针对Input Iterator、Bidirectional Iterator和Random Access Iterator的三种函数定义版本。
	
	template <class InputIterator, class Distance>
	void advance_II(InputIterator& i, Distance n){
		while(n--) ++i;		
	}
	
	template <class InputIterator, class Distance>
	void advance_BI(InputIterator& i, Distance n){
		if(n>=0)
			while(n--) ++i;
		else
			while(n++) ++i;
	}	
	
	template <class InputIterator, class Distance>
	void advance_RAI(InputIterator& i, Distance n){
		i+=n;		
	}
现在，当程序调用advance()时，应该选择跟自己最匹配的版本，所以，我们需要将三者合一。首先想到的是运用条件判断。

	template <class InputIterator, class Distance>
	void advance(InputIterator& i, Distance n){
		if(is_random_access_iterator(i)){
			advance_RAI(i, n);
		}else if(is_bidirectional_iterator(i)){
			advance_BI(i, n);
		}else{
			advance_II(i, n);
		}
	}	
但是像这样在执行时期才决定使用哪一个版本，会影响程序效率。最好能在编译期就选择正确的版本，所以，采用函数重载机制。     
前面三个advance\_xx()都有两个函数参数，型别都未定（都是template参数），为了令其同名，形成重载函数，我们必须加上一个型别已确定的函数参数，使函数重载机制得以有效运作起来。       
STL设计思想为：如果traits有能力萃取出迭代器的种类，我们便可利用这个“迭代器类型”相应型别作为advanced()的三个参数，这个相应型别一定必须是一个class type，因为编译器需要依赖它（一个型别）来进行重载决议。       
定义五个class，代表五种迭代器类型：    

	struct input_iterator_tag { };
	struct output_iterator_tag { };
	struct forward_iterator_tag : public input_iterator_tag｛｝；
	...
现在重新设计__advance，并加上第三参数，使它们形成重载:   
	
	template <class InputIterator, class Distance>
	inline void __advance(InputIterator& i, Distance n,
						  input_iterator_tag){
		while(n--) ++i;
	}
	
	template <class BidirectionalIterator, class Distance n>
	inline void __advance(BidirectionalIterator& i, Distance n,
						  bidirectional_iterator_tag){
		if(n>=0)
			while(n--) ++i;
		else
			while(n++) --i;
	}
	...
每个\_\_advance()的最后一个参数都只声明型别，并未指定参数名称，因为它纯粹只是用来激活重载机制，函数之中根本不使用该参数。     
最后，还需要一个对外开放的上层接口，调用上述各个重载的\_\_advance()，这一上层接口只需要两个参数，当它准备将工作转给上述的\_\_advance()时，才自行加上第三参数：迭代器类型。因此，这个上层函数必须有能力从它所能获得的迭代器中推导出其类型（traits机制）。

	template <class InputIterator, class Distance>
	inline void advance(InputIterator& i, Distance n){
		__advance(i, n, 
					iterator_traits<InputIterator>::iterator_category());
	}
iterator_traits<InputIterator>::iterator_category())将产生一个暂时对象，其型别应该隶属于前述四个迭代器类型，然后根据这个型别，编译器才决定调用哪一个\_\_advance()重载函数。    
因此，为了满足上述行为，traits必须再增加一个相应的型别：

	template <class Iterator>
	struct iterator_traits{
		...
		typedef typename Iterator::iterator_category iterator_category;
	};

	//针对原生指针
	template <class T>
	struct iterator_traits<T*>{
		...
		typedef typename random_access_iterator_tag iterator_category;
	}
最后一个问题，STL算法的命名规则：以算法所能接收之最低阶迭代器类型，来为其迭代器型别参数命名。

     

 
