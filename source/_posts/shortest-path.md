---
title: 最短路问题
date: 2016-11-20 20:03:09
tags: 数据结构与算法
---

最短路问题是图论中最基础的问题。最短路是给定两个顶点，在以这两个点为起点和终点的路径中，边的权值和最小的路径。如果把权值当做距离，考虑最短距离就很容易理解了。    
<!--more-->
### 单源最短路问题
单源最短路问题是固定一个起点，求它到其他所有点的最短路的问题。如果终点也固定，这样的问题叫做两点之间的最短路问题。        
#### Bellman-Ford算法
记从起点s出发到顶点i的最短距离为d[i]。则下列等式成立：  
 ![](http://i.imgur.com/sZLfLxF.png)    
如果给定的图是一个有向无环图，可以按拓扑排序给顶点编号，并利用这条递推关系式计算出d。但是，如果图中有圈，就无法依赖这样的顺序进行排序。这种情况下，记当前到顶点i的最短路长度为d[i]，并设初值d[s]=0，d[i]=INF，再不断使用这条关系式更新d的值，就可以算出新的d。只要图中不存在负圈，这样的更新操作是有限的。结束之后的d就是所求的最短距离。

	struct edge {
		int from, to;
		int cost;
	}

	edge es[MAX_E];		//边
	
	int d[MAX_V];		//最短距离
	int V, E;			//顶点数和边数

	void shortest_path(int s) {
		for(int i=0;i<V;i++){
			d[i]=INF;	//初始化为INF
		}
		d[s]=0;
		while(true){
			bool update =false;
			for(int i=0;i<E;i++){
				edge e = es[i];		//边
				if(d[e.from]!=INF && d[e.to]>d[e.from]+e.cost){
					d[e.to] = d[e.from]+cost;	
					update = true;
				}
			}
			if(!update) break;
		}
	}
该算法叫做Bellman-Ford算法。如果在图中不存在从s可达的负圈，那么最短路不会经过同一顶点两次（即最多通过|V-1|条边），while循环最多执行|V|-1次。因此，复杂度是O(|V|x|E|)。反之，如果存在从s到达的负圈，那么在第|V|次循环中也会更新d的值，因此可以利用这个性质来检查负圈。

#### Dijkstra算法
考虑一下Bellman-Ford算法没有负边的情况。如果d[i]还不是最短距离的话，那么即使进行d[j]=d[i]+(从i到j的边的权值)的更新，d[j]也不会变成最短距离。而且，即使d[i]没有变化，每一次循环也要检查一遍从i出发的所有边，这是很浪费时间的。因此可以对算法做如下修改：     
（1）找到最短距离已经确定的顶点，从它出发更新相邻的最短距离；    
（2）此后不需要再关心（1）中“最短距离已经确定的顶点”。     
得到上述提到的“最短距离已经确定的顶点“是问题的关键。在最开始时，只有起点的最短距离是确定的。而在尚未使用过的顶点中，距离d[i]最小的顶点就是最短距离已经确定的顶点。因为不存在负边，所以d[i]不会在后面的更新中变小。该算法称为Dijkstra算法。

	int cost[MAX_V][MAX_V];		//cost[u][v]表示边e=(u,v)的权值
	int d[MAX_V];				//顶点s出发的最短距离
	bool used[MAX_V];			//已经使用过的图
	int V;

	void dijkstra(int s){
		fill(d, d+V, INF);		//初始距离设为无穷大
		d[s]=0;

		while(true){
			int v=-1;
			for(int u=0;u<V;u++){
				if(!used[u] && (v==-1 || d[u]<d[v])) v=u;	//从尚未使过的顶点中选择一个距离最小的顶点
			}
			if(v==-1) break;
			used[v]=true;
			for(int u=0;u<V;u++){
				d[u] = min(d[u], d[v]+cost[v][u]);
			}
		}
	}
使用邻接矩阵实现的Dijkstra算法的复杂度是O(|V|^2)。使用邻接表的话，更新最短距离只需要访问每条边一次即可，因此这部分的复杂度是O(|E|)。但是每次要枚举所有的顶点来查找下一个使用的顶点，因此最终复杂度还是O(|V|^2)。在|E|比较小时，大部分时间花在了查找下一个使用的顶点上，因此需要使用合适的数据结构对其进行优化。      
使用堆就可以了，把每个顶点当前的最短距离用堆维护，在更新最短距离时，把对应的元素往根的方向移动以满足堆的性质。而每次从堆中取出的最小值就是下一次要使用的顶点。这样堆中元素共有O(|V|)个，更新和取出数值的操作有O(|E|)次，因此整个算法的复杂度是O(|E|log|V|)。

	struct edge {
		int to, cost;	
	}
	
	typedef pair<int, int> P;		//first是最短距离，second是顶点的编号
		
	int V;
	vector<edge> G[MAX_V];
	
	int d[MAX_V];

	void dijkstra(int s) {
		priority_queue<P, vector<P>, greater<P> > que;
		fill(d, d+V, INF);
		d[s]=0;
		que.push(P(0, s));

		while(!que.empty()){
			P p=que.top();
			que.pop();
			int v=p.second;		//顶点
			if(d[v]<p.first) continue;
			for(int i=0;i<G[v].size();i++){		//找距离最短的邻点
				edge e=G[v][i];
				if(d[e.to]>d[v]+e.cost){			//更新
					d[e.to]=d[v]+e.cost;
					que.push(P(d[e.to], e.to));		//把最近的节点和距离放入堆中
				}
			}	
			
		}
	}
### 任意两点间的最短路问题

求解所有两点间的最短路问题叫做任意两点间的最短路问题，可以尝试使用动态规划。只使用顶点0~k和i,j的情况下，记i到j的最短路长度为d[k+1][i][j]。k=-1时，认为只使用i和j，所以d[0][i][j]=cost[i][j]。接下来把只使用顶点0~k的问题归约到只使用0~k-1的问题上。    
只使用0~k时，我们分i到j的最短路正好经过顶点k一次和完全不经过顶点k两种情况来讨论。不经过顶点k的情况，d[k][i][j]=d[k-1][i][j]。通过顶点k的情况下，d[k][i][j]=d[k-1][i][k]+d[k-1][k][j]。合起来，就得到了d[k][i][j]=min(d[k-1][i][j], d[k-1][i][k]+d[k-1][k][j])。这个DP也可以使用同一个数组，不断进行d[i][j]=d[i][k]+d[k][j]的更新来实现。  
该算法叫做Floyd-Warshall算法，可以在O(|V|^3)时间里求得所有两点间的最短路长度。该算法可以处理边是负数的情况。而判断图中是否有负圈，只需检查是否存在d[i][i]是负数的顶点i即可。

	int d[MAX_V][MAX_V];	//边不存在为INF，d[i][i]=0
	int V;
	
	void warshall_floyd() {
		for(int k=0;k<V;k++){
			for(int i=0;i<V;i++){
				for(int j=0;j<V;j++){
					dp[i][j]=min(d[i][j], d[i][k]+d[k][j]);
				}
			}
		}
	}
这样通过三重循环非常简单地就可以求出所有两点间的最短路长度。实现起来简单，复杂度也在可以承受的范围之内。