---
title: Spring Boot（十三）：事务处理
date: 2018-09-02 12:59:44
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

关系型数据库实物处理通过注解`@Transactional`来实现。

### 事务的四个特性

1. 原子性（Atomicity）：事务是一个原子操作，由一系列动作组成。事务的原子性确保动作要么全部完成， 要么完全不起作用。 
2. 一致性（Consistency）：一旦事务完成（不管成功还是失败），系统必须确保它所建模的业务处于一致的状 态，而不会是部分完成部分失败。在现实中的数据不应该被破坏。 
3. 隔离性（Isolation）：可能有许多事务会同时处理相同的数据，因此每个事务都应该与其他事务隔离开来， 防止数据损坏。
4.  持久性（Durability）：一旦事务完成，无论发生什么系统错误，它的结果都不应该受到影响，这样就能从 任何系统崩溃中恢复过来。通常情况下，事务的结果被写到持久化存储器中。  

<!-- more -->

### 传播行为

当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。例如：方法可能继续在现有事务中运 行，也可能开启一个新事务，并在自己的事务中运行。 Spring 定义了七种传播行为：  

1. `PROPAGATION_REQUIRED` 表示当前方法必须运行在事务中。如果当前事务存在，方法将会在该事务中运 行。否则，会启动一个新的事务， **Spring 默认使用 **
2. `PROPAGATION_SUPPORTS` 表示当前方法不需要事务上下文，但是如果存在当前事务的话，那么该方法会 在这个事务中运行 
3. `PROPAGATION_MANDATORY `表示该方法必须在事务中运行，如果当前事务不存在，则会抛出一个异常 
4. `PROPAGATION_REQUIRED_NEW` 表示当前方法必须运行在它自己的事务中。一个新的事务将被启动。如果 存在当前事务，在该方法执行期间，当前事务会被挂起。如果使用 JTATransactionManager 的话，则需要 访问 TransactionManager 
5. `PROPAGATION_NOT_SUPPORTED` 表示该方法不应该运行在事务中。如果存在当前事务，在该方法运行期 间，当前事务将被挂起。如果使用 JTATransactionManager 的话，则需要访问 TransactionManager 
6. `PROPAGATION_NEVER` 表示当前方法不应该运行在事务上下文中。如果当前正有一个事务在运行，则会抛 出异常 
7. `PROPAGATION_NESTED` 表示如果当前已经存在一个事务，那么该方法将会在嵌套事务中运行。嵌套的事务 可以独立于当前事务进行单独地提交或回滚。如果当前事务不存在，那么其行为与 PROPAGATION_REQUIRED一样

### 隔离级别

隔离级别定义了一个事务可能受其他并发事务影响的程度。 

1. `ISOLATION_DEFAULT` 使用后端数据库默认的隔离级别， **Spring 默认使用**， mysql 默认的隔离级别为： `Repeatable Read(可重复读) `，mysql 为例， 存储引擎不能使用 MyISAM，应该使用 InnoDB  
2. `ISOLATION_READ_UNCOMMITTED` 读未提交， 最低的隔离级别，允许读取尚未提交的数据变更，可能会导致 脏读、幻读或不可重复读
3.  `ISOLATION_READ_COMMITTED` 读已提交， 允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读  或不可重复读仍有可能发生
4. ` ISOLATION_REPEATABLE_READ` 可重复读， 对同一字段的多次读取结果都是一致的，除非数据是被本身事 务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生 
5. `ISOLATION_SERIALIZABLE` 可串行化， 最高的隔离级别，完全服从 ACID 的隔离级别，确保阻止脏读、不可 重复读以及幻读，也是最慢的事务隔离级别，因为它通常是通过完全锁定事务相关的数据库表来实现的

**脏读（Dirty reads）** ——脏读发生在一个事务读取了另一个事务改写但尚未提交的数据时。如果改写再 稍后被回滚了，那么第一个事务获取的数据就是无效的。 

**不可重复读（Nonrepeatable read）** ——不可重复读发生在一个事务执行相同的查询两次或两次以上，但 是每次都得到不同的数据时。这通常是因为另一个并发事务在两次查询期间进行了更新。 

**幻读（Phantom read）** ——幻读与不可重复读类似。它发生在一个事务（T1）读取了几行数据，接着另一 个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在 的记录  

### 属性说明

`@Transactional`

1. `isolation`：用于指定事务的隔离级别。默认为底层事务的隔离级别。 
2. `noRollbackFor`：指定遇到指定异常时强制不回滚事务。 
3.  `noRollbackForClassName`：指定遇到指定多个异常时强制不回滚事务。该属性可以指定多个异常类 名。 
4. `propagation`:指定事务的传播属性。 
5. `readOnly`：指定事务是否只读。 表示这个事务只读取数据但不更新数据，这样可以帮助数据库引擎优 化事务。若真的是一个只读取的数据库应设置 readOnly=true 
6. `rollbackFor`：指定遇到指定异常时强制回滚事务。
7.  `rollbackForClassName`：指定遇到指定多个异常时强制回滚事务。该属性可以指定多个异常类名。
8.  `timeout`：指定事务的超时时长。  

### 示例

事务类：

~~~java
@Service
public class UserService {

    @Autowired
    private SbUserDao sbUserDao;

    @Autowired
    private JpaUserDao jpaUserDao;

    /**
     * 用户注册
     */
    @Transactional
    public String register(String name, String ip) {
        // 1.添加用户
        SbUser sbUser = new SbUser();
        sbUser.setName(name);
        sbUser.setCreateTime(new Date());
        sbUserDao.insert(sbUser);

        // 测试使用
        boolean flag = true;
        if (flag) {
            throw new RuntimeException();
        }

        // 2.添加注册日志
        JpaUser jpaUser = new JpaUser();
        jpaUser.setUserName(name);
        jpaUser.setUserIp(ip);
        jpaUser.setCreateTime(new Date());
        jpaUserDao.save(jpaUser);

        return "success";
    }
}
~~~

测试类：

~~~java
@Autowired
private UserService userService;
@Test
public void register() {
    String result = userService.register("dodd3", "192.168.1.1");
    System.out.println(result);
}
~~~

