---
title: 优先队列
date: 2017-11-16 19:35:54
categories: 
  - 数据结构与算法
  - Algorithm
tags:
  - 排序
---

## 介绍

**优先队列**是一种抽象数据类型，表示了一组值和对这些值的操作。优先队列最重要的操作就是**删除最大元素和插入元素**。泛型优先队列的API：

| public class MaxPQ<key extends Comparable<key>> |                    |
| :--------------------------------------: | ------------------ |
|                  MaxPQ                   | 创建一个优先队列           |
|              MaxPQ（int max）              | 创建一个初识容量为max的优先队列  |
|              MaxPQ（key[] a）              | 用a[]中的元素创建一个一个优先队列 |
|            void     insert（）             | 向优先队列中插入一个元素       |
|              key     max（）               | 返回最大元素             |
|             key     delMax（）             | 删除并返回最大元素          |
|          boolean     isEmpty（）           | 返回队列是否为空           |
|              int     size（）              | 返回优先队列中的元素个数       |

<!-- more -->

优先队列的调用示例：

问题：从输入的N个字符串中找出M个最大元素，前提是输入量非常巨大。

构造一个用数字作为键的优先队列，当优先队列的大小超过M时就删掉其中最小的元素。处理完所有交易。优先队列中存放的以降序排列的最大的M个交易，然后将该段代码放在一个栈中，遍历这个栈以颠倒它们顺序，从而将它们按降序打印出来。

~~~java
public class TopM {
	public static void main(String[] args) {
		//打印输入流中最大的M行
		int M = Integer.parseInt(args[0]);
		MinPQ<Transaction> pq = new MinPQ<Transaction>(M+1);

		while(StdIn.hasNextLine()){
			//为下一行输入创建一个元素并放入优先队列中
			pq.insert(new Transaction(StdIn.readLine()));
			if(pq.size() > M){
				pq.delMin();//如果队列中存在M+1个元素则删除其中最小的元素。
			}
		}//最大的M个元素都在优先队列中
		Stack<Transaction> stack = new Stack<Transaction>();
		while(!pq.isEmpty()){
			stack.push(pq.delMin());
		}
		for(Transaction t : stack){
			System.out.println(t);
		}
	}
}
~~~

## 实现

下面使用有序和无序的数组以及链表实现优先队列。

### 数组实现（无序）

insert（）方法的代码和栈的push（）方法一样。实现删除最大元素，添加一段类似选择排序的内循环的代码，将最大元素和边界元素交换然后删除它，和栈的pop（）方法实现一样。同样，可以添加调整数组大小的代码来保证数据结构中至少含有四分之一的元素而永远不会溢出。（详见[栈](https://duanguangguang.github.io/2017/11/08/algorithm/stack/#more)）。

~~~java
/**
 * 无序数组实现优先队列
 * @author guangguang_duan
 *
 */
public class UnorderedArrayMaxPQ<Key extends Comparable<Key>> {

	private Key[] pq; 
	private int n;
	
	public UnorderedArrayMaxPQ(int capacity){
		pq = (Key[]) new Comparable[capacity];
		n = 0;
	}
	
	public boolean isEmpty(){ 
		return n == 0; 
	}
	
    public int size(){
    	return n;
    }
    
    public void insert(Key x){
    	pq[n++] = x;
    }
    
    public Key delMax(){
    	int max = 0;
        for (int i = 1; i < n; i++){
        	if (less(max, i)){
        		max = i;
        	}
        }
        exch(max, n-1);

        return pq[--n];
    }
    
    /***************************************************************************
     * Helper functions
     ***************************************************************************/
     private boolean less(int i, int j) {
         return pq[i].compareTo(pq[j]) < 0;
     }

     private void exch(int i, int j) {
         Key swap = pq[i];
         pq[i] = pq[j];
         pq[j] = swap;
     }
    
     /***************************************************************************
      * Test routine(测试程序,routine-常规)
      ***************************************************************************/
      public static void main(String[] args) {
          UnorderedArrayMaxPQ<String> pq = new UnorderedArrayMaxPQ<String>(10);
          pq.insert("this");
          pq.insert("is");
          pq.insert("a");
          pq.insert("test");
          while (!pq.isEmpty()) 
              StdOut.println(pq.delMax());
      }

}
~~~

### 数组实现（有序）

在insert（）方法中添加代码，将所有较大的元素向右边移动一格以使数组保持有序（和[插入排序](https://duanguangguang.github.io/2017/11/14/algorithm/selsort/#more)一样）。这样，最大的元素总会在数组的一边，优先队列的删除最大元素操作就和栈的pop（）操作一样。

~~~java
/**
 * 有序数组实现优先队列
 * @author guangguang_duan
 *
 */
public class OrderedArrayMaxPQ<Key extends Comparable<Key>> {
    private Key[] pq;          
    private int n;             

    public OrderedArrayMaxPQ(int capacity) {
        pq = (Key[]) (new Comparable[capacity]);
        n = 0;
    }


    public boolean isEmpty(){ 
    	return n == 0;
    }
    
    public int size(){ 
    	return n;
    } 
    
    public Key delMax(){ 
    	return pq[--n];
    }

    public void insert(Key key) {
        int i = n-1;
        while (i >= 0 && less(key, pq[i])) {
            pq[i+1] = pq[i];
            i--;
        }
        pq[i+1] = key;
        n++;
    }

   /***************************************************************************
    * Helper functions
    ***************************************************************************/
    private boolean less(Key v, Key w) {
        return v.compareTo(w) < 0;
    }

   /***************************************************************************
    * Test routine.
    ***************************************************************************/
    public static void main(String[] args) {
        OrderedArrayMaxPQ<String> pq = new OrderedArrayMaxPQ<String>(10);
        pq.insert("this");
        pq.insert("is");
        pq.insert("a");
        pq.insert("test");
        while (!pq.isEmpty())
            StdOut.println(pq.delMax());
    }
}
~~~

### 链表实现

可以用基于链表的下压栈的代码实现，然后可以选择修改pop（）来找到并返回最大元素，或是修改push（）来保证所有元素为逆序并与pop（）来删除并返回链表的首元素（最大元素）。



## 总结

使用无序序列实现优先队列是惰性方法，使用有序序列实现优先队列是积极方法。

实现栈或是队列与实现优先队列的最大不同在于对性能的要求。对于栈和队列，我们的实现能在常数时间内完成所有操作；而对于优先队列，上面的初级实现，插入元素和删除最大元素这两个操作之一在最坏情况下需要线性时间完成。

优先队列的各种实现在最坏情况下运行时间的增长数量级：

| 数据结构 | 插入元素 | 删除最大元素 |
| ---- | ---- | ------ |
| 有序数组 | N    | 1      |
| 无序数组 | 1    | N      |
| 堆    | logN | logN   |
| 理想情况 | 1    | 1      |