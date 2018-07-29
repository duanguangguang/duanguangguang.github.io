---
title: Union-Find算法
date: 2017-11-12 19:58:08
categories: 
  - 数据结构与算法
  - Algorithm
tags:
  - 算法
---

## 介绍

输入一系列整数对，其中每个整数都表示一个某种类型的对象，一对整数p和q可以理解成“p和q是相连的”。我们假设“相连”是一种等价关系，这意味着它具有：

- 自反性：p和p是相连的；
- 对称性：如果p和q是相连的，那么q和p也是相连的；
- 传递性：如果p和q是相连的且q和r是相连的，那么p和r也是相连的。

当程序从输入中读取了整数对p,q时，如果已知的所有整数对都不能说明p和q是相连的，则将这对整数写到输出，如果已知的数据可以说明p和q是相连的，则忽略这对整数对继续处理下一对整数对。要求程序能够判断给定的整数对是否相连。

我们程序限定为网络问题，所以使用网络方面的术语：将对象称为触点，将整数对称为连接，将等价类称为连通分量。

union-find算法的API：

|           public class UF           |                       |
| :---------------------------------: | --------------------- |
|              UF（int N）              | 以整数标识（0到N-1初始化N个触点）   |
|    void     union（int p, int q）     | 在p和q之间建立连接，合并分量       |
|         int     find（int p）         | p所在的分量的标识符            |
| boolean     connected（int p, int q） | 如果p和q存在于同一个分量中则返回true |
|           int     count（）           | 连通分量的数量               |

<!-- more -->

## Union-Find的实现

### 数据结构和算法设计

API已经说明触点和分量都会用int值表示，所以我们用一个以触点为索引的数组id[]作为基本数据结构来表示所有分量，所以每个分量都是由它的触点之一所表示。一开始我们有N个分量，每个触点都构成了一个只包含有它自己的分量，因此我们将id[i]的值初始化为i，其中i在0到N-1之中。

~~~java
public class UF{
	private int[] a;//分量id，以触点作为索引
  	private int count;//分量数量
  
  	public UF(int N){
      	//初始化分量id数组
      	count = N;
      	id = new int[N];
      	for(int i = 0; i < N; i++){
          	id[i] = i;
      	}
  	}
  
  	//返回给定触点所在的连通分量的标识符
  	public int find(int p){
      	
  	}
  
  	//将两个分量归并
  	public void union(int p, int q){
      	
  	}
  
  	//返回所有连通分量的数量
  	public int count(){
      	return count;
  	}
  
  	//判断两个触点是否存在于同一个分量之中
  	public boolean connected(int p, int q){
      	return find(p) == find(q);
  	}
  
  	public static void main(String[] args){
      	int N = StdIn.readInt();//读取触点数量
      	UF uf = new UF(N);//初始化N个分量
      	while(!StdIn.isEmpty()){
          	int p = StdIn.readInt();
          	int q = StdIn.readInt();//读取整数对
          	if(uf.connected(p,q)){
              	continue;//如果已经连通则忽略
          	}
          	uf.union(p,q);//归并分量
          	StdOut.println(p + " " + q);
      	}
      	StdOut.println(uf.count + "components");
  	}
}
~~~

这个实现维护了一个整形数组id[]，使得find（）方法对于处在同一个连通分量中的触点均返回相同的整数值，union（）方法来保证这一点。这样，问题就变成了实现find（）和union（）方法。

### Quick-Find算法

保证当且仅当id[p]等于id[q]时p和q是连通的。也就是，在同一个连通分量中的所有触点在id[]中的值必须全部相同。为了调用union（p，q）确保这一点，必须将两个集合中所有触点所对应的id[]元素变为同一个值。

~~~java
public int find(int p){
  	return id[p]
}

public void union(int p, int q){
  	//将p和q归并到相同的分量中
  	int pID = find(p);
  	int qID = find(q);
  
  	//如果p和q已经在相同的分量中，则忽略
  	if(pID == qID){
      	reutrn;
  	}
  
  	//将p的分量重命名为q的名称
  	for(int i = 0; i < id.length; i++){
      	if(id[i] == pID){
          	id[i] = qID
      	}
  	}
  	count--;
}
~~~

quick-find算法概述：

![](unionfind/quickfind01.png)

quick-find分析：

find（）操作速度很快，因为它只需要访问id[]数组一次。而归并两个分量的union（）操作访问数组的次数在(N+3)到(2N+1)之间。但quick-find算法一般无法处理大型问题，因为对于每一对输入union（）都需要扫描整个id[]数组。假设我们使用该算法解决动态连通性问题并且最后只得到一个连通分量，那么至少调用N-1次union（），即至少(N+3)(N-1)~N^2次数组访问。

### Quick-Union算法

该算法用来提高union（）方法的速度。每个触点所对应的id[]元素都是同一个分量中的另一个触点的名称（也可能是它自己），我们将这种联系称为链接。在实现find（）方法时，我们从给定的触点开始，由它的链接得到另一个触点，如此继续直到到达一个根触点，即链接指向自己的触点。当且仅当分别由两个触点开始的这个过程到达了同一个根触点时它们存在于同一个连通分量。需要union（p，q）保证这一点。它的实现就简单了：由p和q的链接分别找到它们的根触点，然后只需要将一个根触点链接到另一个即可将一个分量命名为另一个分量。

~~~java
private int find(int p){
  	//找出分量的名称
  	while(p != id[p]){//while循环中经过编译的代码对id[p]的第二次访问一般都不会访问数组
      	p = id[p]
  	}
  	return p;
}

public void union(int p, int q){
  	//将p和q的根节点统一
  	int pRoot = find(p);
  	int qRoot = find(q);
  
  	if(pRoot == qRoot){
      	return;
  	}
  
  	id[pRoot] = qRoot;
  	count--;
}
~~~

森林的表示：

![](unionfind/quickfind02.png)

用节点表示触点，用从一个节点到另一个节点的箭头表示链接，由此得到一个树的结构。在数组被初始化之后，每个节点的链接都指向自己，如果在某次union（）操作之前这条性质成立，那么操作之后也必然成立，因此find（）方法能够返回根节点所对应的触点的名称。这样，connected（）才能判断两个触点是否在同一树上。

quick-union算法分析：

quick-union算法可以看做是quick-find算法的一种改良，因为它解决了quick-find算法中最主要的问题，union（）操作总是线性级别的。**quick-union算法中的find（）方法访问数组的次数为1加上给定触点所对应的节点的深度的两倍。union（）和connected（）访问数组的次数为两次find（）操作（如果union（）中给定的两个触点分别存在于不同的树中则还需要加1）** 。这样我们得到quick-union算法最坏的情况：假设输入的整数对是有序的0-1、0-2、0-3等，N-1对之后N个触点将全部处于相同的集合之中，且由quick-union算法得到的树的高度为N-1，其中0链接到1,1链接到2...由上面黑体可知对于整数对0-i，union（）操作访问数组的次数为2i+1（触点为0的深度为i-1，触点i的深度为0）。因此处理N对整数所需要的所有find（）操作访问数组的总次数为3+5+7+...+（2N-1）~ N^2。

### 加权quick-union算法

对quick-union算法进行改进：之前union（）随意将一棵树连接到另一棵树上，现在我们会记录每一棵树的大小并总是将较小的树连接到较大的树上。加权quick-union:

![](unionfind/quickfind03.png)

这项改动需要添加一个数组和一些代码来记录树中的节点数，但它能大大改进算法的效率，称为**加权quick-union算法**。该算法构造的树的高度远远小于未加权的版本所构造的树的高度。

~~~java
public class WeightedQuickUnionUF {
	private int[] id;//父链接数组（由触点索引）
	private int[] sz;//各个根节点所对应的分量的大小
	private int count;//连通分量的数量
	
	public WeightedQuickUnionUF(int N){
		count = N;
		id = new int[N];
		sz = new int[N];
		for(int i = 0; i < N; i++){
			id[i] = i;
			sz[i] = 1;
		}
	}
	
	public int count(){
		return count;
	}
	
	public boolean connected(int p, int q){
		return find(p) == find(q);
	}
	
	public int find(int p){
		//跟随链接找到根节点
		while(p != id[p]){
			p = id[p];
		}
		return p;
	}
	
	public void union(int p, int q){
		int i = find(p);
		int j = find(q);
		if(i == j){
			return;
		}
		//将小树的根节点连接到大树的根节点
		if(sz[i] <sz[j]){
			id[i] = j;
			sz[j] += sz[i];
		}else{
			id[j] = i;
			sz[i] += sz[j];
		}
		count--;
	}
}
~~~

加权quick-union算法分析：

上面显示了加权quick-union算法的最坏情况。其中将要被归并的树的大小总是相等的（且总是2的幂）。这些树均含有2^n个节点，因此高度正好是n。当要归并两个含有2^n个节点的树时，得到的树含有2^n+1个节点，由此将树的高度增加到n+1。由此推广可以证明加权quick-union算法能够保证对数级别的性能。**对于N个触点，加权quick-union算法构造的森林中的任意节点的深度最多为lgN**。**对于加权quick-union算法和N个触点，在最坏情况下find（）、connected（）和union（）的成本的增长数量级为logN**。对于动态连通性问题，加权quick-union算法是三种算法中唯一可以用于解决大型实际问题的算法。

### 路径压缩的加权quick-union算法

路径压缩的加权quick-union算法是每个节点都直接链接到它的根节点上。要实现它只需要为find（）添加一个循环，将在路径上遇到的节点都直接链接到根节点上。这样得到的结果是几乎完全扁平化的树，它和quick-union算法理想情况下所得到的树非常接近。但实际上已经不大可能对加权quick-union算法进行任何改进。

## 总结

### union-find成本模型

数组的访问次数。

### union-find算法的性能特点

| 算法                     |      |         |        |
| ---------------------- | ---- | ------- | ------ |
| 存在N个触点时成本的增长数量级（最坏情况）  | 构造函数 | union（） | find（） |
| quick-find算法           | N    | N       | 1      |
| quick-union算法          | N    | 树的高度    | 树的高度   |
| 加权quick-union算法        | N    | lgN     | lgN    |
| 使用路径压缩的加权quick-union算法 | N    | 非常非常接近1 | 均摊成本   |
| 理想情况                   | N    | 1       | 1      |

### 算法中的增长数量级

| 描述     | 增长的数量级 | 典型的代码                      | 说明   | 举例      |
| ------ | ------ | -------------------------- | ---- | ------- |
| 常数级别   | 1      | a = b + c                  | 普通语句 | 将两个数相加  |
| 对数级别   | logN   | 二分查找                       | 二分策略 | 二分查找    |
| 线性级别   | N      | for(int i = 0; i < N; i++) | 循环   | 找出最大元素  |
| 线性对数级别 | NlogN  | 归并排序                       | 分治   | 归并排序    |
| 平方级别   | N^2    | 双层循环                       | 双层循环 | 检查所有元素对 |
| 立方级别   | N^3    | 三层循环                       | 三层循环 | 检查所有三元组 |
| 指数级别   | 2^N    | 穷举查找                       | 穷举查找 | 检查所有子集  |