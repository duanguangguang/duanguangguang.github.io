---
title: MySQL存储引擎
date: 2018-03-13 14:22:17
categories: 
 - 数据库
 - MySQL
tags:
 - MySQL
---

## MySQL存储引擎

存储引擎说白了就是如何存储数据、如何为存储的数据建立索引和如何更新、查询数据等技术的实现方法。因为在关系数据库中数据的存储是以表的形式存储的，所以存储引擎也可以称为表类型（即存储和操作此表的类型）。
在Oracle 和SQL Server等数据库中只有一种存储引擎，所有数据存储管理机制都是一样的。而MySql数据库提供了多种存储引擎。用户可以根据不同的需求为数据表选择不同的存储引擎，用户也可以根据自己的需要编写自己的存储引擎。

<!-- more -->

## MySQL存储引擎分类

### MyISAM

这种引擎是mysql最早提供的。这种引擎又可以分为静态MyISAM、动态MyISAM 和压缩MyISAM三种：

- **静态MyISAM**：如果数据表中的各数据列的长度都是预先固定好的，服务器将自动选择这种表类型。因为数据表中每一条记录所占用的空间都是一样的，所以**这种表存取和更新的效率非常高。当数据受损时，恢复工作也比较容易做。**
- **动态MyISAM**：如果数据表中出现 `varchar、xxxtext或xxxBLOB` 字段时，服务器将自动选择这种表类型。相对于静态MyISAM，这种表**存储空间比较小**，但由于每条记录的长度不一，所以多次修改数据后，数据表中的数据就可能离散的存储在内存中，进而导致执行效率下降。同时，内存中也可能会出现很多碎片。因此，这种类型的表要经常用 `optimize table` 命令或优化工具来进行碎片整理。
- **压缩MyISAM**：以上说到的两种类型的表都可以用 `myisamchk` 工具压缩。这种类型的表进一步减小了占用的存储，但是这种表压缩之后不能再被修改。另外，因为是压缩数据，所以这种表在读取的时候要先时行解压缩。

但是，不管是何种MyISAM表，目前它都**不支持事务，行级锁和外键约束**的功能。

**MyISAM Merge引擎**：这种类型是MyISAM类型的一种变种。合并表是将几个相同的MyISAM表合并为一个虚表。常应用于日志和数据仓库。

### InnoDB

InnoDB表类型可以看作是对MyISAM的进一步更新产品，它**提供了事务、行级锁机制和外键约束**的功能。

### memory(heap)

这种类型的数据表只存在于内存中。它使用**散列索引**，所以数据的存取速度非常快。因为是存在于内存中，所以这种类型**常应用于临时表**中。

### archive

这种类型只支持select 和 insert语句，而且不支持索引。常应用于日志记录和聚合分析方面。

当然MySql支持的表类型不止上面几种。下面我们介绍一下如何查看和设置数据表类型。

## MySQL中关于存储引擎的操作

### 查看数据库可以支持的存储引擎

`show engines` 命令可以显示当前数据库支持的存储引擎情况

~~~sql
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.00 sec)
~~~

### 查看表的结构等信息的若干命令

`Desc[ribe] tablename` 查看数据表的结构

~~~sql
mysql> desc user;
+----------+-------------+------+-----+---------+----------------+
| Field    | Type        | Null | Key | Default | Extra          |
+----------+-------------+------+-----+---------+----------------+
| id       | int(11)     | NO   | PRI | NULL    | auto_increment |
| password | varchar(80) | YES  |     | NULL    |                |
| username | varchar(80) | YES  |     | NULL    |                |
+----------+-------------+------+-----+---------+----------------+
3 rows in set (0.00 sec)
~~~

`Show create table tablename` 显示表的创建语句

~~~sql
mysql> show create table user;
+-------+--------------------------------------------+
| Table | Create Table                               |
+-------+--------------------------------------------+
| user  | CREATE TABLE `user` (                      |
|       |  `id` int(11) NOT NULL AUTO_INCREMENT,     |
|       |  `password` varchar(80) DEFAULT NULL,      |
|       |  `username` varchar(80) DEFAULT NULL,      |
|       |  PRIMARY KEY (`id`)                        |
|       |  ) ENGINE=InnoDB DEFAULT CHARSET=utf8      |
+-------+--------------------------------------------+
1 row in set (0.01 sec)
~~~

`show table status like ‘tablename’\G` 显示表的当前状态值

~~~sql
mysql> show table status like 'user' \G;
*************************** 1. row ***************************
           Name: user
         Engine: InnoDB
        Version: 10
     Row_format: Dynamic
           Rows: 0
 Avg_row_length: 0
    Data_length: 16384
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: 1
    Create_time: 2017-10-23 23:53:45
    Update_time: NULL
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: 
        Comment: 
1 row in set (0.00 sec)

ERROR: 
No query specified
~~~

综上可见，后两种方式都可以帮助我们查看某一表的存储引擎类型

### 设置或修改表的存储引擎

创建数据库表时设置存储存储引擎的基本语法是：

~~~sql
Create table tableName(
columnName(列名1)  type(数据类型)  attri(属性设置),
columnName(列名2)  type(数据类型)  attri(属性设置),
……..
) engine = engineName
~~~

例如,假设要创建一个名为user的表,此表包括id,用户名username和性别sex三个字段，并且要设置表类型为merge。则可用如下的方式创建此数据表：

~~~sql
create table user(
  id int not null auto_increment,
  username char(20) not null,
  sex char(2),
  primary key(id)
) engine=merge
~~~

修改存储引擎，可以用命令：

~~~sql
Alter table tableName engine=engineName
~~~

假如，若需要将表user的存储引擎修改为archive类型，则可使用命令：

~~~sql
mysql> alter table user engine=archive;
Query OK, 0 rows affected (0.04 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> show table status like 'user' \G
*************************** 1. row ***************************
           Name: user
         Engine: ARCHIVE
        Version: 10
     Row_format: Compressed
           Rows: 0
 Avg_row_length: 487
    Data_length: 8720
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: 1
    Create_time: NULL
    Update_time: 2018-01-26 20:36:34
     Check_time: NULL
      Collation: utf8_general_ci
       Checksum: NULL
 Create_options: 
        Comment: 
1 row in set (0.00 sec)
~~~

## MySQL 常用数据存储引擎区别

mysql 有多种存储引擎，目前常用的是 `MyISAM` 和 `InnoDB` 这两个引擎。

### MyISAM

MyISAM 是 mysql 5.5.5 之前的默认引擎，它支持 B-tree/FullText/R-tree 索引类型。

锁级别为表锁，表锁优点是开销小，加锁快；缺点是锁粒度大，发生锁冲动概率较高，容纳并发能力低，这个引擎适合查询为主的业务。

此引擎不支持事务，也不支持外键。

MyISAM强调了快速读取操作。它存储表的行数，于是SELECT COUNT(*) FROM TABLE时只需要直接读取已经保存好的值而不需要进行全表扫描。

### InnoDB

InnoDB 存储引擎最大的亮点就是支持事务，支持回滚，它支持 Hash/B-tree 索引类型。

锁级别为行锁，行锁优点是适用于高并发的频繁表修改，高并发是性能优于 MyISAM。缺点是系统消耗较大，索引不仅缓存自身，也缓存数据，相比 MyISAM 需要更大的内存。

InnoDB 中不保存表的具体行数，也就是说，执行 select count(*) from table时，InnoDB 要扫描一遍整个表来计算有多少行。

支持事务，支持外键。

### Memory

Memory 是内存级别存储引擎，数据存储在内存中，所以他能够存储的数据量较小。

因为内存的特性，存储引擎对数据的一致性支持较差。锁级别为表锁，不支持事务。但访问速度非常快，并且默认使用 hash 索引。

Memory存储引擎使用存在内存中的内容来创建表，每个Memory表只实际对应一个磁盘文件，在磁盘中表现为.frm文件。

## 总结

|                     | MyISAM                                   | InnoDB                                   |
| ------------------- | ---------------------------------------- | ---------------------------------------- |
| 存储结构                | 每张表被存放在三个文件：frm-格定义MYD(MYData)-数据文件MYI(MYIndex)-索引文件 | 所有的表都保存在同一个数据文件中（也可能是多个文件，或者是独立的表空间文件），InnoDB表的大小只受限于操作系统文件的大小，一般为2GB |
| 存储空间                | MyISAM可被压缩，存储空间较小                        | InnoDB的表需要更多的内存和存储，它会在主内存中建立其专用的缓冲池用于高速缓冲数据和索引 |
| 可移植性、备份及恢复          | 由于MyISAM的数据是以文件的形式存储，所以在跨平台的数据转移中会很方便。在备份和恢复时可单独针对某个表进行操作 | 免费的方案可以是拷贝数据文件、备份 binlog，或者用 mysqldump，在数据量达到几十G的时候就相对痛苦了 |
| 事务安全                | 不支持 每次查询具有原子性                            | 支持 具有事务(commit)、回滚(rollback)和崩溃修复能力(crash recovery capabilities)的事务安全(transaction-safe (ACID compliant))型表 |
| AUTO_INCREMENT      | MyISAM表可以和其他字段一起建立联合索引                   | InnoDB中必须包含只有该字段的索引                      |
| SELECT              | MyISAM更优                                 |                                          |
| INSERT              |                                          | InnoDB更优                                 |
| UPDATE              |                                          | InnoDB更优                                 |
| DELETE              |                                          | InnoDB更优 它不会重新建立表，而是一行一行的删除              |
| COUNT without WHERE | MyISAM更优。因为MyISAM保存了表的具体行数               | InnoDB没有保存表的具体行数，需要逐行扫描统计，就很慢了           |
| COUNT with WHERE    | 一样                                       | 一样，InnoDB也会锁表                            |
| 锁                   | 只支持表锁                                    | 支持表锁、行锁 行锁大幅度提高了多用户并发操作的新能。但是InnoDB的行锁，只是在WHERE的主键是有效的，非主键的WHERE都会锁全表的 |
| 外键                  | 不支持                                      | 支持                                       |
| FULLTEXT全文索引        | 支持                                       | 不支持（5.6.4以上支持英文全文索引） 可以通过使用Sphinx从InnoDB中获得全文索引，会慢一点 |

互联网项目中随着硬件成本的降低及缓存、中间件的应用，一般我们选择都以 InnoDB 存储引擎为主，很少再去选择 MyISAM 了。而业务真发展的一定程度时，自带的存储引擎无法满足时，这时公司应该是有实力去自主研发满足自己需求的存储引擎或者购买商用的存储引擎了。

原文：

[浅谈MySql的存储引擎（表类型）](http://www.cnblogs.com/lina1006/archive/2011/04/29/2032894.html)

[MySQL 常用数据存储引擎区别](https://laravel-china.org/articles/4198/mysql-common-data-storage-engine)

