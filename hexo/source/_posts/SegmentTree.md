---
title: 数据结构之线段树    
date: 2016-09-23 21:28:16
tags: 数据结构与算法
---

线段树，又称为区间数，是一种二叉搜索树，它将一个区间划分成一些单元区间，每个单元区间对应线段树中的一个叶节点。而对于线段树中的每一个非叶节点[a,b]，它的左子树表示的区间为[a,(a+b)/2]，右子树表示的区间为[(a+b)/2+1,b]，因此线段树是平衡二叉树（结构平衡的二叉搜索树，即叶节点的深度差不超过1）。叶节点数目为N，即整个线段区间的长度。
<!--more-->
线段树具有如下性质：   
（1）线段树是一棵平衡树，使用线段树可以快速地查找某一个节点在若干条线段中出现的次数，时间复杂度为O(logN)。    
（2）线段树同一层的节点所代表的区间，相互不会重叠。    
（3）线段树任两节点要么是包含关系要么是没有公共部分，不可能部分重叠。   
（4）给定一个叶子l，从根到l路径上所有节点代表的区间都包含l，且其他节点代表的区间都不包含l。     
一棵[1,10]的线段树表示如下。注意，线段树的构造在各区间的端点处的处理方式不一样，会导致线段树最终的表示有些差别，但是本质上是一样的。       
![](http://i.imgur.com/JEkWmlu.png)

### 线段树的基本操作
线段树的基本操作和普通二叉树很类似，只不过线段树的节点上存储的是一个区间（左右端点值）。下面以线段树的建立、插入线段和删除线段为例简要说明线段树的基本操作。
#### 线段树的构建   
首先定义线段树的结构，这里采用链表的方式组织。     
	
	struct Node
	{
		int left,right;
		int cover;			
		Node* leftChild;
		Node* rightChild;	
	};
其中，cover字段用于计算一条线段被覆盖的次数。接下来，以递归的方式构建线段树。     
	
	Node* build(int l, int r)
	{
		Node* root = new Node();
		root->left = l;
		root->right = r;
		root->cover = 0;
		root->leftChild = NULL;
		root->rightChild = NULL;
		if(r-l > 1)
		{
			int mid = (l+r) >> 1;
			root->leftChild = build(l,mid);
			root->rightChild = build(mid,r);		//build(mid+1,r)
		}
		return root;
	}

#### 线段插入与删除
如上所述，通过cover字段来计算一条线段被覆盖的次数。插入与删除时更新相应线段的cover。     
	
	void Insert(int start, int end, Node* root)
	{
    	if(root==NULL || start>root->right || end<root->left)
    	{
        	cout << "error" << endl;
        	return;
    	}
    	if(start<=root->left && end>=root->right)
    	{
        	root->cover++;
        	return;
    	}
    	int mid=(root->left+root->right)>>1;
    	if(end<=mid)
    	{
        	Insert(start,end,root->leftChild);
    	}
    	else if(start>=mid)
    	{
        	Insert(start,end,root->rightChild);
    	}
    	else
    	{
        	Insert(start,mid,root->leftChild);
        	Insert(mid,end,root->rightChild); 
    	}
	}
	
	void Delete(int start, int end, Node* root)
	{
    	if(root==NULL || start>root->right || end<root->left)
    	{
        	cout << "error" << endl;
        	return;
    	}
    	if(start<=root->left && end>=root->right)
    	{
        	root->cover--;
        	return;
    	}
    	int mid=(root->left+root->right)>>1;
    	if(end<=mid)
    	{
        	Delete(start,end,root->leftChild);
    	}
    	else if(start>=mid)
    	{
        	Delete(start,end,root->rightChild);
    	}
    	else
    	{
        	Delete(start,mid,root->leftChild);
        	Delete(mid,end,root->rightChild);
    	}
	}	

### 线段树的应用

TODO