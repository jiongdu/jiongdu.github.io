---
title: C++构造函数一些常见问题分析     
date: 2016-06-05 16:58:57
tags: C++
---
近期，在阅读陈皓老师博客的时候，看到几年前的一篇[关于C++构造函数的FAQ](http://coolshell.cn/articles/804.html)，想来自己学习C++也有段时间了，就尝试着回答一下，并乘机总结一下C++构造函数的一些常见问题，记录于此。    
借此机会逛了逛C++ FAQ，以前都不了解，以后得多去学习学习，检验下自己的C++基础知识扎不扎实。
<!--more-->  

### 构造函数是用来干什么的？
顾名思义，构造函数是用来从无到有的构建对象。它将一些（连续的）最小粒度的内存单元变成对象。它要初始化对象内部所定义的域，还可以分配资源(内存、文件、套接字等)。

### List x;和List x();有什么不同？
List x;是声明了一个类List的对象x，而List x()是声明了一个函数x()，它返回一个List对象。二者自然是有本质的差别。 

### 一个类的构造函数可否调用另一个构造函数来初始化自己？
不能。如果调用另一个构造函数，编译器将初始化一个临时局部对象，而不是初始化this对象。

### 是否Fred类的默认的构造函数就一定是Fred::Fred()？
不，默认构造函数是可以带提供默认值的参数的。
比如这里：
	
	class Fred
	{
		public:
			Fred(int i=1);
	};

### 如果要创建一个Fred对象数组，什么样的构造函数会被调用？
Fred类的默认构造函数。      
当然，更多的情况下我们都会选择使用标准容器vector，而且vector的一大优势是可以指定Fred的构造函数，而不仅仅是默认构造函数。

### 构造函数初始化成员变量时，用"初始化列表"还是"赋值"？
无疑，初始化列表，在很多C++语法方面的书上都会讲，这里再总结一下。          
使用"初始化列表"最大的好处是可以提高性能，比如使用 Fred::Fread() :x\_(x)来初始化成员变量x\_，x\_直接由x构造--编译器不会产生对象的拷贝。        
而通过赋值的话，`Fred::Fred() { x_ = x }`，x作为一个临时对象被建立，并传递给x\_成员作为赋值操作，然后临时对象x在语句的结束被析构，所以，这样的效率是较低的。      
当然，如果x_的类型是int/char这样的内置类型的话，性能是没有区别的。总之，使用初始化列表来初始化成员变量是更佳的选择。

### 构造函数初始化列表的初始化顺序？
C++初始化成员变量的顺序是按照它们定义的顺序，而非在初始化列表的顺序。      
如果有继承关系的话，先基类，再派生类。

### 在构造函数中用this指针是否有问题？
一般情况下是不应该的，因为在构造阶段，this对象还没有完全形成。     
当然也不绝对，比如构造函数的函数体或其中调用的函数可以访问基类中声明的数据成员或类的静态数据成员，这时它们都已经完整的建立起来了。     
所以，一般情况下不这样做，如果要，也得记住，唯一准则是：这时this指针已经完整构造形成了。

### 什么是“命名的构造函数法”？
为类的用户提供一种更安全、直接的构造。       
因为构造函数的特殊性，函数名和类名相同。因此，区分类的不同的构造函数只能通过参数列表。但如果有多个构造函数，很好地区分它们并不容易。      
命名的构造函数法就是在private或protected内声明类所有的构造函数，并提供返回对象的public static函数。

	class Point
	{
		public:
			static Point rectangular(float x, float y);
			static Point polar(float radius, float angle);
		private:
			Point(float x, float y);
			float x_, y_;
	};	
	inline Point::Point(float x, float y) : x_(x), y_(y) {}
	inline Point::rectangular(float x, float y) { return Point(x, y); }
	inline Point::polar(float radius, float angle)
	{
		return Point(radius*cos(angle), radius*sin(angle));
	}
	int main()
	{
		Point p1 = Point::rectangular(5.0,5.0);
		Point p2 = Point::polar(5.0,5.0);
	}	


### 为什么不能在构造函数的初始化列表中初始化static数据？
因为static数据成员是属于类的，而不是类的对象，所以，必须显示定义类的静态数据成员。

### 为什么有静态数据成员的类得到了链接错误？
因为如上一条所述，static数据成员必须显示定义在一个编译单元中。     
通常在在.cpp源文件中定义static数据成员。

### 什么是"static initialization order fiasco"？
还是初始化顺序的问题，比如两个源文件a.cpp和b.cpp中分别有两个静态对象a和b。假定a对象的构造函数会调用b对象的某些成员函数。由于static对象构造在main()开始之前，且顺序未知，所以，很可能发生a对象的构造函数先运行，当其想调用b对象的成员函数时，发现b对象还没被构造。

### 如何处理构造函数的失败？
当然是抛异常啦。关于C++的异常机制，后面还得好好学学。

### explicit关键字的作用？
使用explicit就是告诉编译器禁止隐式转换。   
下面通过实例说明。
		
	class Foo
	{
		public:
			Foo(int x);
			operator int();
	};		
	int main()
	{
		Foo a = 42;			  	
		Foo b(42);			  
		Foo c = (Foo)(42);	  //显示转换
		int d = c;
		return 0;	
	}
以上都会编译成功，需要注意的是main()中的第一、四行，都涉及到了隐式转换。    
但是，有些时候需要禁止这样的隐式转换。就得使用explicit关键字。
	
	class Foo
	{
		public:
			explicit Foo(int x);
			explicit operator int();
	};     
	int main()
	{
		Foo a = 42;		//error
		Foo b(42);
		Foo c = (Foo)(42);
		int d = c;		//error
	}
这样的话，main()中的隐式转换（第一、四行）将不被允许。

### 待续
C++博大精深，构造函数又是其中非常重要的部分，后面还需要继续学习、补充。