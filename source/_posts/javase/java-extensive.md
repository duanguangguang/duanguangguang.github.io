---
title: JAVA基础(四)泛型
date: 2017-08-07 16:15:22
categories: 
 - JAVA
 - JavaSE
tags:
 - JavaSE
---

## java基础之泛型

泛型（Generic type 或者generics）是对 Java 语言的类型系统的一种扩展，以支持创建可以按类型进行参数化的类。可以把类型参数看作是使用参数化类型时指定的类型的一个占位符，就像方法的形式参数是运行时传递的值的占位符一样。泛型是JDK1.5出现的安全机制。好处：

1. 将运行期间的问题classCastException转移到了编译时期
2. 避免了强制转换的问题
3. 潜在的性能收益。泛型为较大的优化带来可能。在泛型的初始实现中，编译器将强制类型转换（没有泛型的话，程序员会指定这些强制类型转换）插入生成的字节码中。但是更多类型信息可用于编译器这一事实，为未来版本的JVM 的优化带来可能。

<!-- more -->

### 泛型简介

1. 使用场景

   - <>是一个用于接受具体的引用数据类型的参数范围
   - 泛型技术是给编译器使用的技术，用于编译时期，确保类型的安全

2. 泛型的擦除和补偿

   - 擦除：运行时会将泛型去掉，生成的class文件是不带泛型的，这个是为了兼容运行时的类加载器，即不改变原来的运行时的类加载器，原来怎么运行，现在还怎么运行
   - 补偿：在运行时，通过获取元素的类型进行转换动作，不用使用者再强制转换

3. 使用TreeSet集合

   - 使用TreeSet集合，所使用的泛型的引用类必须实现comparable接口，该接口也有泛型，类型是该引用类型Comparator<Person>，例如：

     ~~~java
     TreeSet<Person> ts = new TreeSet<Person>();
     ~~~

   - 定义的比较器实现comparator接口

     ~~~java
     public class ComparatorByName implements Comparator<Person>{
       public int compare(Person p1, Person p2){
         int temp = p1.getName().compareTo(p2.getName());
         return temp == 0?p1.getAge()-p2.getAge():temp;
       }
     }
     ~~~

   - 定义的引用类实现comparable接口

     ~~~java
     public class Person implements Comparable<Person>{
       //里面实现自定义的比较方法
       public int compareTo(Person p){
         int temp = this.age-p.age;
         return temp == 0?this.name.compareTo(p.name):temp;
       }
     }
     ~~~

   - 泛型里面只能写引用数据类型，基本数据类型使用包装类

4. 泛型不是协变的

   - 关于泛型的混淆，一个常见的来源就是假设它们像数组一样是协变的。其实它们不是协变的。List<Object>不是List<String>的父类型。 
   - 如果 A 扩展 B，那么 A 的数组也是 B 的数组，并且完全可以在需要B[]的地方使用A[]： 

   ~~~java
   Integer[] intArray = new Integer[10]; 
   Number[] numberArray = intArray; 
   ~~~

   上面的代码是有效的，因为一个Integer是一个Number，因而一个Integer数组是一个Number数组。

   - 但是对于泛型来说，下面的代码是无效的

   ~~~java
   List<Integer> intList = new ArrayList<Integer>(); 
   List<Number> numberList = intList; // invalid
   ~~~

### 泛型类

在JDK1.5之后，使用泛型来接收类中要操作的引用数据类型，这就是泛型类（自定义泛型类）

~~~java
public class Tool<QQ>{
  private QQ q;
  public QQ getObject(){
    return q;
  }
  public void setObject(QQ object){
    this.q = object;
  }
}
~~~

1. 使用场景

   当类中的操作的引用数据类型不确定的时候，就用泛型类表示，当不使用泛型，就使用object（需要强转）

2. 使用泛型比object安全，将运行时错误提前到编译期

   ~~~java
   public class Test{
     public static void main(String[] args){
       Tool<Student> tool = new Tool<Student>();
       tool.setObject(new Student());
       Student stu = tool.getObject();
       //Student stu = (Student)tool.getObject();使用了泛型不用强转
     }
   }
   ~~~

### 泛型方法

泛型方法是跟着对象走的

1.  非泛型方法

   ~~~java
   public void show(String str){
     syso("show:"+str);
   }

   Tool<String> tool = new Tool<String>();
   tool.show("abc");
   tool.show(new Integer(4));//类型报错
   ~~~

2. 泛型方法

   ~~~java
   public <W> void show(W str){//等同于object,所以可以自己定义W
     syso("show:"+str);
   }
   //将泛型定义在方法上，类似于object，传什么就show什么
   Tool<String> tool = new Tool<String>();
   tool.show("abc");
   tool.show(new Integer(4));//因为是自定义泛型方法，不会报错
   ~~~

3. 泛型定义在方法上等同于object，不能使用具体对象的方法，只能使用object的方法

4. 当方法静态时，不能访问类上定义的泛型（静态不需要对象，泛型需要对象明确），所以静态方法使用泛型只能定义在方法上

   ~~~java
   public static <Y> void method(Y obj){
     syso(obj);
   }
   //使用
   tool.method("haha");
   tool.method(new Integer(9));
   ~~~

### 泛型接口

1. 泛型接口

   ~~~java
   public interface Inter<T>{
     public void show(T t);
   }
   ~~~

2. 泛型接口使用1

   ~~~java
   public class InterImp1 implements Inter<String>{
      public void show(String str){
        syso("show:"+str);
      }
   }

    public static void main(String[] args){
      InterImp1 in = new InterImp1();
      in.show("abc");
    }
   ~~~

3. 泛型接口使用2

   ~~~java
   public class InterImp2 implements Inter<Q>{
      public void show(Q q){
        syso("show:"+q);
      }
   }

    public static void main(String[] args){
      InterImp2<Integer> in = new InterImp2<Integer>();
      in.show(5);
    }
   ~~~

### 泛型通配符

？，未知类型

1. 通配符的基本使用

   ~~~java
   ArrayList<String> al = new ArrayList<String>();
   al.add("abc");
   ArrayList<Integer> al2 = new ArrayList<Integer>();
   al2.add(5);
   printCollection(a1);
   printCollection(a2);

   public static void printCollection(Collection<?> al){
     Iterator<?> it = al.iterator();
     while(it.hasNext()){
       syso(it.next()); //这里不能用？str = it.next();
     }
   }
   ~~~

2. 类似的将泛型定义在方法上

   ~~~java
   public static <T> void printCollection(Collection<T> al){
     Iterator<T> it = al.iterator();
     while(it.hasNext()){
       T str = it.next();
       syso(str);
     }
   }
   ~~~

3. T和？的区别

   ~~~java
   public static <T> void printCollection(Collection<T> al){
     Iterator<T> it = al.iterator();
     while(it.hasNext()){
       T str = it.next();
       syso(str);//区别就是可以对T的类型进行操作
       syso(it.next())//?的使用，类似于object，只有调用object方法都可以it.next().toString();
     }
   }
   ~~~

### 泛型的限定

1. 泛型的上限 --- ? extends E

   - 当引用类型（泛型类型）为Student和Worker时

     ~~~java
     public static void printCollection(Collection<? extends Person> al){}
     ~~~

     ? extends Person表示类型只接收Person及其子类

     ~~~java
     Iterator<? extends Person> it = al.iterator();
       while(it.hasNext()){
         Person p = it.next();//限定后可以使用父类方法
         syso(p.getName()+":"+p.getAge());
       }
     ~~~

   - 上限体现

     ~~~java
     class MyCollection<E>{
       public void add(E e){
         
       }
       public void addAll(MyCollection<? extends E> e){
         
       }
     }
     ~~~

     存元素一般都是上限，这样取出元素都是按照上限类型来运算的，不会出现安全隐患

2. 泛型的下限 --- ? super E

   - ? super student表示类型只接收student及其父类

     ~~~java
     public static void printCollection(Collection<? super Student> al){
       Iterator<? super Student> it = al.iterator();
       while(it.hasNext()){
         syso(it.next());
       }
     }
     ~~~

   - 下限体现

     ~~~java
     class TreeSet<Student>{
       TreeSet(Comparator<? super Student> comp);
     }
     ~~~

     通常对集合中的元素进行取出操作时，可以使用下限（存什么类型用什么类型接收，存什么类型用父类型接收）

3. 与object最大的区别就是安全，object全存，泛型限定部分存取

