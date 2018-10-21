---
title: Spring Cloud（二）：服务注册发现Eureka
date: 2018-09-29 18:23:16
tags:
 - SpringCloud
categories: 
 - SpringCloud
---

### 介绍

Eureka是Netflix开发的服务发现框架，本身是一个基于REST的服务，主要用于定位运行在AWS域中的中间层服务，以达到负载均衡和中间层服务故障转移的目的。Spring Cloud将它集成在其子项目spring-cloud-netflix中。

<!-- more -->

### 服务间调用

![](C:/duanguangguang.github.io/source/_posts/SpringCloud/eureka/Eureka01.png)

将大的系统拆分成小的系统，子系统间远程调用就必须知道IP地址和端口，手动维护IP地址的变更就很麻烦，而在在SpringCloud中一般使用Eureka进行服务发现注册。

### 服务注册发现

创建一个服务发现组件，Eureka服务，将服务消费者和服务提供者都注册到Eureka服务上，Eureka服务维护这些注册服务的信息，包括：服务名和IP地址。

![](eureka\Eureka02.png)

服务消费者和服务提供者都可以拿到Eureka服务的注册清单，服务间相互调用不再通过具体的IP地址，而是通过服务名来调用，服务名一般不会变。

#### 服务发现方式

1. 客户端发现：Eureka, Zk
2. 服务端发现：Consul, nginx

#### Eureka原理

1. 区域（Region）和可用区（Availability Zone，AZ）概念

![](eureka\Eureka03.png)

2. Eureka原理

   ![](eureka\Eureka04.png)

   1.  Application Service 就相当于服务提供者，Application Client就相当于服务消费者；
   2.  Make Remote Call，可以简单理解为调用RESTful的接口；
   3.  us-east-1c、us-east-1d等是zone，它们都属于us-east-1这个region；
   4.  由图可知，Eureka包含两个组件：Eureka Server 和 Eureka Client；
   5.  Eureka Server提供服务注册服务，各个节点启动后，会在Eureka Server中进行注册；
   6.  Eureka Client是一个Java客户端，用于简化与Eureka Server的交互，客户端同时也具备一个内置的、使用轮询（round-robin）负载算法的负载均衡器；
   7. 在应用启动后，将会向Eureka Server发送心跳（默认周期为30秒），如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳，Eureka Server将会从服务注册表中把这个服务节点移除（默认90秒）；
   8.  Eureka Server之间将会通过复制的方式完成数据的同步（数据一致性）；
   9.  Eureka还提供了客户端缓存的机制，客户端可以利用缓存中的信息消费其他服务的API。

#### Eureka的服务注册发现机制

##### Eureka Server

1. 失效剔除：默认每隔一段时间（默认为60秒） 将当前清单中超时（默认为90秒）没有续约的服务剔除出去。
2. 自我保护：EurekaServer 在运行期间，会统计心跳失败的比例在15分钟之内是否低于85%(通常由于网络不稳定导致)。 Eureka Server会将当前的实例注册信息保护起来， 让这些实例不会过期，尽可能保护这些注册信息。

##### 服务提供者

1. 服务注册：启动的时候会通过发送REST请求的方式将自己注册到Eureka Server上，同时带上了自身服务的一些元数据信息。
2. 服务续约：在注册完服务之后，服务提供者会维护一个心跳用来持续告诉Eureka Server:  "我还活着 ” 。
3. 服务下线：当服务实例进行正常的关闭操作时，它会触发一个服务下线的REST请求给Eureka Server, 告诉服务注册中心：“我要下线了 ”。

##### 服务消费者

1. 获取服务：当我们启动服务消费者的时候，它会发送一个REST请求给服务注册中心，来获取上面注册的服务清单。
2. 服务调用：服务消费者在获取服务清单后，通过服务名可以获得具体提供服务的实例名和该实例的元数据信息。在进行服务调用的时候，优先访问同处一个Zone中的服务提供方。

### 参考资料

1. [服务发现方式](http://blog.daocloud.io/microservices-4/)

2. [AWS的区域和可用区概念解释](https://blog.csdn.net/awschina/article/details/17639191)

3. [分布式中几种服务注册与发现组件的原理与比较](https://mp.weixin.qq.com/s/Qu3o2ffayBdw3k2gHzuWyQ)