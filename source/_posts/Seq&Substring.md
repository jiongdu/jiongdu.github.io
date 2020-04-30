---
title: 动态规划之最长公共子序列，最长公共子串，最长递增子序列，最长子序列和等  
date: 2016-03-25 15:05:10
tags: 数据结构与算法
---

近来，重新研究了动态规划的一些常见问题，特别是针对数组、子序列和串。学习过程又有了些新的看法与感悟，特记录如下。

总结：动态规划是算法设计中非常重要的思想，值得我们多领悟、总结。   
1.在研究动态规划过程中，一定要深刻理解递归解决公共子问题，并且要能将其和分治法区分开。   
2.状态转移方程，就是反应解决动态规划的思路。当理解写出状态转移方程后，离成功就不远了。   
3.针对很多关于串、子串的问题，注意边界问题的处理。       
<!--more-->

### 最大子数组和
[leetcode：maximum-subarray](https://leetcode.com/problems/maximum-subarray/)  
说明：找出和最大的子数组。当然，子数组是连续的序列。    
动态规划方案：使用两个变量，分别保存到目前为止的局部最优解和全局最优解。为什么要这样呢？因为，局部最优不一定是全局最优。  

核心代码：   

     for(int i=1; i<nums.size(); i++)
   	 {
    	local = max(nums[i], nums[i]+local);
		global = max(global, local);
   	 }

### 最长递增子序列长度
[leetcode:longest increasing subsequence](https://leetcode.com/problems/longest-increasing-subsequence/)    
说明：子序列，不要求连续。这里，只说明获取最长子序列的长度。     
动态规划方案：使用dp[i]保存到目前为止的最长递增子序列长度，maxRet保存整个序列的最长递增子序列长度。把当前数据值与其前面所有的数据进行比较，从而更新子序列的长度。   

核心代码：
	
	 vector<int> dp(n,0);
	 dp[0]=1; 
	 int maxRet=0;
     for(int i=1; i<n; i++)
   	 {
		for(int j=0; j<i; j++)
		{
			if(nums[i]> nums[j])
				dp[i] = max(dp[i], dp[j]); 
			dp[i] += 1;	
			maxRet = max(maxRet, dp[i]); 
		}
		return maxRet;
   	 }

### 最长公共子串 
说明:找出两个字符串的最长公共字串。子串是连续的      
动态规划方案：使用dp[i][j]表示以x[i]和y[j]结尾的最长公共子串的长度，因为子串是连续的，所以，对于x[i]与y[j]来讲，它们要么与之前的公共子串构成新的公共子串；要么不构成。故状态转移方程为：     
（1） X[i]==Y[j], dp[i][j] = dp[i-1][j-1]+1;      
（2） X[i]!=Y[j], dp[i][j] = 0       
对于初始化，i==0或者j==0,如果x[i]=y[j],dp[i][j] = 1;否则dp[i][j]=0;     

核心代码：
     
    const string LCS(const string& str1,const string& str2)   //s1
	{
        int xlen=str1.size();
        int ylen=str2.size();
        int maxlen=0;
        int maxindex=0;
        vector<vector<int>> dp(xlen,vector<int>(ylen));
        for(int k=0;k<xlen;k++)
            for(int j=0;j<ylen;j++)
            {
                dp[k][j]=0;
            }
        int i=0;
        for(i=0;i<xlen;i++)
        {
            for(int j=0;j<ylen;j++)
            {
                if(str1[i]==str2[j])
                {
                    if(i&&j)
                    {
                         dp[i][j]=dp[i-1][j-1]+1;
                    }
                    if(i==0||j==0)
                    {
                        dp[i][j]=1;
                    }
                    if(dp[i][j]>maxlen)
                    {
                        maxlen = dp[i][j];
                        maxindex=i+1-maxlen;
                    }
                }
            }
        }
        string res=str1.substr(maxindex,maxlen);
        return res;
}

### 最长公共子序列
说明：相比于最长公共子串，差别在于公共子序列不要求数组中的元素连续。  
和公共子串类似，不再啰嗦，直接上代码。     
    
	int LCSseq(const string& str1,const string& str2)
	{
    ...
    vector<vector<int>> dp(xlen,vector<int>(ylen));
    for(int i=0;i<xlen;i++)
        for(int j=0;j<len;j++)
        {
            if(str[i]==str2[j])
            {   
                if(i==0 || j==0)
                    dp[i][j]=1; 
            }
            else
            {
                dp[i][j]=dp[i-1][j-1]+1;
            }
            else
            {
                if(i==0&&j==0)
                        continue;
                else if(i!=0&&j!=0)
                        dp[i][j]=max(dp[i-1][j],dp[i][j-1]);
                esle
                        dp[i][j]=1;
            }
        }	
		return dp[xlen-1][ylen-1];
	}

### 字符串编辑距离
说明：给定一个源字符串和目标字符串，能够对源串进行如下操作：   
(1)在给定位置上插入一个字符   
(2)替换任意字符   
(3)删除任意字符    
所以，字符串编辑距离，是指两个字符串之间，由一个转换成另一个所需的最少操作次数。    
动态规划方案：定义f[i,j]为子串str1[0...i]和str2[0...j]的最小编辑距离，则状态转移方程为：    
f[i,j] = Min(f[i-1,j]+1,f[i,j-1]+1,f[i-1,j-1]+(str1[i]==str2[j]?0:1))  

核心代码：

        int minDistance(string word1, string word2) 
		{
        int n1 = word1.size(), n2 = word2.size();
        int dp[n1 + 1][n2 + 1];
        for (int i = 0; i <= n1; ++i) dp[i][0] = i;
        for (int i = 0; i <= n2; ++i) dp[0][i] = i;
        for (int i = 1; i <= n1; ++i) 
		{
            for (int j = 1; j <= n2; ++j) 
			{
                if (word1[i - 1] == word2[j - 1]) 
				{
                    dp[i][j] = dp[i - 1][j - 1];
                } 
				else 
				{
                    dp[i][j] = min(dp[i - 1][j - 1], min(dp[i - 1][j], dp[i][j - 1])) + 1;
                }
            }
        }
        return dp[n1][n2];
    }


### 待续
这里通过几个典型实例简单说明了动态规划类问题的一些思路和方法。但是对于想很好的掌握动态规划，还是远远不够的。所以，还得多想，多领悟。 加油吧!