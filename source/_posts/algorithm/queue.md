---
title: 队列（Queue）
date: 2017-11-08 19:33:48
categories: 
  - 数据结构与算法
  - 数据结构
tags:
  - 数据结构
---

## 介绍

**队列**是一种基于**先进先出（FIFO）**策略的集合类型。按照任务产生的顺序来完成它们的策略。在应用程序中使用队列的主要原因是在用集合保存元素的同时保存它们的相对顺序：入列顺序和出列顺序相同。队列的一个典型用例：

<!-- more -->

~~~java
public class QueueOfTyoe {

	public static int[] readInts(String name) {
		StdIn in = new StdIn(name);
		Queue<Integer> q = new Queue<Integer>();
		
		while(!in.isEmpty()){
			q.enqueue(in.readInt());
		}
		
		int N = q.size();
		int[] a = new int[N];
		
		for(int i=0; i<N; i++){
			a[i] = q.dequeue();
		}
		return a;
	}

}

~~~

## API

泛型可迭代的队列API：

| public class Queue<Item> implements Iterable<Item> |           |
| :--------------------------------------: | --------- |
|                 Queue()                  | 创建空队列     |
|       void     enqueue(Item item)        | 添加一个元素    |
|            Item     dequeue()            | 删除最早添加的元素 |
|            boolean isEmpty()             | 队列是否为空    |
|              int     size()              | 队列中的元素数量  |

## 实现

[网站]()上关于队列的一个实现：动态可调整数组大小。

~~~java
public class ResizingArrayQueue<Item> implements Iterable<Item> {
    private Item[] q;       // queue elements
    private int n;          // number of elements on queue
    private int first;      // index of first element of queue
    private int last;       // index of next available slot


    public ResizingArrayQueue() {
        q = (Item[]) new Object[2];
        n = 0;
        first = 0;
        last = 0;
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
        for (int i = 0; i < n; i++) {
            temp[i] = q[(first + i) % q.length];
        }
        q = temp;
        first = 0;
        last  = n;
    }

    public void enqueue(Item item) {
        // double size of array if necessary and recopy to front of array
        if (n == q.length) resize(2*q.length);   // double size of array if necessary
        q[last++] = item;                        // add item
        if (last == q.length) last = 0;          // wrap-around
        n++;
    }

    public Item dequeue() {
        if (isEmpty()) throw new NoSuchElementException("Queue underflow");
        Item item = q[first];
        q[first] = null;                            // to avoid loitering
        n--;
        first++;
        if (first == q.length) first = 0;           // wrap-around
        // shrink size of array if necessary
        if (n > 0 && n == q.length/4) resize(q.length/2); 
        return item;
    }

    public Item peek() {
        if (isEmpty()) throw new NoSuchElementException("Queue underflow");
        return q[first];
    }


    /**
     * Returns an iterator that iterates over the items in this queue in FIFO order.
     * @return an iterator that iterates over the items in this queue in FIFO order
     */
    public Iterator<Item> iterator() {
        return new ArrayIterator();
    }

    // an iterator, doesn't implement remove() since it's optional
    private class ArrayIterator implements Iterator<Item> {
        private int i = 0;
        public boolean hasNext()  { return i < n;                               }
        public void remove()      { throw new UnsupportedOperationException();  }

        public Item next() {
            if (!hasNext()) throw new NoSuchElementException();
            Item item = q[(i + first) % q.length];
            i++;
            return item;
        }
    }

    public static void main(String[] args) {
        ResizingArrayQueue<String> queue = new ResizingArrayQueue<String>();
        while (!StdIn.isEmpty()) {
            String item = StdIn.readString();
            if (!item.equals("-")) queue.enqueue(item);
            else if (!queue.isEmpty()) StdOut.print(queue.dequeue() + " ");
        }
        StdOut.println("(" + queue.size() + " left on queue)");
    }

}
~~~

