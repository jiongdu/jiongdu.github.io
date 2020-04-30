---
title: 更好地使用STL关联容器
date: 2016-12-30 10:13:50
tags: C++
---
在STL的使用过程中，一直对关联容器掌握的不够熟练。这一篇就来总结下使用关联容器时的一些注意问题。
<!--more-->
### 理解等价关系
在STL中，对两个对象进行比较，看它们的值是否相等，这样的操作随处可见。在实际操作中，相等的概念是基于operator==的。如果表达式“x==y”返回真，则x和y的值相等。    
等价关系是以“在以排序的区间中对象值的相对顺序”为基础的。如果从每个标准关联容器的排列顺序来考虑等价关系，那么这将是非常有意义的。对于两个对象x和y，如果按照关联容器c的排列顺序，每个都不在另一个的前面，那么称这两个对象按照c的排列顺序由等价的值。      
下面以一个实例进行说明：    

	bool ciCharLess(char c1, char c2){
		return tolower(static_cast<unsigned char>(c1))<tolower(static_cast<unsigned char>(c2));
	}

	bool ciStringCompare(const string& s1, const string& s2){
		return lexicographical_compare(s1.begin(), s1.end(), s2.begin(), s2.end(), ciCharLess);
	}

	struct CIStringCompare : public binary_function<string, string, bool>{
		bool operator()(const string& lhs, const string& rhs){
			return ciStringCompare(lhs, rhs); 
		}
	}

	int main()
	{
		set<string, CIStringCompare> s;
		s.insert("STL");
		s.insert("stl");

		for(auto n:s){
			cout << n << endl;			//STL
		}
	
		if(s.find("stl")!=s.end()){
        	cout << "success" << endl;		//success
    	}else{
        	cout << "fail" << endl;
    	}
		if(std::find(s.begin(), s.end(), "stl")!=s.end()){
        	cout << "success" << endl;
    	}else{
        	cout << "fail" << endl;			//fail
    	}
		return 0;
	}
s是一个不区分大小写的set<string>，即当set的比较函数忽略字符串中字符的大小写时的set<string>。这样一个比较函数将把“STL”和“stl”看做是等价的。因此，在先后插入“STL”和“stl”时，只有“STL”会成功插入。如果使用set的find成员函数来查找“stl”时，该查找会成功；而如果是使用非成员的find算法，则查找将失败。因为“STL”和“stl”是等价的。顺便说一句，该例子从一个方面解释了为什么应该优先选用成员函数而不是与之对应的非成员函数。
### 为包含指针的关联容器指定比较类型
假如有一个包含string*指针的set，把一些动物的名字插入到该集合中：     
	
	set<string*> ssp;
	ssp.insert(new string("Anteater"));
	ssp.insert(new string("Wombat"));
	ssp.insert(new string("Lemur"));
	ssp.insert(new string("Penguin"));
因为集合中所包含的是指针，所以，可能会想到使用下面的代码来打印出动物的名字：
	
	for(set<string*>::const_iterator i=ssp.begin(); i!=ssp.end(); ++i){
        cout << **i << endl;
    }
没错，动物的名称会被打印出来，但它们以字母顺序出现的概率仅为1/24。ssp会按顺序保存它的内容，但因为它包含的是指针，所以会按指针的值而不是按字符串的值进行排序，4个指针的值共有24个可能的排列方式，所以对要存储的指针会有24种可能的排列。    
为了解决这个问题，需要知道set<string\*> ssp是如下代码set<string\*,less<string*>> ssp的缩写，当然，更精确的讲是set<string\*,less<string\*>,allocator<string\*>>的缩写，只是这里不考虑分配子的影响。         
因此，如果想让string\*指针在集合中按字符串的值排序，那么不能使用默认的比较函数子类。必须自己编写比较函数子类。        

	struct StringPtrLess : public binary_function<const string*, const string*, bool>{
    	bool operator() (const string* s1, const string* s2) const{
        	return *s1 < *s2;
    	}
	};  

	set<string*, StringPtrLess> ssp;

	/*
	void print(const string* ps){
		cout << *ps << endl;
	}

	for_each(ssp.begin(), ssp.end(), print);
	*/

现在上述的打印循环可以做到预期的事情了。     
所以，当需要创建包含指针的关联容器时，容器将会按照指针的值进行排序。绝大多数情况下，这不会是所希望的，这种情况下，几乎肯定要创建自己的函数子类作为该关联容器的比较类型。

### 考虑用排序的vector替代关联容器
个人使用STL的经历中，当需要一个可提供快速查找功能的数据结构时，都会立刻想到标准关联容器，即set、multiset、map和multimap。但是，它们并不总是适合的。比如，如果查找速度真的很重要，那么，非标准的散列容器（unordered_map等）几乎是值得考虑的。因为通过适当的散列函数，散列容器几乎能提供常数时间的查找能力，优于set、multiset、map和multimap的确定的对数时间查找能力。          
但是，即使确定的对数时间查找能力满足需求，标准关联容器可能也不是最好的选择。标准关联容器的效率比vector还低的情况并不少见。标准关联容器通常被实现为平衡的二叉查找树。二叉查找树这种数据结构对插入、删除和查找的混合操作做了优化，也就是，它所适合的那些应用程序的主要特征是插入、删除和查找混在一起。即没办法预测出针对这棵树的下一个操作是什么。      
而还有很多应用程序使用其数据结构的方式并不这么混乱。它们使用其数据结构的过程可以明显地分为三个阶段：      
（1）设置阶段。创建新的数据结构，并插入大量元素。      
（2）查找阶段。查询该数据结构以找到特定的信息。    
（3）重组阶段。改变该数据结构的内容。         
对于以这种方式使用其数据结构的应用程序来说，vector可能比关联容器提供了更好的性能。但是不是任意的vector，而必须是排序的vector，因为只有对排序的容器才能够正确地使用查找算法binary\_search、lower\_bound和equal\_range等。      
那么，为什么通过排序的vector执行的二分搜索，比通过二叉查找树执行的二分搜索具有更好的性能呢？      
其原因主要是：关联容器几乎肯定在使用平衡二叉树。这就意味着在一个关联容器中存储一个类型所伴随的空间开销至少是三个指针（父指针，左儿子，右儿子）。相反，存储在vector中则不会有任何的额外开销；只是简单地存储一个类型。    
当然，对于排序的vector，最不利的地方在于它必须保持有序！当一个新的元素被插入时，新元素之后的所有元素都必须向后移动一个元素的位置。当一个元素从vector中删除了，则在它之后的所有元素也都要向前移动。插入和删除操作对于vector来说是昂贵的，但对于关联容器却是廉价的。这就是为什么当“对数据结构的使用方式是：查找操作几乎从不跟插入和删除操作混在一起”时，再考虑使用排序的vector而不是关联容器才是合理的。     

### 在map::operator[]和map::insert之间选择

map的operator[]函数与众不同。它与vector、deque和string的operator[]函数无关，与用于数组的内置operator[]也没有关系。相反，map::operator[]的设计目的是为了提供“添加和更新”的功能，也就是说，对于map<K,V> m；来说，表达式m[k]=v;检查键k是否已经在map中了，如果没有，它就被加入，并以v作为相应的值。如果k已经在映射表中了，则与之关联的值被更新为v。      
下面以一个例子来说明：      

	class Widget {
		public:
			Widget();
			Widget(double weight);
			Widget& operator=(double weight);
		private:
			double weight_;
		...		
	};

	map<int, Widget> m;
	m[1]=1.50;

表达式m[1]是m.operator\[\]\(1)的缩写形式，即对map::operator[]的调用。该函数必须返回一个指向Widget的引用，因为m所映射的值类型是Widget。这时，m中什么都没有，所以键1没有对应的值对象。因此，operator[]默认构造了一个Widget，作为与1相关联的值，然后返回一个指向该Widget的引用。最后，这个Widget成了赋值的目标。因此，m[1]=1.50在功能上等价于以下代码：      
	
	typedef map<int, Widget> IntWidgetMap;
	
	pair<IntWidgetMap::iterator, bool> result = m.insert(IntWidgetMap::value_type(1, Widget()));

	result.first->second = 1.50;
因此，使用operator[]会降低效率。因为我们先默认构造了一个Widget，然后立刻赋给它新的值。而如果我们换成对insert的直接调用：    
	
	m.insert(IntWidgetMap::value_type(1, 1.50));

最终效果和前面相同，但是它通常会节省三个函数调用：一个用于创建默认构造的临时Widget对象，一个用于析构该临时对象，还有一个是调用Widget的赋值描述符。     
而operator[]的设计目的是为了提供“添加和更新”的功能，现在我们已经知道，当做为“添加”操作时，insert比operator[]效率更高，而当我们做更新操作时，即当一个等价的键已经在映射表中时，形势就反过来了。因为调用insert时，必须构造和析构一个pair类型的对象，需要付出一个pair构造函数和一个pair析构函数的代价。而这又会导致对Widget的构造和析构动作。而operator[]不使用pair对象，所以它不会构造和析构任何pair或Widget。      
总结：当向映射表中添加元素时，优先选用insert而不是operator[]；而当更新已经在映射表中的元素的值时，要优先选择operator[]。
    