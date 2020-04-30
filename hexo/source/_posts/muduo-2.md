---
title: muduo源码阅读之muduo的回调函数机制    
date: 2016-04-28 19:18:24
tags: muduo
---

回调是指将一段可执行的代码作为变量传给另外一部分代码，以供同步或异步调用。在Reactor模式中，在事件到来时调用相应的处理函数就是一种异步回调的过程。回调函数的实现可以由各种各样的方式。
<!--more-->
### C语言的函数指针
最熟悉的回调机制应该属于C语言的函数指针了。考虑一个简单的例子，实现对两个数的操作，这个操作可以是加减乘除等。     
首先声明一个函数指针：           

	typedef int (operFunc*)(int a, int b);
    
这个类型的函数指针可以做为参数传给用于实现两个数操作的函数，这个函数仅仅是简单的调用传入的函数指针：                                                                             	 

	int operation(int a, int b, operFunc func)
	{
		return func(a,b);
	}
现在可以定义各种回调函数:
    
	int add(int a, int b) 
	{
        return a + b;
	}
    int minus(int a, int b) 
	{   
		return a - b;
	}
	...
这样，在调用operation()时可以根据传入回调函数的不同，实现不同的功能。

    int main()
	{
		int a = 4;
		int b = 2;
		std::cout << operation(a,b,add) << std::endl;
		std::cout << operation(a,b,minus);
		...
	}
### 传统C++的回调
在传统的C++程序中，事件回调是通过虚函数进行的。网络库往往会定义一个或几个抽象基类，其中声明一些(纯)虚函数，使用者需要继承这些基类，并覆写这些虚函数，以获得事件回调通知。由于C++的动态绑定只能通过指针和引用实现，使用者必须把派生类（myHandler）对象的指针或引用隐士转换为基类(Handler)的指针或引用。网络库调用积累的虚函数，通过动态绑定机制实际调用的是用户在派生类中覆写的虚函数。  
下面通过一段代码示例看看虚函数如何实现回调。   
    
    #include <stdio.h>
	class Handler	//interface
	{
		public:
			virtual void say(const char* text) = 0;
	};
	
	class MyHandler : public Handler
	{
		public:
			void say(const char* text)
			{
				printf("hello,%s",text);
			}	
	};

	int main()
	{
		Handler *handler;
		MyHandler myHandler;
		handler = dynamic_cast<Hander*>(myHandler);
		handler.say("world");

		return 0;
	}

### muduo采用的事件回调
muduo使用的是boost库的boost::function和boost::bind实现函数回调。现在，已经可以在c++11中使用它们。    
boost::function类似于函数指针的封装，boost::function类型的变量可以保存一个可以调用的函数指针。可以指向任何函数，包括成员函数。所以boost::function与boost::bind结合起来使用可以实现一些函数指针无法完成的功能，比如，绑定特定变量到函数上，实现某些函数式编程语言的功能，甚至是真正的闭包。    
以下的例子均使用c++11。
    
    #include <functional>
	#include <string>
	#include <iostream>
	
	namespace
	{
		void function(int number, float floation, std::string string)
		{
			std::cout << "Int: \"" << number << "\" Float: \"" << floatation
					  << "\" String: \"" << string << "\"" << std::endl; 
		}
	}

	int main()
	{
		//declare function pointer variables 
		std::function<void(std::string, float, int)> shuffleFunction;
		std::function<void(void)> voidFunction;
		std::function<void(float)> reduceFunction;
		
		//bind the method
		shuffleFunction = std::bind(::function,_3,_2,_1);
		voidFunction = std::bind(::function, 5,5.f, "five");
		reduceFunction = std::bind(::function, 13, _1, "empty");
		
		//call
		shuffleFunction("String",0.f,0);
		voidFunction();
		reduceFunction(13.f);
	
	}
  
传统的指向成员函数的回调函数用法如下例。	
	                              		
    class A
	{
		public:
			int add(int a, int b)
			{
				std::cout << "A::add()" << endl
				return a + b;
			}
	}；
    typedef int (A::*operFunc)(int, int);
	int main(void)
	{
    	operFunc oper = &A::add;
    	A ca;
		int a = 2;
    	int b = 3;
    	int res = (ca.*oper)(a, b);
    	std::cout << res << std::endl;
	}
成员函数具有隐含的this指针，在调用成员函数时必须通过类的实例。所以成员函数指针变量不仅在定义时有特殊的语法，更重要的是在实际调用回调函数必须采用(a.*oper)(a,b)这样的形式。这样的语法比较复杂，由于与非成员函数的声明不同，无法传入同一个函数。所以，需要调用函数既能接收成员函数又能接收非成员函数。     
这就是在muduo库中广泛应用的function和bind的机制，如下示例代码。

	class A
	{
		...
	}；
    typedef std::function<int(int,int)> operFunc;
	int main()
	{
		int a = 2;
		int b = 4;
		operFunc oper;
		A ca;
		oper = std::bind(&A::add, &ca, _1, _2);
		std::out << operation(a,b,oper) << std::endl;
		...
	}
muduo使用function和bind实现了闭包的功能，即，函数+引用环境，或者说是附有数据的行为，因为，在事件到来时调用回调函数处理，通常还需要知道其他的一些信息，比如在网络程序中，可能是连接当前的状态等。     
在muduo中，一个回调函数通常是一个类的成员函数，所以这个回调函数同时也保存了某个类实例的信息，包括这个类的成员变量、成员函数等，在实际调用函数的时候就可以使用这些信息和状态，这样的处理当时让程序的编写显得非常清晰和方便。    

