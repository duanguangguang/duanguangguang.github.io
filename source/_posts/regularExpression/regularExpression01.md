---
title: JAVA正则表达式
date: 2017-08-12 20:41:55
categories: 
 - 正则表达式
tags:
 - RegularExpression
---



## 正则表达式

正则表达式是对[字符](https://baike.baidu.com/item/%E5%AD%97%E7%AC%A6)串操作的一种逻辑公式，就是用事先定义好的一些特定字符、及这些特定字符的组合，组成一个“规则字符串”，这个“规则字符串”用来表达对字符串的一种过滤逻辑。给定一个正则表达式和另一个字符串，我们可以达到如下的目的：

1. 给定的字符串是否符合正则表达式的过滤逻辑（称作“匹配”）
2. 可以通过正则表达式，从字符串中获取我们想要的特定部分

<!-- more -->


### 一、正则表达式常见规则

1. 字符

   | X    | 字符x   |
   | ---- | ----- |
   | \ \  | 反斜线字符 |
   | \t   | 制表    |
   | \n   | 换行    |
   | \r   | 回车    |
   | \f   | 换页    |

2. 字符类

   | [abc]          | a、b或c     |
   | -------------- | --------- |
   | [a-zA-Z]       | a到z或A到Z   |
   | [a-z&&[def]]   | d、e或f(交集) |
   | [^abc]         | 非a、b、c    |
   | [a-d[m-p]]     | a到d或m到p   |
   | [a-z&&[ ^bc]]  | a到z除b、c   |
   | [a-z&&[ ^m-p]] | a-z除m到p   |

3. 预定义字符类

   | .    | 任何字符               |
   | ---- | ------------------ |
   | \d   | 数字[0-9]            |
   | \D   | 非数字                |
   | \s   | 空白字符[\t\n\f\r\xOB] |
   | \S   | 非空白字符              |
   | \w   | 单词字符[a-zA-Z_0-9]   |
   | \W   | 非单词字符              |

4. 边界匹配器

   | ^    | 行的开头     |
   | ---- | -------- |
   | $    | 行的结尾     |
   | \b   | 单词边界     |
   | \B   | 非单词边界    |
   | \A   | 输入的开头    |
   | \G   | 上一个匹配的结尾 |
   | \z   | 输入的结尾    |
   | \Z   | 输出的结尾    |

5. Greedy数量词

   | x?     | 至多一次 |
   | ------ | ---- |
   | x*     | 零或多次 |
   | x+     | 至少一次 |
   | x{n}   | 恰好n次 |
   | x{n,}  | 至少n次 |
   | x{n,m} | n到m次 |

6. 正则表达式组概念

   | ()   | 表示组，组后面\数组表示组编号，组零表示整个表达式 |
   | ---- | ------------------------- |
   | {}   | 表示重复操作                    |
   | []   | 用于定义字符集                   |



### 二、正则表达式匹配

其使用的就是String类中的matches方法。例：

~~~java
String regex = "1[358][0-9]{9}";//匹配电话号码
//String regex = "1[358]\\d{9}";
~~~



### 三、正则表达式切割

其使用的就是String类中的split方法。例：

~~~java
String str = "atttbmmc";//切割字符串
str.split("{.}\\1+");//任意字符重复时切割
/*
1.{.}表示复用的，重复的[.]任意字符
2.\\1表示，第一个\表示在字符串中需要转义，第二个\表示将1转义成组编号，表示第一组
*/
~~~



### 四、正则表达式替换

其使用的就是String类中的replaceAll方法。例：

~~~java
String str = "zhangsantttlisimmm";
str = str.replaceAll("(.)\\1+","#"); //将叠词替换成#
str = str.replaceAll("(.)\\1+","$1");//获取第一个叠词前面的内容

String tel = "15800001111";//158****1111
tel = tel.replaceAll("(\\d{3})\\d{4}(\\d{4})","$1****$2");
~~~



### 五、正则表达式获取

类Pattern，java.util.regex包。

1. 将正则规则进行对象的封装

   ~~~java
   Pattern p = Pattern.compile("a*b");
   ~~~

2. 通过正则对象的matches方法和字符串相关联，获取要对字符串操作的匹配器对象Matcher

   ~~~java
   Matcher m = p.matcher("aaaaab");
   ~~~

3. 通过Matcher匹配器对象的方法对字符串进行操作

   ~~~java
   boolean b = m.matches();
   ~~~

例：

~~~java
String str = "da jia hao,ming tian bu fang jia";//获取三个字母的单词
String regex = "\\b[a-z]{3}\\b";
//1.将正则封装成对象
Pattern p = Pattern.compile(regex);
//2.通过正则对象获取匹配器对象
Matcher m = p.matcher(str);
//3.使用Matcher匹配器对象的方法对字符串进行操作，既然要获取三个字母的单词，则查找find()
while(m.find()){
  syso(m.group());//获取匹配的子序列
}
//匹配的起始索引
m.start();//3   jia---[3,6)
//匹配的结束索引
m.end();//6
~~~



### 六、正则表达式练习

1. 字符串替换

   ~~~java
   String str = "我我...我..要要要...学学学...编程程";
   //1.将字符串.去掉
   str = str.replaceAll("\\.+","");
   //2.替换叠词
   str = str.replaceAll("(.)\\1+","$1");
   ~~~

2. 对ip地址排序

   ~~~java
   String ip_str = "192.168.10.34 127.0.0.1 3.3.3.3 105.70.11.55";
   //1.将ip地址切出
   String[] ips = ip_str.split(" +");//空格
   //2.为了让ip可以按照字符串顺序比较（TreeSet自带），只要让ip的每一段位数相同，补零
   ip_str = ip_str.replaceAll("(\\d+)","00$1");
   //3.然后每一段保留位数3位
   ip_str = ip_str = .replaceAll("0*(\\d{3})","$1");
   //4.再将ip地址切出
   String[] ips = ip_str.split(" +");//空格
   //5.排序
   TreeSet<String> ts = new TreeSet<String>();
   for(String ip :ips){
     ts.add(ip);
   }
   //ip地址复原
   for(String ip :ts){
     syso(ip.replaceAll("0*(\\d+)","$1"));
   }
   ~~~

3. 对邮箱校验

   ~~~java
   String mail = "abc1@sina.com";
   String regex = "[a-zA-Z0-9_]+@[a-zA-Z0-9]+(\\.[a-zA-Z]{1,3})+";
   regex = "\\W+@\\w+(\\.\\w+)+";//1@1.1，笼统校验
   //mail.indexof("@") != -1;
   boolean b = mail.matches(regex);
   syso(mail+":"+b);
   ~~~

4. 爬虫练习

   网页爬虫：其实就是一个程序用于在互联网中获取符合指定规则的数据。例：爬邮箱地址，本地

   ~~~java
   public static List<String> getMails() throws IOException{
     //1.读取源文件
     BufferedReader bufr = new BufferedReader(new FileReader("c:\\mail.html"));
     //2.对读取的数据进行规则的匹配，从中获取符合规则数据
     String mail_regex = "\\W+@\\w+(\\.\\w+)+";
     Pattern p = Pattern.compile(mail_regex);
     String line = null;
     List<String> list = new ArrayList<String>();
     while((line = bufr.readLine()) != null){//读每行数据
       Matcher m = p.matcher(line);//读一行匹配一行
       //3.获取到加到集合
       while(m.find()){//匹配器
         list.add(m.group());
       }
     }
     return list;
   }

   public static void main(String[] args)throws IOException{
     List<String> list = getMails();
     for(String mail : list){
       syso(mail);
     }
   }
   ~~~

   爬网络的话，改变源文件

   ~~~java
   URL url = new URL("http://baidu.com");
   BufferedReader bufr = new BufferedReader(new InputStreamReader(url.openStream()));
   ~~~

   ​

   ​

   ​

   ​