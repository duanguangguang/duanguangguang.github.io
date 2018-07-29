---
title: 栈（Stack）
date: 2017-11-08 19:34:01
categories: 
  - 数据结构与算法
  - 数据结构
tags:
  - 数据结构
---

## 介绍

**栈**是一种基于**后进先出（LIFO）**策略的集合类型。在应用程序中使用栈迭代器的一个典型原因是在用集合保存元素的同时颠倒它们的顺序。栈的一个典型用例：把标准输入的所有整数逆序排列。

~~~java
public class StackOfType {

	public static void main(String[] args) {
		Stack<Integer> stack = new Stack<Integer>();
		while(!StdIn.isEmpty()){
			stack.push(StdIn.readInt());
		}
		
		for(int i : stack){
			System.out.println(i);
		}
	}
}
~~~

<!-- more -->

## API

泛型可迭代的栈API：

| public class Stack<Item> implements Iterable<Item> |           |
| :--------------------------------------: | --------- |
|                 Stack()                  | 创建一个空栈    |
|         void     push(Item item)         | 添加一个元素    |
|              Item     pop()              | 删除最近添加的元素 |
|          boolean     isEmpty()           | 栈是否为空     |
|              int     size()              | 栈中元素数量    |

## Dijkstra的双栈算术表达式求值算法

求表达式：（1 + （（ 2 + 3 ）* （ 4 * 5 ）））。用两个栈，一个用于保存运算符，一个用于保存操作数实现。

~~~java
public class Evaluate{
  	public static void main(String[] args){
      	Stack<String> ops = new Stack<String>();
      	Stack<Double> vals = new Stack<Double>();
      	while(!StdIn.isEmpty()){
          	//读取字符，如果是运算符则压入运算符栈
          	String s = StdIn.readString();
          	if(s.equals("(")){
              	
          	}else if(s.equals("+")){
              	ops.push(s);
          	}else if(s.equals("-")){
              	ops.push(s);
          	}else if(s.equals("*")){
              	ops.push(s);
          	}else if(s.equals("/")){
              	ops.push(s);
          	}else if(s.equals("sqrt")){
              	ops.push(s);
          	}else if(s.equals(")")){
              	//如果字符为")"，弹出运算符和操作数，计算结果并压入栈
              	String op = ops.pop();
              	double v = vals.pop();
              
              	if(op.equals("+")){
                  	v = vals.pop() + v;
              	}else if(op.equals("-")){
                  	v = vals.pop() - v;
              	}else if(op.equals("*")){
                  	v = vals.pop() * v;
              	}else if(op.equals("/")){
                  	v = vals.pop() / v;
              	}else if(op.equals("sqrt")){
                  	v = Math.sqrt(v);
              	}
              
              	vals.push(v);
          	}else{
              	//如果字符既不是运算符，也不是括号，则将它作为double值压入操作数栈
              	vals.push(Double.parseDouble(s));
          	}
      	}
      	system.out.println(vals.pop());
  	}
}
~~~

表达式由括号、运算符和操作数组成，所以上面代码的处理过程是：

- 将操作数压入操作数栈；
- 将运算符压入运算符栈；
- 忽略左括号；
- 在遇到右括号时，弹出一个运算符弹出所需数量的操作数并将运算符和操作数的运算结果压入操作数栈。

## 定容栈

一种表示容量固定的的字符串栈的抽象数据类型。该定容栈的实现：

~~~java
public class FixedCapacityStackOfString{
	private String[] a;//保存栈中的元素的数组
  	private int N; //栈中元素的数量
  	public FixedCapacityStackOfString(int cap){
      	a = new String[cap];
  	}
  
  	public boolean isEmpty(){
      	return N == 0;
  	}
  
  	public int size(){
      	return N;
  	}
  
  	public void push(String item){
      	a[N++] = item;
  	}
  
  	public String pop(){
      	return a[--N];
  	}
}
~~~

该栈的第一个缺点是只能处理String对象，而**java不允许创建泛型数组**，为了解决这个问题，下面实现一个泛型栈

~~~java
public class FixedCapacityStack<Item>{
	private Item[] a;//保存栈中的元素的数组
  	private int N; //栈中元素的数量
  	public FixedCapacityStackOfString(int cap){
      	a = (Item[])new Object[cap];
  	}
  
  	public boolean isEmpty(){
      	return N == 0;
  	}
  
  	public int size(){
      	return N;
  	}
  
  	public void push(Item item){
      	a[N++] = item;
  	}
  
  	public Item pop(){
      	return a[--N];
  	}
}
~~~

选择用数组表示栈内容，因为数组一旦创建，其大小是无法改变的，所以栈使用的空间只能是这个数组大小的一部分。为了能够使栈保存所有元素，又不浪费空间，我们需要一个能动态调整栈大小，并支持迭代的栈。

## 动态调整大小的栈实现

能够动态调整数组大小，并支持迭代的栈的实现：

~~~java
import java.util.Iterator;
public class ResizingArrayStack<Item> implements Iterable<Item>{
  	private Item[] a = (Item[]) new Object[1];//栈元素，初始数量为1
  	private int N = 0;
  	
    public boolean isEmpty(){
         return N == 0;
    }

    public int size(){
         return N;
    }
  
  	private void resize(int max){
      	//将栈移动到一个大小为max的新数组
      	Item[] temp = (Item[]) new Object[max];
      	for(int i = 0; i < N; i++){
          	temp[i] = a[i]
      	}
      	a = temp;
  	}
  
  	public void push(Item item){
      	//将元素添加到栈顶
      	if(N == a.length){ //当没有多余空间，将数组长度加倍
          	resize(2*a.length);
      	}
      	a[N++] = item;
  	}
  
  	public Item pop(){
    	//从栈顶删除元素
      	Item item = a[--N];
      	a[N] = null;//避免对象游离
      	//调减的条件是：栈的大小是否小于数组的四分之一，这样在数组长度被减半之后，它的状态为半满，
        //在下次调整数组大小前仍然能够进行多次push()和pop()操作
      	if(N > 0 && N == a.length/4){
          	resize(a.length/2);
      	}
      	return item;
    }
  	
  	public Iterator<Item> iterator(){
      	return new ReverseArrayIterator();
  	}
  
  	private class ReverseArrayIterator implements Iterator<Item>{
      	//支持后进先出的迭代
      	private int i = N;
      	public boolean hasNext(){
          	return i > 0;
      	}
      
      	public Item next(){
          	return a[--i];
      	}
      
      	public void remove(){
          	//避免在迭代中穿插能够修改数据结构的操作
      	}
  	}
}
~~~

这份泛型的可迭代的Stack API的实现是所有集合类抽象数据类型实现的模板，它将所有元素保存在数组中，支持按照后进先出的顺序迭代访问所有栈元素，并动态调整数组的大小以保持数组大小和栈的大小之比小于一个常数。该实现达到了任意集合类数据类型的实现的最佳性能：

- 每项操作的用时都与集合大小无关；
- 空间需求总是不超过集合大小乘以一个常数
- 缺点是push()和pop()操作可能会调整数组的大小，这项操作的耗时和栈的大小成正比。

### 对象游离

java的垃圾收集策略是回收所有无法被访问的对象的内存。在上面对pop()的实现中，被弹出的元素的引用仍然存在于数组中，这个元素实际上已经不会再被访问了，但java的垃圾收集器没法知道这一点，除非改引用被覆盖。这种保存一个不需要的对象的引用称为游离。

避免游离只需将被弹出的数组元素的值设为null，这将覆盖无用的引用并使系统可以在用例使用完被弹出的元素后回收它的内存。

### 迭代

~~~java
Iterator<String> i = collection.iterator();
while(i.hasNext()){
  	String s = i.next();
  	system.out.println(s);
}
~~~

这段代码展示了任意可迭代的集合数据类型中需要实现的东西：

- 集合数据类型必须实现一个iterator()方法并返回一个iterator对象；
- Iterstor类必须包含两个方法：hasNext()和next()。

要是一个类可迭代，第一步就是在它的声明中加入：implements Iterable<Item>，对应的接口是：

~~~java
public interface Iterable<Item>{
  	Iterator<Item> iterator();
}
~~~

然后在类中添加一个方法iterator()并返回一个迭代器Iterator<Item>，上面的实现：

~~~java
public Iterator<Item> iterator(){
      	return new ReverseArrayIterator();
}
~~~

迭代器是一个实现了hasNext()和next()方法的类的对象，由Iterator接口定义：

~~~java
public interface Iterator<Item>{
  	boolean hasNext();
  	Item next();
  	void remove();
}
~~~

对于上面的迭代器，它的实现在栈的一个嵌套类中：

~~~java
private class ReverseArrayIterator implements Iterator<Item>{
    private int i = N;
    public boolean hasNext(){
      return i > 0;
    }

    public Item next(){
      return a[--i];
    }

    public void remove(){
    }
}
~~~

