---
title: 使用STL算法时需注意的问题
date: 2016-12-19 20:21:14
tags: C++
---
接着上一篇，对STL中非常有用而又容易犯错的一些算法做总结和记录。希望在后面的使用中更加注重STL算法。     
<!--more-->
### 确保目标空间足够大
如上篇所述，当有新的对象加入容器时，STL容器会自动扩充存储空间以容纳这些对象。但是，需要注意的是，STL容器并不总是能够正确地管理其存储空间。比如下面这个例子：     

	int trans(int x){
		return -x;
	}

	vector<int> ivec{1,2};
	vector<int> res;

	std::transform(ivec.begin(), ivec.end(), res.begin(), trans);	// error

这段代码中，transform首先以ivec[0]为参数调用trans，并将结果赋给*\res.end()。然后，再以ivec[1]为参数调用trans，并将结果赋给\*(res.end()+1)。这可能会引起灾难性的后果，因为在\*res.end()中并没有对象，\*(res.end()+1)就更没有对象了。这种transform调用时错误的，因为导致了对无效对象的赋值操作。       
犯这种错误的程序员总是希望他们所调用的算法的结果会被插入到目标容器中。事实上，必须向STL明确表达意图。所以，上面的例子需要通过调用back\_insert生成一个迭代器来指定目标区间的起始位置：      

	int trans(int x){
		return -x;	
	}

	vector<int> ivec{1,2};
	vector<int> res;
	
	std::transform(ivec.begin(), ivec.end(), back_inserter(res), trans);
在内部，back\_insert返回的迭代器将使得push\_back被调用，所以back\_insert可使用于所有提供push\_back方法的容器。而如果想让一个算法在容器的头部而不是尾部插入对象可以使用front\_inserter。      
因此，无论何时，如果所使用的算法需要指定一个目标区间，那么必须确保目标区间足够大，或者确保它会随着算法的运行而增大。要么在算法执行过程中增大目标区间，请使用插入型迭代器，比如ostream_iterator，或者由back\_inserter、front\_inserter返回的迭代器。

### 与排序有关的选择

提到排序，首先想到的是std::sort。但需要注意的是，std::sort并非在任何场合下都是完美无缺的。有些场景并不需要完全的排序操作。比如使用partial\_sort来选择前n个最好的商品送给n个最重要的客户，或者当只需要将最好的20个商品送给最重要的20个客户，而不关心哪个商品送给哪位客户的时候选用nth\_element。   

	vector<int> ivec{2,5,10,23,12,56,34,100};

	std::partial_sort(ivec.begin(), ivec.begin()+4, ivec.end(), greater<int>());
	for(auto n : ivec){
		cout << n << endl
	}

运行结果：   
![](http://i.imgur.com/5BIVF20.png)   

	vector<int> ivec{2,5,10,23,12,56,34,100};

	std::nth_element(ivec.begin(), ivec.begin()+3, ivec.end(), greater<int>());
	for(auto n : ivec){
		cout << n << endl
	}
运行结果:       
![](http://i.imgur.com/aZpaEcc.png)    
可以看出，对nth\_element的调用与partial\_sort几乎一样。在效果上唯一不同之处在于，partial\_sort对前20个元素进行了排序，而nth\_element没有对他们进行排序。         
sort、partial\_sort、nth\_element和stable\_sort都属于非稳定的排序算法，有一个名为stable\_sort的算法可以提供稳定排序特性。    
另外，sort、partial\_sort、nth\_element和stable\_sort算法都要求随机访问迭代器，所以这些算法只能被应用于vector、string、deque和数组。对标准关联容器中的元素进行排序并没有实际意义，因为这样的容器总是使用比较函数来维护内部元素的有序性。list是唯一需要排序却无法使用这些排序算法的容器，为此，list特别提供了sort成员函数（稳定排序）。    

### remove算法后调用erase删除元素
我们需要知道，从容器中删除元素唯一的办法是调用容器的成员函数，几乎总是erase的某种形式。因此，remove并不知道它操作的元素所在的容器，所以remove不可能从容器中删除元素。那么remove究竟做了什么？       
其实，remove移动了区间中的元素，其结果是，“不用背删除”的元素移到了区间的前部（保持原来的相对顺序）。返回的是一个迭代器指向最后一个“不用背删除”的元素之后的元素。这个返回值相当于该区间“新的逻辑结尾”。但是，需要注意的是，通常来说，当调用了remove以后，从区间中被删除的那些元素可能在也可能不在区间中，这是算法操作的附带结果。         
所以，如果真想删除元素，那就必须在remove之后使用erase。要删除的元素的位置从“新的逻辑结尾”一直到原区间的结尾。比如:

	vector<int> ivec;
	...
	v.erase(remove(v.begin(), v.end(), 99), v.end());	
此外，remove并不是唯一一个适用于这种情形的算法，其他还有两个属于“remove类”的算法：remove\_if和unique。

### 使用accumulate或者for_each进行区间统计

有时候，我们需要按照某种自定义的方式对区间进行统计处理。这个时候，必须自己定义统计方法。STL通过算法accumulate来提供。    
accumulate有两种形式：第一种形式有两个迭代器和一个初始值，它返回该初始值加上由迭代器标识的区间中的值的总和。如下面的例子：      

	vector<int> ivec{1,2,3};
	
	std::accumulate(ivec.begin(), ivec.end(), 0);
而另一种形式是accumulate带一个初始值和一个任意的统计函数。

	vector<int> ivec{2,2,3};
	
	std::accumulate(ivec.begin(), ivec.end(), 2, multiplies<int>());  //24	
另一个可用来统计区间的算法时for\_each，如图accumulate一样，for\_each也带两个参数：一个是区间，另一个是函数（通常是函数对象）---对区间中的每个元素都要调用这个函数，但是，传给for\_each的这个函数只接受一个实参（即当前的区间元素）；for\_each执行完毕后会返回它的函数。

### 使用排序的区间作为参数的算法

有些算法既可以与排序的区间一起工作，也可以与未排序的区间一起工作，但是当它们作用在排序的区间上时，算法会更加有效。         
首先，罗列出要求排序区间的STL算法:     
binary\_search, lower\_bound, upper\_bound, equal\_range, set\_union, set\_intersection, set\_difference, set\_symmetric\_difference, merge, inplace\_merge, includes     
还有一些算法并不一定要求排序的区间，但通常情况下会与排序区间一起使用。     
unique, unique\_copy         
这其中，用于查找的算法binary\_search, lower\_bound, upper\_bound, equal\_range要求排序的区间，因为它们用二分法查找数据。         
set\_union, set\_intersection, set\_difference, set\_symmetric\_difference这四个算法提供了线性时间效率的集合操作。它们要求排序的区间，因为如果不满足，它们就无法在线性时间内完成工作。       
merge和inplace\_merge实际上实现了合并和排序的联合操作：它们读入两个排序的区间，然后合并成一个新的排序区间。如果源区间没有排过序，就不可能在线性时间内完成。    
同样，includes也要求两个区间是排序的，承诺线性时间的效率。   
而unique和unique\_copy与上述讨论过的算法有所不同，它们即使对于未排序的区间也有很好的行为。以unique为例，unique用于删除区间中所有重复的元素，所以必须保证所有相等的元素都是连续存放的。因此，总是要确保传给unique的区间是排序的。      
顺便提一下，unique使用了与remove类似的办法来删除区间中的元素，而并非真正意义上的删除。
        

      

    

	
