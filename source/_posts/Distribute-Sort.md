---
title: 分配排序（桶排序和基数排序）总结
date: 2017-01-20 19:51:47
tags: 数据结构与算法
---
分配排序，是指不需要进行两两之间的比较，而根据记录自己的关键码的分配来进行排序的一类方法，因此，在进行分配排序时，我们通常需要知道记录序列的一些具体情况，比如关键码的分布。分配排序主要包括桶排序和基数排序。   
<!--more-->
### 桶排序
首先来介绍桶排序。如果我们知道序列中的记录都位于一个比较小的区间范围之内，那我们可以把相同值的记录分配到同一个桶里面，然后依次收集这些桶就能得到一个有序的序列。    
例如，下图中，我们知道待排序数组的记录值在0~9的范围之内，因此，我们可以设定10个桶，然后就把这些记录值分配到各个桶里面。
![](http://i.imgur.com/SVkuDWp.png)  
此外，我们还维护了前若干桶的一个累加值，并且，在进行排序的时候，我们从数组记录值的最右边开始遍历，这样来保持排序的稳定性。 
![](http://i.imgur.com/yGcdDMb.png)

#### 桶排序的实现
在弄清楚了原理之后，桶排序的实现就很简单了。

	 void BucketSort(vector<int>& nums, int max){
	 	int n=nums.size();
	 	vector<int> temp(begin(nums), end(nums));
	 	vector<int> count(max);
	 	for(int i=0;i<max;i++){
		 	count[i]=0;
	 	}
	 	for(int i=0;i<n;i++){
		 	count[nums[i]]++;
	 	}
	 	for(int i=0;i<max;i++){
		 	count[i]=count[i-1]+count[i];
	 	}
	 	for(int i=n-1;i>=0;i--){	
		 	nums[--count[temp[i]]]=temp[i];
	 	}
	}

#### 桶排序性能分析
由前面的分析可知，桶排序使用的场景是：待排序的数组长度为n，所有数组记录值位于[0,m)上，且m相对于n很小。    
因为排序过程需要遍历原数组和count数组，所以总的时间复杂度为O（n+m）。空间代价为O（n+m），包括临时数组和计数器。桶排序是稳定的。   
### 静态基数排序
我们已经知道，桶排序只适合m很小的情况，那如果出现m很大，甚至大于n的情况，应该怎么办呢？不难想象，我们可以把这些记录的关键码认为地拆成几个部分，然后再运用几次桶排序，从而完成整个排序过程，这就是基数排序的思想，基数排序又可以分为静态基数排序和链式基数排序。        
比如，我们需要对0到9999之间的整数进行排序，这时直接使用桶排序显然是代价较高的。我们可以将四位数看做是由四个排序码决定，个位为最低排序码。基数等于10。然后，按照千，百，十，个位数字依次进行4次桶排序。     
下面以一个记录值都是两位数的数组的排序过程来说明静态基数排序两趟桶排的具体过程，这里和后面的代码都采用的是低位优先法。
![](http://i.imgur.com/gPF11Hv.png)
![](http://i.imgur.com/hr8v9TV.png)

#### 静态基数排序的实现

同样，给出静态基数排序的实现。

	void RadixSort(vector<int>& nums, int d, int r){
		int n=nums.size();
		vector<int> temp(n,0);
		vector<int> count(r,0);		//r:radix,10
		int radix=1;
		int i,j,k;
		for(i=1;i<=d;i++){		//d:排序码的个数
			for(j=0;j<r;j++){
				count[j]=0;
			}
			for(j=0;j<n;j++){
				k=(nums[j]/radix)%r;
				count[k]++;
			}
			for(j=1;j<r;j++){
				count[j]=count[j]+count[j-1];
			}
			for(j=n-1;j>=0;j--){
				k=(nums[j]/radix)%r;
				count[k]--;
				temp[count[k]]=nums[j];
			}
			for(j=0;j<n;j++){
				nums[j]=temp[j];
			}
			radix*=r;
		}
	}
#### 静态基数排序性能分析
同样的，静态基数排序也需要一个临时数组和r个计数器，所以总的空间代价为O（n+r）。时间代价则为O（d*(n+r)），因为它相当于进行了d次桶式排序。此外，静态基数排序也是稳定的。

### 链式基数排序
基于静态链的基数排序相对于静态基数排序的差别在于将分配出来的子序列存放在r个静态链组织的队列中。   
下面，同样以一个例子来说明链式基数排序的过程。    
假设有待排序数组[97, 53, 88, 59, 26, 41, 88, 31, 22]，首先按照个位进行第一趟分配，分配完元素后的队列如下图所示。   
![](http://i.imgur.com/D5ZCnQk.png)    
然后进行第一趟收集，即个位有序：[41, 31, 22, 53, 26, 97, 88, 88]。    
接下来，再按照十位进行第二趟分配。    
![](http://i.imgur.com/BUeHB5r.png)    
最后进行第二趟收集，即完成最终的排序。

#### 链式基数排序的实现

下面给出链式基数排序的实现。

	typedef struct Node{
		int key;
		int next;	//下一个节点在数组中的下标
	}Node;

	typedef struct StaticQueue{
		int head;
		int tail;
	}StaticQueue;

	void Distribute(vector<Node>& node, int first, int i, int r, StaticQueue* queue);
	void Collect(vector<Node>& node, int& first, int r, StaticQueue* queue);

	void RadixSort(vector<int>& nums, int d, int r){
		int i, first=0;
		int n=nums.size();
		vector<Node> node(n);
		StaticQueue* queue = new StaticQueue[r];
		for(i=0;i<n-1;i++){
			node[i].key=nums[i];
			node[i].next=i+1;
		}
		node[n-1].key=nums[n-1];
		node[n-1].next=-1;
	
		for(i=0;i<d;i++){			//d趟的分配和收集
			Distribute(node, first, i, r, queue);			//分配到不同队列
			Collect(node, first, r, queue);					//聚合
		}
		for(i=0;i<r;i++){
			if(queue[i].head==-1) continue;
			else break;
		}
	
		int j=queue[i].head;
		while(node[j].next!=-1){		//排序结果
			cout << node[j].key << endl;
			j = node[j].next;
		}
		cout << node[j].key << endl;
	
		delete[] queue;
	}

	void Distribute(vector<Node>& node, int first, int i, int r, StaticQueue* queue){
		int current=first;
		for(int j=0;j<r;j++){
			queue[j].head=-1;
		}
		while(current!=-1){
			int k = node[current].key;
			for(int a=0;a<i;a++){
				k=k/r;
			}
			k=k%r;
			if(queue[k].head==-1){
				queue[k].head=current;
			}else{
				node[queue[k].tail].next=current;
			}
			queue[k].tail=current;
			current=node[current].next;
		}
	}

	void Collect(vector<Node>& node, int& first, int r, StaticQueue* queue){
		int last,k=0;
		while(queue[k].head==-1) k++;
		first=queue[k].head;
		last=queue[k].tail;
		while(k<r-1){
			k++;
			while(k<r-1 && queue[k].head==-1){
				k++;
			}
			if(queue[k].head!=-1){
				node[last].next=queue[k].head;
				last=queue[k].tail;
			}
		}
		node[last].next=-1;
	}

#### 链式基数排序性能分析
由前面的分析和代码可知，链式基数排序在分配和收集的过程中，不需要移动记录本身，只是在做记录next指针的修改。因此，它的时间代价和空间代价与静态基数排序一致，分别为O（d*(n+r)）和O（n+r）。并且，根据队列的先入先出的特点，链式基数排序也是稳定的。

### 总结
本文总结了分配排序中最常见的桶排序和基数排序，并给出了具体的实现和相关分析。可以看出，分配排序不会对记录值进行两两比较，因此不受O（nlgn）时间复杂度的限制。从另外一个角度看，分配排序也可以看做是以空间换时间的典型方法。