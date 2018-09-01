---
title: Spring Boot（五）：常用注解介绍
date: 2018-08-29 10:47:35
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

SpringBoot本身是基于Spring和SpringMvc等各类spring家族的一个解决方案，可快速进行集合。故相关知识点其实大部分都是基于spring或者springmvc既有的知识点的。这里主要讲解的是关于web开发及springboot独有的一些常用注解进行说明。

<!-- more -->

### 常见注解

#### 1. @SpringBootApplication

此注解是个组合注解，包括了`@SpringBootConfiguration`、`@EnableAutoConfiguration`和`@ComponentScan`注解：

- `@SpringBootConfiguration` 继承至`@Configuration`，此标注当前类是配置类，并会将当前类内声明的一个或多个以`@Bean`注解标记的方法的实例纳入到srping容器中，并且实例名就是方法名
- `@EnableAutoConfiguration`这个注解就是springboot能自动进行配置的魔法所在了。主要是通过此注解，能所有符合自动配置条件的bean的定义加载到spring容器中，比如根据spring-boot-starter-web ，来判断你的项目是否需要添加了webmvc和tomcat，就会自动的帮你配置web项目中所需要的默认配置。但比如需要排除一些无需自动配置的类时，可利用exclude进行排除
- `@ComponentScan` 会扫描当前包及其子包下被`@Component`，`@Controller`，`@Service`，`@Repository`等注解标记的类并纳入到spring容器中进行管理

#### 2. @Controller 和 @RestController

`@RestController` 是Spring4之后加入的注解，原来在`@Controller`中返回json需要`@ResponseBody`来配合，如果直接用`@RestController`替代`@Controller`就不需要再配置`@ResponseBody`，默认返回json格式。而`@Controller`是用来创建处理http请求的对象，一般结合`@RequestMapping`使用

#### 3. @RequestMapping

一个用来处理请求地址映射的注解，可用于类或方法上。用于类上，表示类中的所有响应请求的方法都是以该地址作为父路径。常用属性：

- value： 指定请求的实际地址，指定的地址可以是URI Template 模式；
- method： 指定请求的method类型， GET、POST、PUT、DELETE等；
- consumes： 指定处理请求的提交内容类型（Content-Type），例如application/json, text/html；
- produces: 指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回；
- params： 指定request中必须包含某些参数值是，才让该方法处理；
- headers： 指定request中必须包含某些指定的header值，才能让该方法处理请求；

#### 4. @RequestBody和@ResponseBody

- `@RequestBody`注解允许request的参数在reqeust体中，常常结合前端POST请求，进行前后端交互。
- `@ResponseBody`注解支持将的参数在reqeust体中，通常返回json格式给前端。

#### 5. @RequestParam

`@RequestParam `用来接收URL中的参数,如`/get?name=dodd`,可接收dodd作为参数:

~~~java
//http://localhost:8080/index/get?name=dodd
@RequestMapping("/get")
public Map<String,String> get(@RequestParam String name){
    Map<String,String> map = new HashMap<>();
    map.put("name",name);
    map.put("value","hello world!");
    return map;
}
~~~

#### 6. @PathVariable

@PathVariable用来接收参数，动态获取URL参数：

~~~java
//http://localhost:8080/index/find/1/dodd
@RequestMapping("/find/{id}/{name}")
public User get(@PathVariable int id, @PathVariable String name){
    User u = new User();
    u.setId(id);
    u.setName(name);
    u.setDate(new Date());
    return u;
}
~~~

#### 7. @RequestAttribute

@RequestAttribute用于访问由过滤器或拦截器创建的、预先存在的请求属性，效果等同与request.getAttrbute():

~~~java
@GetMapping("/req/attr")
public String reqAttr(@RequestAttribute("id") String id){
    return "id:" + id;
}
~~~

#### 8. @Component、@Service、@Repository

这三者都是申明一个单例的bean类并纳入spring容器中，后两者其实都是继承于@Component：

- `@Component `最普通的组件，可以被注入到spring容器进行管理
- `@Repository `作用于持久层
- `@Service `作用于业务逻辑层

通常一些类无法确定是使用`@Service`还是`@Component`时，注解使用`@Component`，比如redis的配置类等

#### 9. @ModelAttribute

主要是绑定请求参数到指定对象上。此注解可被用于方法、参数上:

- 运用在参数上，会将客户端传递过来的参数按名称注入到指定对象中，并且会将这个对象自动加入ModelMap中，便于View层使用；
- 运用在方法上，会在每一个`@RequestMapping`标注的方法前执行，如果有返回值，则自动将该返回值加入到ModelMap中；

由于现在都采用前后端分离开发，故此注解相对用的较少了，但对于一些在每次请求前需要进行一些额外操作时。使用此注解依然是个选择，比如进行统一的业务校验等，但使用此注解实现类似功能时需要注意，使用异步调用时，比如callable或者DeferredResult时，被此注解的方法会执行两次，因为异步请求时，是挂起另一个线程去重新执行，对于配置了拦截器而已，它们的执行顺序为:

~~~java
preHandle--->afterConcurrentHandlingStarted--->Controller--->preHandle--->postHandler--->afterCompletion
~~~

解决方案的话可简单根据DispatcherType类型进行判断，异步时对应类型为：ASYNC，第一次请求正常为：REQUEST。

#### 10. @autowired、@resource、@Qualifier

- `@Autowire` 默认按照类型装配，默认情况下它要求依赖对象必须存在如果允许为null，可以设置它required属性为false，如果我们想使用按照名称装配，可 以结合`@Qualifier`注解一起使用；
- `@Resource`默认按照名称装配，当找不到与名称匹配的bean才会按照类型装配，可以通过name属性指定，如果没有指定name属 性，当注解标注在字段上，即默认取字段的名称作为bean名称寻找依赖对象，当注解标注在属性的setter方法上，即默认取属性名作为bean名称寻找 依赖对象，但一旦指定了name属性，就只能按照名称 装配了；
- `@Qualifier` 则按照名称经行来查找转配的