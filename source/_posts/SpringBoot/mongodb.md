---
title: Spring Boot（十五）：NoSql数据库mongodb
date: 2018-09-02 14:59:44
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

pom文件引入：

~~~java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
~~~

<!-- more -->

启动命令： 在mongo安装bin目录下`mongod.exe --dbpath d:\mongodb\ ` 注： 要先创建文件夹 ，保存临时文件 。指定路径： --dbpath  

配置文件：

~~~java
# MONGODB (MongoProperties)
spring.data.mongodb.uri=mongodb://localhost/test
spring.data.mongodb.port=27017
#spring.data.mongodb.authentication-database=
#spring.data.mongodb.database=test
#spring.data.mongodb.field-naming-strategy=
#spring.data.mongodb.grid-fs-database=
#spring.data.mongodb.host=localhost
#spring.data.mongodb.password=
#spring.data.mongodb.repositories.enabled=true
#spring.data.mongodb.username=
~~~

设置日志打印：<logger name="org.springframework.data.mongodb.core.MongoTemplate" level="debug"/>

### 方式一：MongoTemplate  

实现类：

~~~java
@Component
public class MongodbComponent {
    @Autowired
    private MongoTemplate mongoTemplate;
    public void insert(SbUser sbUser) {
        mongoTemplate.insert(sbUser);
    }
    public void deleteById(int id) {
        Criteria criteria = Criteria.where("id").in(id);
        Query query = new Query(criteria);
        mongoTemplate.remove(query, SbUser.class);
    }
    public void updateById(SbUser roncooUser) {
        Criteria criteria = Criteria.where("id").in(roncooUser.getId());
        Query query = new Query(criteria);
        Update update = new Update();
        update.set("name", roncooUser.getName());
        update.set("createTime", roncooUser.getCreateTime());
        mongoTemplate.updateMulti(query, update, SbUser.class);
    }
    public SbUser selectById(int id) {
        Criteria criteria = Criteria.where("id").in(id);
        Query query = new Query(criteria);
        return mongoTemplate.findOne(query, SbUser.class);
    }
}
~~~

测试类：

~~~java
@Autowired
private MongodbComponent mongodbComponent;
@Test
public void set() {
    SbUser sbUser = new SbUser();
    sbUser.setId(2); //mongoTemplate方式mongo主键不会自增
    sbUser.setName("doddmongo");
    sbUser.setCreateTime(new Date());
    mongodbComponent.insert(sbUser);
}
@Test
public void select() {
    System.out.println(mongodbComponent.selectById(1));
}
@Test
public void update() {
    SbUser sbUser = new SbUser();
    sbUser.setId(1);
    sbUser.setName("updateTest");
    sbUser.setCreateTime(new Date());
    mongodbComponent.updateById(sbUser);
    System.out.println(mongodbComponent.selectById(1));
}
@Test
public void delete() {
    mongodbComponent.deleteById(1);
}
~~~

### 方式二：MongoRepository

接口：

~~~java
public interface MongoDao extends MongoRepository<JpaUser, Integer> {
    JpaUser findByUserName(String string);
    JpaUser findByUserNameAndUserIp(String string, String ip);
    Page<JpaUser> findByUserName(String string, Pageable pageable);
}
~~~

测试类：

~~~java
@Autowired
private MongoDao mongoDao;
@Test
public void insertRep() {
    JpaUser entity = new JpaUser();
    entity.setId(1);//MongoRepository方式主键相同会覆盖
    entity.setUserName("doddRep");
    entity.setUserIp("192.168.0.1");
    entity.setCreateTime(new Date());
    mongoDao.save(entity);
}
@Test
public void deleteRep() {
    mongoDao.delete(1);
}
@Test
public void updateRep() {
    JpaUser entity = new JpaUser();
    entity.setId(2);
    entity.setUserName("doddRep2");
    entity.setUserIp("192.168.0.1");
    entity.setCreateTime(new Date());
    mongoDao.save(entity);
}
@Test
public void selectRep() {
    JpaUser result = mongoDao.findOne(1);
    System.out.println(result);
}
@Test
public void selectRep2() {
    JpaUser result = mongoDao.findByUserName("doddRep2");
    System.out.println(result);
}
// 分页
@Test
public void queryForPage() {
    Pageable pageable = new PageRequest(0, 20, 
                        new Sort(new Sort.Order(Sort.Direction.DESC, "id")));
    // Page<RoncooUserLog> result = mongoDao.findByUserName("doddRep2", pageable);
    Page<JpaUser> result = mongoDao.findAll(pageable);
    System.out.println(result.getContent());
}
~~~

### 方式三：使用嵌入式的 mongo  

可以不用启动本地mongo进行测试了。引入依赖即可使用：

~~~java
<dependency>
    <groupId>de.flapdoodle.embed</groupId>
    <artifactId>de.flapdoodle.embed.mongo</artifactId>
</dependency>
~~~



