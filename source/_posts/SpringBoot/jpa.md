---
title: Spring Boot（十二）：关系型数据库之spring-data-jpa
date: 2018-09-02 11:26:13
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

pom引入：

~~~java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
~~~

JPA配置：

~~~java
# JPA自动更新检查，没有表则会新增
spring.jpa.hibernate.ddl-auto= update
# 显示sql语句
spring.jpa.show-sql=true
~~~

<!-- more -->

### 默认方法

实体类：

~~~java
@Entity
public class JpaUser {
    @Id
    @GeneratedValue
    private Integer id;
    @Column
    private Date createTime;
    @Column
    private String userName;
    @Column
    private String userIp;
    // 省略setter/getter
}
~~~

接口：

~~~java
public interface JpaUserDao extends JpaRepository<JpaUser, Integer> {
    // 先不用自定义，JpaRepository有很多默认方法使用
}
~~~

测试类：

~~~java
 @Autowired
    private JpaUserDao jpaUserDao;

    @Test
    public void insert() {
        JpaUser entity = new JpaUser();
        entity.setUserName("dodd");
        entity.setUserIp("192.168.0.1");
        entity.setCreateTime(new Date());
        jpaUserDao.save(entity);
    }
    @Test
    public void delete() {
        jpaUserDao.delete(1);
    }
    @Test
    public void update() {
        JpaUser entity = new JpaUser();
        entity.setId(2);
        entity.setUserName("dodd 2");
        entity.setUserIp("192.168.0.1");
        entity.setCreateTime(new Date());
        jpaUserDao.save(entity);
    }
    @Test
    public void select() {
        JpaUser result = jpaUserDao.findOne(2);
        System.out.println(result);
    }
~~~

### 自定义方法

1. 使用内置的关键词查询  

   `https://docs.spring.io/spring-data/jpa/docs/1.10.2.RELEASE/reference/html/`

2. 使用自定义语句查询  

   `https://docs.spring.io/spring-data/jpa/docs/1.10.2.RELEASE/reference/html/`

3. @Query 注解

   例：`@Query(value = "select u from JpaUser u where u.userName=?1")  `

接口：

~~~java
public interface JpaUserDao extends JpaRepository<JpaUser, Integer> {

    /**
     * 内置关键字匹配
     * 通过实体类属性名称匹配
     */
    List<JpaUser> findByUserName(String userName);

    /**
     * 自定义扩展方法
     * 通过实体类属性名称匹配
     * 多个属性使用And连接
     */
    List<JpaUser> findByUserNameAndUserIp(String string, String string2);

    /**
     * 自定义扩展方法
     * 不匹配时使用@Query
     * 注解的优先级高
     */
    @Query(value = "select u from JpaUser u where u.userName=?1")
    List<JpaUser> findByName(String userName);

    /**
     * 分页
     */
    Page<JpaUser> findByUserName(String userName, Pageable pageable);
}
~~~

测试类：

~~~java
@Test
public void select1() {
    List<JpaUser> result = jpaUserDao.findByUserName("dodd 2");
    System.out.println(result);
}

@Test
public void select2() {
    List<JpaUser> result = jpaUserDao.findByName("dodd 2");
    System.out.println(result);
}

@Test
public void select3() {
    List<JpaUser> result = jpaUserDao.findByUserNameAndUserIp("dodd 2", "192.168.0.1");
    System.out.println(result);
}

// 分页
@Test
public void queryForPage() {
    Pageable pageable = new PageRequest(0, 20, new Sort(new Sort.Order(Sort.Direction.DESC, "id")));
    Page<JpaUser> result = jpaUserDao.findByUserName("dodd 2", pageable);
    System.out.println(result.getContent());
}
~~~



 