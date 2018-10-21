---
title: Spring Cloud（三）：客户端负载均衡Ribbon
date: 2018-10-02 18:23:16
tags:
 - SpringCloud
categories: 
 - SpringCloud
---

### 介绍

Ribbon是Netflix发布的云中间层服务开源项目，其主要功能是提供客户端侧负载均衡算法。简单的说，Ribbon是一个客户端负载均衡器，我们可以在配置文件中列出Load Balancer后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询，随机连接等）去连接这些机器，我们也很容易使用Ribbon实现自定义的负载均衡算法。

<!-- more -->

### Ribbon工作原理

![](ribbon\ribbon01.png)

Ribbon工作时分为两步：

1. 第一步先选择 Eureka Server, 它优先选择在同一个Zone且负载较少的Server；
2. 第二步再根据用户指定的策略，在从Server取到的服务注册列表中选择一个地址。其中Ribbon提供了多种策略，例如轮询round robin、随机Random、根据响应时间加权等。

### 负载均衡

1. 客户端负载均衡（Ribbon）：服务实例的清单在客户端，客户端进行负载均衡算法分配；
2. 服务端负载均衡（Nginx）：服务实例的清单在服务端，服务器进行负载均衡算法分配

##### ribbon负载均衡算法

![](ribbon\ribbon02.png)

1. AvailabilityFilteringRule

   按可用性进行过滤服务的负载均衡策略。先线性轮询选出服务，通过predicate对象判断可用性

2. BestAvailableRule extends ClientConfigEnabledRoundRobinRule

   从所有的没有断开的服务中，找到请求数量最小的服务

3. ClientConfigEnabledRoundRobinRule

   默认通过RoundRobinRule策略选取服务，策略还不能自定义。这个策略的的目的就是通过继承该类，并且通过对choose的重写，来实现更多的功能

4. PredicateBasedRule extends ClientConfigEnabledRoundRobinRule

   通过调用AbstractServerPredicate实现类的过滤方法来过滤出目标的服务，再通过轮询方法选出一个服务

5. RandomRule

   随机选取负载均衡策略。通过随机Random对象，在所有服务实例数量中随机找一个服务的索引号，然后从上线的服务中获取对应的服务

6.  RetryRule

   这个策略默认就是用RoundRobinRule策略选取服务，当然可以通过配置，在构造RetryRule的时候传进想要的策略。为了应对在有可能出现无法选取出服务的情况，比如瞬时断线的情况，那么就要提供一种重试机制，在最大重试时间的限定下重复尝试选取服务，直到选取出一个服务或者超时

7. RoundRobinRule

   线性轮询负载均衡策略。在类中维护了一个原子性的成员变量作为当前服务的索引号，每次在所有服务数量的限制下，就是将服务的索引号加1，到达服务数量限制时再从头开始

8.  WeightedResponseTimeRule  extends RoundRobinRule

   响应时间作为选取权重的负载均衡策略。响应时间越短的服务被选中的可能性大

### 自定义ribbon负载均衡策略

1.  使用@RibbonClient

   ![](ribbon\ribbon03.png)

   这里的name属性必须是注册在Eureka Server上的服务名称

2. 定义Configuration配置类

   ![](ribbon\ribbon04.png)

3. 注意事项：

   - FooConfiguration必须是@Configuration，但要注意它不在主应用程序上下文的@ComponentScan中，否则它将由所有@RibbonClients共享。
   - 如果使用@ComponentScan（或@SpringBootApplication），则需要采取措施以避免包含它（例如将其放在单独的非重叠包中，或指定要在@ComponentScan中显式扫描的包）。

### 参考资料

1. [ribbon负载均衡策略](https://www.cnblogs.com/kongxianghai/p/8477781.html)
2. [Nginx 学习 —— 负载均衡](https://mp.weixin.qq.com/s/KPS0d7pIm_ViFcUlM1FYcg)