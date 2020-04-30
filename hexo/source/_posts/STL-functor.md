---
title: C++ STL仿函数
date: 2016-10-21 20:10:17
tags: C++
---
仿函数，又称为函数对象，即一种具有函数特质的对象，指的是那些可以被传入其他函数或是从其他函数返回的函数。
<!--more-->
### 函数指针与函数对象
我们知道，STL所提供的各种算法，往往有两个版本，其中一个版本表现出最常用（或最直观）的某种运算，第二个版本则表现出最泛化的演算流程，允许用户“以template参数来指定所要采取的策略”。以accumulate()为例，其第一版本是将指定范围内的所有元素相加，第二版本则允许你指定某种“操作”，取代第一版本的“相加”行为。所以，要将某种“操作”当做算法的参数，首先想到的办法就是先将该“操作”设计为一个函数，再将函数指针当做算法的一个参数；或者将该“操作”设计为一个仿函数（语言层面而言是个class），再以该仿函数产生一个对象，并以此对象作为算法的一个参数。      下面通过一个实际的例子说明这两种方法。   

	#include <iostream>
	
	using namespace std;
	
	int add_func(int a, int b){
    	return a+b;
	}
	
	class AddClass{
	public:
       	int operator()(int a, int b) { return a+b; }
	};

	typedef int (*AddFunction)(int a, int b);

	int main()
	{
		{
			AddClass addClass;
			int sum = addClass(1,2);
			cout << "addClass: " << sum << endl;
		}
		
		{
			AddFunction addFunction = &add_func;
			int sum = addFunction(1,2);
			cout << "addFunction: " << sum << endl;
		}
		return 0;
	}
可以看出，函数对象实际上就是一个重载了operator()操作符的类，这就是函数对象与普通类的差别。另外，函数对象和函数指针在定义的方式不一样，但是调用的方式是一样的。那既然已经有了函数指针这个东西，为什么还要发明函数对象呢？其实很简单，函数对象可以将附加数据保存在成员变量中，从而实现携带附加数据，而函数指针就不行了。考虑下面一个应用场景，我们需要使用std::for\_ecah将一个std::vector中的每一个值加上某个值然后输出，如果使用普通函数，则其声明应该为`void add_num(int value, int num);`，其中value为容器中的元素，num为要加上的数。但是由于std::for\_each函数的第三个参数要求传入接受一个参数的函数或函数对象，所以将add\_num函数传入std::for\_each是错误的，然而函数对象可以携带附加数据解决这个问题。   

	#include <iostream>
	#include <vector>
	#include <algorithm>
	
	using namespace std;

	class Add
	{
		public:
			void operator()(int value){
				cout << value + num_ << endl;
			}
		private:
			int num_;
	}；
	
	int main()
	{
		vector<int> ivec;
		ivec.push_back(1);
		ivec.push_back(2);
		ivec.push_back(3);

		Add add(2);
		std::for_each(ivec.begin(), ivec.end(), add);
		return 0;
	}
### 仿函数的型别
仿函数的相应型别主要用来表现函数参数型别和传回值型别。为了方便起见，<stl\_function.h>定义了两个classes，分别代表一元仿函数和二元仿函数（STL不支持三元仿函数），其中没有任何data members或member functions，唯有一些型别定义，任何仿函数，只要依个人需求选择继承其中一个class，便自动拥有了那些相应型别，也就自动拥有了配接能力。    
unary\_function和binary\_function分别用来呈现一元和二元函数的参数型别和返回值型别。用户可以继承它们，从而定义自己的仿函数。   

	template <class T>
	struct negate : public unary_function<T,T> {
		T operator() (const T& x) const { return -x; }
	}

	template <class T>
	struct plus : public binary_function<T,T,T> {
		T operator() (const T& x, const T& y) const { return x + y; }
	}
STL仿函数的分类，若以操作数的个数划分，可分为一元和二元仿函数，若以功能划分，可分为算术运算、关系运算和逻辑运算三大类。     
### lambda
lambda是C++11中引入的新特性，使程序员能定义匿名对象，而不必定义独立的函数和函数对象，这样使代码更容易编写和理解，又能防止别人的访问(调用)。简而言之，一个lambda函数是一个可以内联在代码中的函数，且通常也会传递给另外的函数（类似于仿函数）。    
下面用lambda来重写上面的for\_each例子。  

	#include <iostream>
	#include <vector>
	#include <algorithm>
	
	using namespace std;	
	
	int main()
	{
		vector<int> ivec;
		ivec.push_back(1);
		ivec.push_back(2);
		ivec.push_back(3);

		std::for_each(ivec.begin(), ivec.end(),
					  [](int x) { cout << x + x << endl; });
		return 0;
	}  
lambda表达式`[] (int x) { cout << x + 2 << endl; }`会让编译器产生一个类似前面例子中Add的未命名函数对象类。相比于函数对象类，lambda表达式具有如下优点：（1）简洁，不需要自己去实现函数对象类；（2）不会为临时的使用而引入新的名字，所以不会导致名字污染；（3）函数对象类名不如它的实际代码表达能力强，把代码放在更靠近它的地方将提高代码的清晰度。    
### std::function
lambda产生的闭包类型可以转换成std::function。std::function对象是对C++中现有的可调用实体的一种类型安全的包裹。  
	
	#include <iostream>
	#include <functional>
	#include <algorithm?
	#include <vector>
	
	using namespace std;
	
	int main()
	{
		vector<int> ivec;
		ivec.push_back(1);
		ivec.push_back(2);
		ivec.push_back(3);

		std::function<void(int)> func = [](int val) { std::cout << val+2 << endl; }
		std::for_each(ivec.begin(), ivec.end(), func);
		
		return 0;
	}
std::function是个类模板，可以对函数（普通函数、成员函数）、lambda表达式、std::bind的绑定表达式、函数对象等进行封装。std::function的实例可以对这些封装的目标进行存储、复制和调用等操作。  
### std::bind
函数模板std::bind能对普通函数、成员函数、静态成员函数、公共成员变量、公共静态成员变量等进行包装，调用std::bind的包装相当与将函数名和参数绑定在函数内部。std::bind函数模板返回的函数对象的类型是不确定的，但是可以存储在std::function内。std::bind绑定的参数是通过传值的方式传递的，如果需要通过引用传递则参数先需要用std::ref、std::cref进行引用，然后在传递给std::bind 将std::bind函数模板的返回值保存在std::function后，调用时需要传递的参数个数由std::bind中的占位符（std::placeholders::1、std::placeholders::2、std::placeholders::_3等）个数决定，即有几个占位符调用时就需要几个参数。     
	
	#include <iostream>
	#include <functional>
	
	using namespace std;
	
	int testFunc(int a, char c, float f){
		cout << a << endl;
		cout << c << endl;
		cout << f << endl;

		return a;
	}
	
	int main()
	{
		auto bindFunc1 = std::bind(testFunc, std::placeholders::_1, 'A', 100.1);
		bindFunc1(10);
		
		auto bindFunc2 = std::bind(testFunc, std::placeholders::_1, std::placeholders::_1, 100.1);
		bindFunc2('B', 10);

		auto bindFunc3 = bind(TestFunc, std::placeholders::_2, std::placeholders::_3, std::placeholders::_1);
    	bindFunc3(100.1, 30, 'C');

		return 0;
	}

### 总结
本文从STL functor开始，研究了那些可以被传入其他函数或是从其他函数返回的函数，包括函数指针、函数对象、lambda、function和bind。