---
title: 字符串的排列与组合
date: 2016-10-23 19:09:33
tags: 数据结构与算法
---
很久没有深入研究数据结构和算法了...    
这次换个套路，先不说那些干瘪瘪的理论，首先来看一道编程题。   
<!--more--> 
### 题目描述
编写一个方法，确定某字符串的所有排列组合。  
给定一个string A[]和一个int n，代表字符串和其长度，请返回所有该字符串字符的排列，保证字符串长度小于等于11且字符串中字符均为大写英文字符，排列中的字符串按字典序从大到小排序。  
样例： “ABC”     
返回：["CBA","CAB","BCA","BAC","ACB","ABC"]

### 思考
...

### 分析
#### next_permutation
很容易看出来这是一道排列的问题。由于自己最近对STL甚是着迷，却又没有掌握透彻，故先写出了下面的错误代码。
	
	vector<string> getPermutation(string A){
		std::sort(A.begin(), A.end());
		vector<string> ret;
		do{
			ret.push_back(A);
		}while(next_permutation(A.begin(), A.end()));
		std::sort(ret.begin(), ret.end(), greater<string>());
		return ret;
	}
当我满怀欣喜地提交代码运行时，却被告知通过率只有13.3%，给出的错误分析表明，当字符串中含有两个及以上的相同字符时，所得结果错误，即原题的意思是字符串中的每一个字符都是独立的，即使它们相同。所以，得出的结果应该有n!个字符串。而算法next\_permutation不能处理这样的情况。比如输入"AAB"，使用next\_permutation得到的结果如下图所示。     
![](http://i.imgur.com/40N1AVT.png)
#### 递归
递归方法的思想很容易理解，即从串中依次选出每一个元素，作为排列的第一个元素，然后对剩余的元素递归的进行全排列。     
以对字符串"abc"进行全排列为例：    
（1）固定a，求后面bc的排列：abc、acb，求好后，b放在第一位置，得到bac；      
（2）固定b，求后面ac的排列：bac、bca，求好后，c放在第一位置，得到cba；    
（3）固定c，求后面ba的排列，cba、cab。     
其过程如下图所示：   
![](http://i.imgur.com/iqS7F6D.png)
分析了算法思想后，就很容易写出代码了。
	
	void getPermutation(vector<string>& ret, string A, int start){
		int end = A.size()-1;
		if(start == end){
			ret.push_back(A);
			return;
		}else{
			for(int j=start; j<=end; ++j){
				std::swap(A[start], A[j]);		//轮流固定在第一个位置
				getPermutation(ret, A, start+1);
				std::swap(A[start], A[j]);
			}
		}
	}
运行上面的例子，结果如下图所示。    
![](http://i.imgur.com/aVKXDCF.png)   
所以，其实STL算法next_permutation是去掉了重复的全排列。
### 组合
如果不是求字符的所有排列，而是求其所有组合呢？比如字符a、b、c，它们的组合有a,b,c,ab,ac,bc,abc。    
#### 基于位图的算法
简单的数学知识：假设原有n个字符，则最终组合结果是2^n-1个。采用位操作：假设有元素a、b、c三个，规定二进制1表示取该元素，0表示不取。故取a的二进制表示为001，ab为011，依次类推，000没有意义。   
所以，这些结果与位图的对应关系：
001, 010, 011, 100, 101, 110, 111      
a, b, ab, c, ac, bc, abc 
因此，可以循环1~2^n-1，然后输出对应代表的组合即可。    

	void Combination(string A){
		int n=1<<A.size();
		for(int i=1;i<n;++i){
			for(int j=0;j<A.size();++j){
				if((1>>j)&i)		//对应位上为1，则输出对应的字符
					cout << A[j];
			}						//每一个组合
			cout << endl;
		}
	}  
代码运行结果如下图所示：    
![](http://i.imgur.com/hJmuKDD.png)