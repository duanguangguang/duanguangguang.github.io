---
title: 求两个正整数的最大公约数
date: 2017-10-28 16:56:01
categories: 
  - 数据结构与算法
  - 算法题
tags:
  - 算法
---

##  题目

求两个正整数的最大公约数，要求尽可能保证性能。

### 方法一：暴力枚举法

~~~java
public static int getGreatestCommonDivisor(int numA, int numB){
  	int samllNum = numA < numB ? numA : numB;
  	int bigNum = numA > numB ? numA : numB;
  	
  	if(bigNum % samllNum){
      	return samllNum;
  	}
  
  	int greatestCommonDivisor = 1;
  
  	for(int i = 2; i <= samllNum/2; i++){
      	if(numA % i == 0 && numB & i == 0){
          	greatestCommonDivisor = i;
      	}
  	}
  
  	 return greatestCommonDivisor;
}
~~~

暴力枚举的效率比较低，如果传入10000和10001，则需要循环4900次。

<!--more -->

### 方法二：辗转相除法

辗转相除法， 又名欧几里得算法（Euclidean algorithm），目的是求出两个正整数的最大公约数。这条算法基于一个定理：**两个正整数a和b（a>b），它们的最大公约数等于a除以b的余数c和b之间的最大公约数。**使用递归的方法来把问题逐步简化：

~~~java
public static int getGreatestCommonDivisor(int numA, int numB){
  	int result = 1;
  	
  	if(numA > numB){
      	result = gcd(numA, numB);
  	}else{
      	result = gcd(numB, numA);
  	}
  	return result
}

//递归计算最大公约数
private static int gcd(int a, int b){
  	if(a%b == 0){
       return b;
  	}else{
      	return gcd(b, a%b);
  	}  
}
~~~

注：当两个整型数较大时，做a%b取模运算性能会比较低。

### 方法三：更相减损术

更相减损术， 出自于中国古代的《九章算术》，他的原理更加简单：**两个正整数a和b（a>b），它们的最大公约数等于a-b的差值c和较小数b的最大公约数**。

~~~java
private static int gcd(int a, int b){
  	if(a == b){
       return a;
  	}
  	if(a < b){
      	return gcd(b-a, a);
  	}else{
      	 return gcd(a-b, b);
  	}
}
~~~

更相减损术避免了大整数取模的性能问题。但是更相减损术依靠两数求差的方式递归，当两数相差悬殊时，比如计算10000和1，就要递归9999次。有什么办法既可以避免大整数取模，又能尽可能减少运算次数？

### 方法四：移位运算

众所周知，移位运算的性能非常快。对于给定的正整数a和b，不难得到如下的结论。其中gcb(a,b)的意思是a,b的最大公约数函数：

- 当a和b均为偶数，gcb(a,b) = 2*gcb(a/2, b/2) = 2*gcb(a>>1, b>>1)
- 当a为偶数，b为奇数，gcb(a,b) = gcb(a/2, b) = gcb(a>>1, b)
- 当a为奇数，b为偶数，gcb(a,b) = gcb(a, b/2) = gcb(a, b>>1)
- 当a和b均为奇数，利用更相减损术运算一次，gcb(a,b) = gcb(b, a-b)， 此时a-b必然是偶数，又可以继续进行移位运算。

比如计算10和25的最大公约数的步骤如下：

1. 整数10通过移位，可以转换成求5和25的最大公约数
2. 利用更相减损法，计算出25-5=20，转换成求5和20的最大公约数
3. 整数20通过移位，可以转换成求5和10的最大公约数
4. 整数10通过移位，可以转换成求5和5的最大公约数
5. 利用更相减损法，因为两数相等，所以最大公约数是5

在两数比较小的时候，暂时看不出计算次数的优势，当两数越大，计算次数的节省就越明显。

~~~java
public static int gcd(int numA, int numB){
	  	if(numA == numB){
	      	return numA;
	  	}
	  	if(numA < numB){
	      	//保证参数A大于参数B，减少代码量
	      	return gcd(numB, numA);
	  	}else{
	      	//和1做按位与运算，判断奇偶
	      	if((numA&1) == 0  && (numB&1) == 0){ //a和b均为偶数
	          	return gcd(numA >> 1, numB >> 1) << 1;
	      	}else if((numA&1) == 0  && (numB&1) != 0){ //a为偶数,b为奇数
	          	return gcd(numA >> 1, numB);
	      	}else if((numA&1) != 0  && (numB&1) == 0){ //a为奇数,b为偶数
	          	return gcd(numA, numB >> 1);
	      	}else{ //a和b均为奇数
	          	return gcd(numA, numA - numB);
	      	}
	  	}
	}
~~~

## 总结

1. **暴力枚举法**：时间复杂度是O(min(a, b)))。
2. **辗转相除法**：时间复杂度不太好计算，可以近似为O(log(max(a, b)))，但是取模运算性能较差。
3. **更相减损术**：避免了取模运算，但是算法性能不稳定，最坏时间复杂度为O(max(a, b)))。
4. **更相减损术与移位结合**：不但避免了取模运算，而且算法性能稳定，时间复杂度为O(log(max(a, b)))。

如果两数都是偶数，计算差值之前会首先让两个数都折半，使得计算次数更少。这种方法做到了部分优化，但一奇一偶的情况也是可以优化的。