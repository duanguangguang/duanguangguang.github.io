---
title: 索引优先队列
date: 2017-11-18 17:52:53
categories: 
  - 数据结构与算法
  - Algorithm
tags:
  - 排序
---

## 介绍

在很多应用中，允许用例引用已经进入优先队列中的元素是必要的。做到这一点的一种简单方法是给每个元素一个索引。另外一种常见的情况是用例已经有了总量为N的多个元素，而且可能还同时使用了多个（平行）数组来存储这些元素的信息。此时，其他无关的用例代码可能已经在使用一个整数索引来引用这些元素了。

<!-- more -->

关联索引的泛型优先队列的API：

| public class IndexMinPQ<Item extends Comparable<Item>> |                    |
| :--------------------------------------: | ------------------ |
|           IndexMinPQ(int maxN)           | 创建一个最大容量为maxN的优先队列 |
|    void     insert(int k, Item item)     | 插入一个元素，将它和索引k相关联   |
|    void     change(int k, Item item)     | 将索引为k的元素设为item     |
|       boolean     contains(int k)        | 是否存在索引为k的元素        |
|          void     delete(int k)          | 删去索引k及其相关联的元素      |
|            Item     minKey()             | 返回最小元素             |
|            int     minIndex()            | 返回最小元素的索引          |
|             int     delMin()             | 删除最小元素并返回它的索引      |
|          boolean     isEmpty()           | 优先队列是否为空           |
|              int     size()              | 优先队列的元素数量          |

这种数据结构可以理解成：它能够快速访问数组的一个特定子集中的最小元素。换句话说，可以将pq的IndexMinPQ类优先队列看做数组pq[0...N-1]中的一部分元素的代表。

## 实现

~~~java
public class IndexMinPQ<Key extends Comparable<Key>> implements Iterable<Integer> {
    private int maxN;        // maximum number of elements on PQ
    private int n;           // number of elements on PQ
    private int[] pq;        // binary heap using 1-based indexing
    private int[] qp;        // inverse of pq - qp[pq[i]] = pq[qp[i]] = i
    private Key[] keys;      // keys[i] = priority of i

    public IndexMinPQ(int maxN) {
        if (maxN < 0) throw new IllegalArgumentException();
        this.maxN = maxN;
        n = 0;
        keys = (Key[]) new Comparable[maxN + 1];    // make this of length maxN??
        pq   = new int[maxN + 1];
        qp   = new int[maxN + 1];                   // make this of length maxN??
        for (int i = 0; i <= maxN; i++)
            qp[i] = -1;
    }
  
    public boolean isEmpty() {
        return n == 0;
    }
   
    public boolean contains(int i) {
        if (i < 0 || i >= maxN) throw new IllegalArgumentException();
        return qp[i] != -1;
    }
  
    public int size() {
        return n;
    }

    public void insert(int i, Key key) {
        if (i < 0 || i >= maxN) throw new IllegalArgumentException();
        if (contains(i)) throw new IllegalArgumentException("index is already in the priority queue");
        n++;
        qp[i] = n;
        pq[n] = i;
        keys[i] = key;
        swim(n);
    }

    public int delMin() {
        if (n == 0) throw new NoSuchElementException("Priority queue underflow");
        int min = pq[1];
        exch(1, n--);
        sink(1);
        assert min == pq[n+1];
        qp[min] = -1;        // delete
        keys[min] = null;    // to help with garbage collection
        pq[n+1] = -1;        // not needed
        return min;
    }
  
    public void changeKey(int i, Key key) {
        if (i < 0 || i >= maxN) throw new IllegalArgumentException();
        if (!contains(i)) throw new NoSuchElementException("index is not in the priority queue");
        keys[i] = key;
        swim(qp[i]);
        sink(qp[i]);
    }
  
    public void change(int i, Key key) {
        changeKey(i, key);
    }

    public void delete(int i) {
        if (i < 0 || i >= maxN) throw new IllegalArgumentException();
        if (!contains(i)) throw new NoSuchElementException("index is not in the priority queue");
        int index = qp[i];
        exch(index, n--);
        swim(index);
        sink(index);
        keys[i] = null;
        qp[i] = -1;
    }

  	public int minIndex() {
        if (n == 0) throw new NoSuchElementException("Priority queue underflow");
        return pq[1];
    }

    public Key minKey() {
        if (n == 0) throw new NoSuchElementException("Priority queue underflow");
        return keys[pq[1]];
    }
  
    private boolean greater(int i, int j) {
        return keys[pq[i]].compareTo(keys[pq[j]]) > 0;
    }

    private void exch(int i, int j) {
        int swap = pq[i];
        pq[i] = pq[j];
        pq[j] = swap;
        qp[pq[i]] = i;
        qp[pq[j]] = j;
    }

    private void swim(int k) {
        while (k > 1 && greater(k/2, k)) {
            exch(k, k/2);
            k = k/2;
        }
    }

    private void sink(int k) {
        while (2*k <= n) {
            int j = 2*k;
            if (j < n && greater(j, j+1)) j++;
            if (!greater(k, j)) break;
            exch(k, j);
            k = j;
        }
    }

    public Iterator<Integer> iterator() { return new HeapIterator(); }

    private class HeapIterator implements Iterator<Integer> {
        // create a new pq
        private IndexMinPQ<Key> copy;

        // add all elements to copy of heap
        // takes linear time since already in heap order so no keys move
        public HeapIterator() {
            copy = new IndexMinPQ<Key>(pq.length - 1);
            for (int i = 1; i <= n; i++)
                copy.insert(pq[i], keys[pq[i]]);
        }

        public boolean hasNext()  { return !copy.isEmpty();                     }
        public void remove()      { throw new UnsupportedOperationException();  }

        public Integer next() {
            if (!hasNext()) throw new NoSuchElementException();
            return copy.delMin();
        }
    }
  
    public static void main(String[] args) {
        // insert a bunch of strings
        String[] strings = { "it", "was", "the", "best", "of", "times", "it", "was", "the", "worst" };

        IndexMinPQ<String> pq = new IndexMinPQ<String>(strings.length);
        for (int i = 0; i < strings.length; i++) {
            pq.insert(i, strings[i]);
        }

        // delete and print each key
        while (!pq.isEmpty()) {
            int i = pq.delMin();
            StdOut.println(i + " " + strings[i]);
        }
        StdOut.println();

        // reinsert the same strings
        for (int i = 0; i < strings.length; i++) {
            pq.insert(i, strings[i]);
        }

        // print each key using the iterator
        for (int i : pq) {
            StdOut.println(i + " " + strings[i]);
        }
        while (!pq.isEmpty()) {
            pq.delMin();
        }
    }
}
~~~

含有N个元素的基于堆的索引优先队列所有操作在最坏情况下的成本：

|     操作     | 比较次数的增长数量级 |
| :--------: | :--------: |
|  insert()  |    logN    |
|  change()  |    logN    |
| contains() |     1      |
|  delete()  |    logN    |
|  minKey()  |     1      |
| minIndex() |     1      |
|  delMin()  |    logN    |

## 使用索引优先队列的多向归并

将多个有序的输入流归并成一个有序的输出流。

~~~java
public class Multiway { 
    private static void merge(In[] streams) {
        int n = streams.length;
        IndexMinPQ<String> pq = new IndexMinPQ<String>(n);
        for (int i = 0; i < n; i++) {
          	if (!streams[i].isEmpty()) {
              	pq.insert(i, streams[i].readString());
            }  
        }
                 
        while (!pq.isEmpty()) {
            StdOut.print(pq.minKey() + " ");
            int i = pq.delMin();
            if (!streams[i].isEmpty()) {
              	pq.insert(i, streams[i].readString());
            }   
        }
        StdOut.println();
    }

    public static void main(String[] args) {
        int n = args.length;
        In[] streams = new In[n];
        for (int i = 0; i < n; i++) {
           streams[i] = new In(args[i]);
        }
        merge(streams);
    }
}
~~~

