---
title: Redis命令
date: 2018-01-13 15:12:39
categories: 
 - 数据库
tags:
 - Redis
---

## 字符串

Redis的字符串就是一个由字节组成的序列，在Redis里面，字符串可以存储以下3中类型的值：

- 字节串（byte string）
- 整数
- 浮点数

### 自增自减操作

Redis对字符串执行自增和自减的命令：

|     命令      |                  用例和描述                   |
| :---------: | :--------------------------------------: |
|    INCR     |     INCR key-name ---------将键存储的值加1      |
|    DECR     |     DECR key-name ---------将键存储的值减去1     |
|   INCRBY    | INCRBY key-name amount -------将键存储的值加上整数amount |
|   DECRBY    | DECRBY key-name amount -------将键存储的值减去整数amount |
| INCRBYFLOAT | INCRBYFLOAT key-name amount -------将键存储的值加上浮点数amount，redis2.6及以上可用 |

<!-- more -->

注意：

1. 当存储在redis字符串里面的值可以被解释（interpret）为十进制整数或者浮点数时，redis允许用户对这个字符串执行各种*INCR*和*DECR*操作；
2. 当用户对一个不存在的键或者保存了空串的键执行自增或者自减操作，则redis会将这个键当作0来处理；
3. 当用户尝试对一个值无法解释为整数或者浮点数的字符串键执行自增或者自减操作，则redis会返回一个错误。

Redis的INCR和DECR操作示例：

~~~java
Jedis conn = new Jedis("localhost");
conn.get('key');   //尝试获取一个不存在的键将得到一个none值，不显示
conn.incr('key');   //1
conn.incr('key',15);  //16
conn.decr('key',5);   //11
conn.get('key');    //'11'
conn.set('key',13);   //true
conn.incr('key');   //14
~~~

### Redis对字符串进行读取和写入操作

Redis处理子串和二进制位的命令：

|    命令    |                  用例和描述                   |
| :------: | :--------------------------------------: |
|  APPEND  | APPEND key-name value ---将值value追加到给定键key-name当前存储的值的末尾 |
| GETRANGE | GETRANGE key-name start end ---获取一个由偏移量start至end范围内的子串，包含start和end在内 |
| SETRANGE | SETRANGE key-name offset value ---将从offset偏移量开始的子串设置为给定值 |
|  GETBIT  | GETBIT key-name offset --- 将字节串看作是二进制位串（bit string），并返回位串中偏移量为offset的二进制位的值 |
|  SETBIT  | SETBIT key-name offset value --- 将字节串看作是二进制位串，并将位串中偏移量为offset的二进制位的值设置为value |
| BITCOUNT | BITCOUNT key-name [start end] --- 统计二进制位串中值为1的二进制位的数量 |
|  BITOP   | BITOP operation dest-key key-name [key-name ...] --- 对一个或多个二进制位串执行包括并（AND）、或（OR）、异或（XOR）、非（NOT）在内的任意一种按位运算操作（bitwise operation），并将计算得出的结果保存在dest-key键里面 |

注意：

1. redis现在的GETRANGE命令是由以前的SUBSTR命令改写而来的，因此，Python现在仍然可以使用substr()方法来获取子串；
2. 在使用SETRANGE或者SETBIT命令对字符串进行写入的时候，如果字符串当前的长度不能满足写入的要求，那么redis会自动地使用空字节null来将字符串扩展至所需的长度，然后才执行写入或者更新操作。
3. 在使用GETRANGE读取字符串的时候，超出字符串末尾的数据会被视为是空串，而在使用GETBIT读取二进制位串的时候，超出字符串末尾的二进制位会被视为是0。

Redis的子串操作和二进制位操作示例：

~~~java
//将'hello'追加到目前并不存在的'new-string-key'键里
conn.append('new-string-key', 'hello '); //返回6L，append命令会在执行之后返回字符串当前的长度

conn.append('new-string-key', 'world!'); //12L

//redis的索引以0开始，范围的终点（endpoint）默认包含在这个范围
conn.substr('new-string-key', 3, 7); //'lo wo'

//将h和w从小写改成大写
conn.setrange('new-string-key', 0, 'H'); //12 ，setrange命令会在执行之后同样返回字符串当前总长度
conn.setrange('new-string-key', 6, 'W'); //12
conn.get('new-string-key'); //'Hello World!'

//setrange命令既可以用于替换字符串里已有的内容，又可以增长字符串
conn.setrange('new-string-key', 11, ', how are you?'); //25
conn.get('new-string-key'); //'Hello World, how are you?'

//在对redis存储的二进制位进行解释（interpret）时，记住redis存储的二进制位是按照偏移量从高到低排列的
conn.setbit('another-key', 2, 1); //0, setbit命令会返回二进制位被设置之前的值
conn.setbit('another-key', 7, 1); //0
//通过将第2个二进制位以及第7个二进制位的值设置为1，键的值将变为'!'，也就是编码为33的字符
conn.get('another-key'); //'!'
~~~

Redis通过使用子串操作和二进制位操作，配合WATCH命令、MULTI命令和EXEC命令，用户可以自己动手去构建任何他们想要的数据结构。

## 列表

列表是由多个字符串组成的有序序列结构。Redis的列表允许用户从序列的两端推入或者弹出元素，获取列表元素，以及执行各种常见的列表操作。除此之外，列表还可以用来存储任务信息、最近浏览过的文章或者常用联系人信息。

一些常用的列表命令：

|   命令   |                  用例和描述                   |
| :----: | :--------------------------------------: |
| RPUSH  | RPUSH key-name value [value ...] --- 将一个或多个值推入列表的右端 |
| LPUSH  | LPUSH key-name value [value ...] --- 将一个或多个值推入列表的左端 |
|  RPOP  |     RPOP key-name --- 移除并返回列表最右端的元素      |
|  LPOP  |     LPOP key-name --- 移除并返回列表最左端的元素      |
| LINDEX | LINDEX key-name offset --- 返回列表偏移量为offset的元素 |
| LRANGE | LRANGE key-name start end --- 返回[start, end]范围内的所有元素 |
| LTRIM  | LTRIM key-name start end --- 对列表进行修剪，只保留[start, end]范围内的元素 |

常用命令示例：

~~~java
conn.rpush('list-key', 'last'); //1L 向列表推入元素，执行完毕会返回列表当前的长度
conn.lpush('list-key', 'first'); //2L
conn.rpush('list-key', 'new last');
conn.lrange('list-key', 0, -1); //['first', 'last', 'new last']
//通过重复地弹出列表左端的元素，可以按照从左到右的顺序来获取列表中的元素
conn.lpop('list-key'); //'first'
conn.lpop('list-key'); //'last' 
conn.lrange('list-key', 0, -1); //['new last']
//可以同时推入多个元素
conn.rpush('list-key', 'a','b','c'); //4L
conn.lrange('list-key', 0, -1); //['new last','a','b','c']
conn.ltrim('list-key', 2, -1) //true
  
conn.lrange('list-key', 0, -1); //['b','c']  
~~~

组合使用LTRIM和LRANGE可以构建一个在功能上类似于LPOP或RPOP，但能够一次返回并弹出多个元素的操作。下面的几个列表命令可以将元素从一个列表移动到另一个列表，或者阻塞（block）执行命令的客户端直到有其他客户端给列表添加元素为止。

阻塞式的列表弹出命令以及在列表之间移动元素的命令：

|     命令     |                  用例和描述                   |
| :--------: | :--------------------------------------: |
|   BLPOP    | BLPOP key-name [key-name ...] timeout --- 从第一个非空列表中弹出位于最左端的元素，或者在timeout秒之内阻塞并等待可弹出的元素出现 |
|   BRPOP    | BRPOP key-name [key-name ...] timeout --- 从第一个非空列表中弹出位于最右端的元素，或者在timeout秒之内阻塞并等待可弹出的元素出现 |
| RPOPLPUSH  | RPOPLPUSH source-key dest-key --- 从source-key 列表中弹出最右端的元素，然后将这个元素推入dest-key 列表的最左端，并向用户返回这个元素 |
| BRPOPLPUSH | BRPOPLPUSH source-key dest-key timeout --- 从source-key 列表中弹出最右端的元素，然后将这个元素推入dest-key 列表的最左端，并向用户返回这个元素；如果source-key为空，那么在timeout秒之内阻塞并等待可弹出的元素出现 |

 阻塞式命令示例：

~~~java
conn.rpush('list', 'item1'); //1
conn.rpush('list', 'item2'); //2
conn.rpush('list2', 'item3'); //1
//将一个元素从一个列表移动到另一个列表，并返回被移动的元素
conn.brpoplpush('list2', 'list', 1); //'item3'

//当列表不包含任何元素时，阻塞弹出操作会在给定的时限内等待可弹出的元素的出现，并在时限到达后返回None
conn.brpoplpush('list2', 'list', 1); 
conn.lrange('list', 0, -1); //['item3','iten1','item2']
conn.brpoplpush('list', 'list2', 1); //'item2'

//blpop命令会从左至右地检查传入的列表，并对最先遇到的非空列表执行弹出操作
conn.blpop(['list', 'list2'], 1); //('list', 'item3')
conn.blpop(['list', 'list2'], 1); //('list', 'item1')
conn.blpop(['list', 'list2'], 1); //('list2', 'item2')
conn.blpop(['list', 'list2'], 1);
~~~

对于阻塞弹出命令和弹出并推入命令，最常见的用例就是消息传递（messaging）和任务队列（task queue）。列表的一个主要优点在于它可以包含多个字符串值，这使得用户可以将数据集中在同一个地方。Redis的集合也提供了与列表类似的特性，但集合只能保存各不相同的元素。

## 集合

Redis的集合以无序的方式来存储多个各不相同的元素，用户可以快速地对集合执行添加元素操作、移除元素操作以及检查一个元素是否存在于集合里。

一些常用的集合命令：

|     命令      |                  用例和描述                   |
| :---------: | :--------------------------------------: |
|    SADD     | SADD key-name item [item ... ] --- 将一个或多个元素添加到集合里面，并返回被添加元素当中原本并不存在于集合里面的元素数量 |
|    SREM     | SREM key-name item [item ... ] --- 从集合里面移除一个或多个元素，并返回被移除元素的数量 |
|  SISMEMBER  | SISMEMBER key-name item --- 检查元素是否存在于集合key-name里 |
|    SCARD    |     SCARD key-name --- 返回集合包含的元素的数量      |
|  SMEMBERS   |    SMEMBERS key-name --- 返回集合包含的所有元素     |
| SRANDMEMBER | SRANDMEMBER key-name [count] --- 从集合里面随机地返回一个或多个元素。当count为正数时，命令返回的随机元素不会重复，当count为负数时，命令返回的随机元素可能会出现重复 |
|    SPOP     | SPOP key-name --- 随机地移除集合中的一个元素，并返回被移除的元素 |
|    SMOVE    | SMOVE source-key dest-key item --- 如果集合source-key包含元素item，那么从集合source-key里面移除元素item，并将元素item添加到集合dest-key中；成功返回1，否则返回0 |

redis集合示例：

~~~java
conn.sadd('set-key', 'a', 'b', 'c'); //3 --返回被添加元素的数量
conn.srem('set-key', 'c', 'd'); //1 --返回被移除元素的数量
conn.srem('set-key', 'c', 'd'); //0
conn.scard('set-key'); //2
conn.smembers('set-key'); //set(['a', 'b'])
//将元素移动到另一个集合
conn.smove('set-key', 'set-key2', 'a'); //true
conn.smove('set-key', 'set-key2', 'c'); //false
conn.smembers('set-key'); //set(['a'])
~~~

用于组合和处理多个集合的Redis命令：

|     命令      |                  用例和描述                   |
| :---------: | :--------------------------------------: |
|    SDIFF    | SDIFF key-name [key-name ...] --- 返回那些存在于第一个集合、但不存在于其他集合中的元素（差集运算） |
| SDIFFSTORE  | SDIFFSTORE dest-key key-name [key-name ... ] --- 将差集运算存储到dest-key键里面 |
|   SINTER    | SINTER key-name [key-name ...] --- 返回那些同时存在于所有集合中的元素（交集运算） |
| SINTERSTORE | SINTERSTORE dest-key key-name [key-name ... ] --- 将交集运算存储到dest-key键里面 |
|   SUNION    | SUNION key-name [key-name ...] --- 返回那些至少存在于一个集合中的元素（并集运算） |
| SUNIONSTORE | SUNIONSTORE dest-key key-name [key-name ... ] --- 将并集运算存储到dest-key键里面 |

redis的差集、交集、并集示例：

~~~java
conn.sadd('skey1', 'a', 'b', 'c', 'd'); //4
conn.sadd('skey2', 'c', 'd', 'e', 'f'); //4
conn.sdiff('skey1', 'skey2'); // set(['a', 'b'])
conn.sinter('skey1', 'skey2'); // set(['c', 'd'])
conn.sunion('skey1', 'skey2'); // set(['a', 'b', 'c', 'd', 'e', 'f'])
~~~

## 散列

Redis的散列可以让用户将多个键值对存储到一个Redis键里面。

用于添加和删除键值对的散列操作：

|  命令   |                  用例和描述                   |
| :---: | :--------------------------------------: |
| HMGET | HMGET key-name key [key ... ] --- 从散列获取一个或多个键的值 |
| HMSET | HMSET key-name key value [key value ... ] --- 为散列里面的一个或多个键设置值 |
| HDEL  | HDEL key-name key [key ...] --- 删除散列里面的一个或多个键值对，返回成功找到并删除的键值对数量 |
| HLEN  |      HLEN key-name --- 返回散列包含的键值对数量      |

hlen命令用于一次读取或者设置多个键的hmget和hmset，像hmget和hmset这种批量处理多个键的命令既可以给用户带来方便，又可以通过减少命令的调用次数以及客户端与Redis之间的通信往返次数来提升Redis的性能。

redis散列命令示例：

~~~java
conn.hmset('hash-key', {'k1':'v1', 'k2':'k2', 'k3':'k3'}); //true
conn.hmget('hash-key', ['k2', 'k3']); //['v2', 'v3']
//hlen命令通常用于调试一个包含非常多键值对的散列
conn.hlen('hash-key'); //3
//hdel命令在成功地移除了至少一个键值对时返回true
conn.hdel('hash-key', 'k1', 'k3'); //true
~~~

redis更高级特性：

|      命令      |                  用例和描述                   |
| :----------: | :--------------------------------------: |
|   HEXISTS    |  HEXISTS key-name key --- 检查给定键是否存在于散列中  |
|    HKEYS     |      HKEYS key-name --- 获取散列包含的所有键       |
|    HVALS     |      HVALS key-name --- 获取散列包含的所有值       |
|   HGETALL    |    HGETALL key-name --- 获取散列包含的所有键值对     |
|   HINCRBY    | HINCRBY key-name key increment --- 将键key存储的值加上整数increment |
| HINCRBYFLOAT | HINCRBYFLOAT key-name key increment --- 将键key存储的值加上浮点数increment |

尽管有hgetall存在，但hkeys和hvals也是很有用，如果散列包含的值非常大，那么用户可以先使用hkeys取出散列包含的所有键，然后再使用hget一个一个地取出键的值，从而避免一次获取多个大体积的值而导致服务器阻塞。hincrby和hincrbyfloat类似于处理字符串的incrby和incrbyfloat，区别就是前者处理散列，后者处理字符串。

散列高级特性示例：

~~~java
conn.hmset('hash-key2', {'short':'hello', 'long':1000*'1'}); //true
conn.hkeys('hash-key2'); //['long', 'short']
conn.hexists('hash-key2', 'num'); //false
//和字符串一样，对散列中尚未存在的键执行自增操作时，redis会将键的值当作0来处理
conn.hincrby('hash-key2', 'num'); //1L
conn.hexists('hash-key2', 'num'); //true
~~~

## 有序集合

和散列存储着键与值之间的映射类似，有序集合也存储着成员与分值之间的映射，并且提供了分值处理命令，以及根据分值大小有序地获取（fetch）或扫描（scan）成员和分值的命令。

一些常用的有序集合命令：

|   命令    |                  用例和描述                   |
| :-----: | :--------------------------------------: |
|  ZADD   | ZADD key-name score member [score member ... ] --- 将带有给定分值的成员添加到有序集合里面 |
|  ZREM   | ZREM key-name member [member ... ] --- 从有序集合里面移除给定的成员，并返回被移除成员的数量 |
|  ZCARD  |     ZCARD key-name --- 返回有序集合包含的成员数量     |
| ZINCRBY | ZINCRBY key-name increment member --- 将member成员的分值加上increment |
| ZCOUNT  | ZCOUNT key-name min max --- 返回分值介于min和max之间的成员数量 |
|  ZRANK  | ZRANK key-name member --- 返回成员member在有序集合中的排名 |
| ZSCORE  | ZSCORE key-name member --- 返回成员member的分值 |
| ZRANGE  | ZRANGE key-name start stop [WITHSCORES] --- 返回有序集合中排名介于start和stop 之间的成员，如果给定了WITHSCORES选项，则将分值一并返回 |

redis有序集合示例：

~~~java
conn.zadd('zset-key', 3, 'a', 2, 'b', 1, 'c'); //3
conn.zcard('zset-key'); //3
conn.zincrby('zset-key', 'c', 3); //4.0
//获取单个成员的分值对于实现计数器或者排行榜之类的功能非常有用
conn.zscore('zset-key', 'b'); //2.0
//排名以0开始
conn.zrank('zset-key', 'c'); //2
conn.zcount('zset-key', 0, 3); //2L
conn.zrem('zset-key', 'b'); //true
conn.zrange('zset-key', 0, -1, withscores); //[('a', 3.0), ('c', 4.0)]
~~~

有序集合的范围型数据获取和删除命令以及并集、交集命令：

|        命令        |                  用例和描述                   |
| :--------------: | :--------------------------------------: |
|     ZREVRANK     | ZREVRANK key-name member --- 返回有序集合里成员member的排名，成员按照分值从大到小排列（逆序） |
|    ZREVRANGE     | ZREVRANGE key-name start stop [WITHSCORES] --- 返回有序集合给定排名范围内的成员 |
|  ZRANGEBYSCORE   | ZRANGEBYSCORE key min max [WITHSCORES]\|[LIMIT offset count]返回有序集合中分值介于min和max之间的所有成员 |
| ZREVRANGEBYSCORE | ZREVRANGEBYSCORE key max min [WITHSCORES]\|[LIMIT offset count]获取有序集合中分值介于min和max之间的所有成员，并按分值从大到小的顺序来返回它们 |
| ZREMRANGEBYRANK  | ZREMRANGEBYRANK key-name start stop --- 移除有序集合中排名介于start和stop之间的所有成员 |
| ZREMRANGEBYSCORE | ZREMRANGEBYSCORE key-name min max --- 移除有序集合中分值介于min和max之间的所有成员 |
|   ZINTERSTORE    | ZINTERSTORE dest-key key-count key [key ... ] \| [WEIGHTS weight [weight ... ]] \| [AGGREGATE SUM\|MIN\|MAX] --- 对给定的有序集合执行类似于集合的交集运算 |
|   ZUNIONSTORE    | ZUNIONSTORE dest-key key-count key [key ... ] \| [WEIGHTS weight [weight ... ]] \| [AGGREGATE SUM\|MIN\|MAX] --- 对给定的有序集合执行类似于集合的并集运算 |

ZINTERSTORE和ZUNIONSTORE命令示例：

~~~java
conn.zadd('zset-1', 1, 'a', 2, 'b', 3, 'c'); //3
conn.zadd('zset-1', 4, 'b', 1, 'c', 0, 'd'); //3
conn.zinterstore('zset-i', ['zset-1', 'zset-2']); //2L
//默认使用的聚合函数是SUM
conn.zrange('zset-i', 0, -1, withscore=true); //[(4.0, 'c'), (6.0, 'b')]
conn.zunionstore('zset-u', ['zset-1', 'zset-2'], aggregate='min'); //4L
conn.zrange('zset-u', 0, -1, withscore=true); //[(0.0, 'd'), (1.0, 'a'), (1.0, 'c'), (2.0, 'b')]
//用户还可以把集合作为输入传给zinterstore和zunionstore，命令会将集合看作是成员分值全为1的有序集合来处理
conn.sadd('set-1', 'a', 'b'); //2
conn.zunionstore('zset-u2', ['zset-1', 'zset-2', 'set-1']); //4L
conn.zrange('zset-u2', 0, -1, withscore=true); //[(1.0, 'd'), (2.0, 'a'), (4.0, 'c'), (6.0, 'b')]
~~~

## 发布与订阅

发布与订阅（publish/subscribe）模式，又称pub/sub模式，redis也实现了这种模式。发布与订阅特点是订阅者（listener）负责订阅频道（channel），发送者（publisher）负责向频道发送二进制字符串消息（binary string message）。每当有消息被发送至给定频道时，频道的所有订阅者都会收到消息。

redis提供的发布与订阅命令：

|      命令      |                  用例和描述                   |
| :----------: | :--------------------------------------: |
|  SUBSCRIBE   | SUBSCRIBE channel [channel ... ] --- 订阅给定的一个或多个频道 |
| UNSUBSCRIBE  | UNSUBSCRIBE [channel [channel ... ]] --- 退订给定的一个或多个频道，如果执行时没有给定频道则退订所有频道 |
|   PUBLISH    |  PUBLISH channel message --- 向给定频道发送消息   |
|  PSUBSCRIBE  | PSUBSCRIBE pattern [pattern ... ] --- 订阅与给定模式相匹配的所有频道 |
| PUNSUBSCRIBE | PUNSUBSCRIBE [pattern [pattern ... ]] --- 退订给定的模式，如果执行时没有给定频道则退订所有模式 |

redis提供的发布与订阅很少用有两个原因：

1. redis系统的稳定性有关。对于旧版redis，当用户读取速度慢导致消息积压，会使得redis输出缓冲区体积变大，导致redis速度变慢甚至奔溃，新版会设置client-output-buffer-limit pubsub选项，会自动断开不符合要求的订阅客户端。
2. 数据传输的可靠性有关。客户端在执行订阅的过程中断线则会丢失断线期间发送的消息。后面总结[解决方法]()。

## 排序

Redis的排序操作和其他编程语言的排序操作一样，都可以根据某种比较规则对一系列元素进行有序的排列。负责执行排序操作的SORT命令可以根据字符串、列表、集合、有序集合、散列这5种键里面存储着的数据，对列表、集合以及有序集合进行排序。SORT命令的定义：

|  命令  |                  用例和描述                   |
| :--: | :--------------------------------------: |
| SORT | SORT source-key [BY pattern] \| [LIMIT offset count] \| [get pattern [get pattern ... ]] \| [ASC\|DESC] \| [ALPHA] \| [STORE dest-key] --- 根据给定的选项，对输入列表、集合或者有序集合进行排序，然后返回或者存储排序的结果 |

使用SORT命令提供的选项可以实现以下功能：

1. 根据降序而不是默认的升序来排序元素；
2. 将元素看作是数字来进行排序，或者将元素看作是二进制字符串来进行排序；
3. 使用被排序元素之外的其他值作为权重来进行排序，甚至还可以从输入的列表、集合、有序集合以外的其他地方进行取值。

SORT命令示例：

~~~java
conn.rpush('sort-input', 23, 15, 110, 7); //4
//根据数字大小排序
conn.sort('sort-input'); //['7', '15', '23', '110']
//根据字母表顺序排序
conn.sort('sort-input', alpha=true); //['110', '23', '15', '7']
//添加一些用于执行排序操作和获取操作的附加数据
conn.hset('d-7', 'field', 5); //1L
conn.hset('d-15', 'field', 1); //1L
conn.hset('d-23', 'field', 9); //1L
conn.hset('d-110', 'field', 3); //1L
//将散列的域（field）用作权重，对sort-input列表进行排序
conn.sort('sort-input', by='d-*->field'); //['15', '110', '7', '23']
//获取外部数据，并将它们用作命令的返回值，而不是返回被排序的数据
conn.sort('sort-input', by='d-*->field', get='d-*->field'); //['1', '3', '5', '9']
~~~

SORT是redis唯一一个可以同时处理3种不同类型的数据的命令，但基本的redis事务同样可以让我们在一连串不间断执行的命令里面操作多种不同类型的数据。

## 基本的Redis事务

Redis的基本事务（basic transaction）需要用到MULTI命令和EXEC命令，这种事务可以让一个客户端在不被其他客户端打断的情况下执行多个命令。Redis有5个命令可以让用户在不被打断（interruption）的情况下多个键执行操作，它们分别是WATCH、MULTI、EXEC、UNWATCH和DISCARD。和关系数据库那种可以在执行的过程中进行回滚（rollback）的事务不同，在redis里面，被multi命令和exec命令包围的所有命令会一个接一个地执行，直到所有命令都执行完毕为止。当一个事务执行完毕之后，redis才会处理其他客户端的命令。

在redis里面执行事务，当redis从一个客户端那里接收到multi命令时，redis会将这个客户端之后发送的所有命令都放到一个队列里面，直到这个客户端发送exec命令为止，然后redis就会在不被打断的情况下，一个接一个地执行存储在队列里面的命令。

说明：

1. multi和exec事务的一个主要作用是移除竞争条件；
2. 在redis里面使用流水线的另一个目的是提高性能，在执行一连串命令时，减少redis与客户端之间的通信往返次数可以大幅度降低客户端等待回复所需要的时间。

## 键的过期时间

过期时间特性对于清理缓存数据非常有用，但是只有少数几个命令可以原子地为键设置过期时间，并且对于列表、集合、散列和有序集合这样的容器（container）来说，键的过期时间命令只能为整个键设置而没有办法为键里面的单个元素设置过期时间，为了解决这个问题，我们可以用存储时间戳的有序集合来实现针对单个元素的过期时间。用于处理过期时间的redis命令：

|    命令     |                  示例和描述                   |
| :-------: | :--------------------------------------: |
|  PERSIST  |      PERSIST key-name --- 移除键的过期时间       |
|    TTL    |    TTL key-name --- 查看给定键距离过期时间还有多少秒     |
|  EXPIRE   | EXPIRE key-name seconds --- 让给定键在指定的秒数之后过期 |
| EXPIREAT  | EXPIREAT key-name timestsmp --- 将给定键的过期时间设置为给定的UNIX时间戳 |
|   PTTL    |     PTTL key-name 查看给定键距离过期时间还有多少毫秒      |
|  PEXPIRE  | PEXPIRE key-name milliseconds --- 让给定键在指定的毫秒数之后过期 |
| PEXPIREAT | PEXPIREAT key-name timestsmp-milliseconds --- 将一个毫秒级精度的UNIX时间戳设置为给定键的过期时间 |

redis过期时间命令示例：

~~~java
conn.set('key', 'value'); //true
conn.get('key'); //'value'
conn.expire('key', 2); //true
time.sleep(2);
conn.get('key');
conn.set('key', 'value2'); //true
conn.expire('key', 100); conn.ttl('key'); //true 100
~~~

