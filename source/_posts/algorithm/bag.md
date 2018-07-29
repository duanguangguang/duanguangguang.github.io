---
title: 背包（Bag）
date: 2017-11-08 19:33:32
categories: 
  - 数据结构与算法
  - 数据结构
tags:
  - 数据结构
---

## 介绍

许多基础数据类型都和对象的集合有关，具体来说，数据类型的值就是一组对象的集合，所有操作都是关于添加、删除或是访问集合中的对象。背包、队列和栈这三种数据类型，不同之处在于删除或者访问对象的顺序不同。

**背包**是一种不支持从中删除元素的集合数据类型，它的目的就是帮助用例收集元素并迭代遍历所有收集到的元素，迭代的顺序不确定切与用例无关。背包的典型用例：

<!-- more -->

计算标准输入中的所有double值的平均值和样本标准差。样本标准差为每个值和平均值之差的平方之和除以N-1之后的平方根。

~~~java
public class BagOfType {
	public static void main(String[] args) {
		Bag<Double> numbers = new Bag<Double>();
		
		while(!StdIn.isEmpty()){
			numbers.add(StdIn.readDouble());
		}
		
		int N = numbers.size();
		double sum = 0.0;
		for(double x : numbers){
			sum += x;
		}
		
		double mean = sum/N;
		sum = 0.0;
		for(double x : numbers){
			sum += (x-mean)*(x-mean);
		}
		
		double std = Math.sqrt(sum/(N-1));
		
		System.out.println("Mean: " + mean);
		System.out.println("Std: " + mean);
	}
}

~~~

## API

泛型可迭代的背包API：

| public class Bag<Item> implements Iterable<Item> |          |
| :--------------------------------------: | -------- |
|                  Bag()                   | 创建一个空背包  |
|         void     add(Item item)          | 添加一个元素   |
|          boolean     isEmpty()           | 背包是否为空   |
|              int     size()              | 背包中的元素数量 |

## 实现

[网站](algs4.cs.princeton.edu)上关于背包的一个实现：用数组实现的可动态调整数组大小的背包实现，后面学习链表会给出一个链表实现的背包。

~~~java
public class ResizingArrayBag<Item> implements Iterable<Item> {
    private Item[] a;         // array of items
    private int n;            // number of elements on bag

    public ResizingArrayBag() {
        a = (Item[]) new Object[2];
        n = 0;
    }

    public boolean isEmpty() {
        return n == 0;
    }

    public int size() {
        return n;
    }

    private void resize(int capacity) {
        assert capacity >= n;
        Item[] temp = (Item[]) new Object[capacity];
        for (int i = 0; i < n; i++)
            temp[i] = a[i];
        a = temp;
    }

    public void add(Item item) {
        if (n == a.length) resize(2*a.length);    // double size of array if necessary
        a[n++] = item;                            // add item
    }


    public Iterator<Item> iterator() {
        return new ArrayIterator();
    }

    private class ArrayIterator implements Iterator<Item> {
        private int i = 0;
        public boolean hasNext()  { return i < n;                               }
        public void remove()      { throw new UnsupportedOperationException();  }

        public Item next() {
            if (!hasNext()) throw new NoSuchElementException();
            return a[i++];
        }
    }

    public static void main(String[] args) {
        ResizingArrayBag<String> bag = new ResizingArrayBag<String>();
        bag.add("Hello");
        bag.add("World");
        bag.add("how");
        bag.add("are");
        bag.add("you");

        for (String s : bag)
            StdOut.println(s);
    }

}
~~~

