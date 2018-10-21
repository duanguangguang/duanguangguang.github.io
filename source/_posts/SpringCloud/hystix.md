---
title: Spring Cloud（四）：断路器Hystrix
date: 2018-10-03 18:23:16
tags:
 - SpringCloud
categories: 
 - SpringCloud
---

### 介绍

雪崩效应：在高并发的情况下，由于单个服务的延迟，可能导致所有的请求都处于延迟状态，甚至在几秒钟就使服务处于负载饱和的状态，资源耗尽，直到不可用，最终导致这个分布式系统都不可用，这就是“雪崩”。Hystrix实现了断路器、线程隔离等一系列服务保护功能。

<!-- more -->

### Hystrix工作原理

1. 超时机制

   通过网络请求其他服务时，都必须设置超时时间。

2. Fallback

   Fallback(失败快速返回)：当某个服务发生故障之后，通过断路器的故障监控， 向调用方返回一个错误响应， 而不是长时间的等待。这样就不会使得线程因调用故障服务被长时间占用不释放，避免了故障在分布式系统中的蔓延。

   ~~~java
   @HystrixCommand(fallbackMethod = "findByIdFallback")
   ~~~

3. 线程池隔离

   资源/依赖隔离(线程池隔离)：它会为每一个依赖服务创建一个独立的线程池，这样某个依赖服务出现延迟过高的情况，也只是对该依赖服务的调用产生影响， 而不会拖慢其他的依赖服务。commandProperties属性：获取线程本地上下文，使用不同的隔离策略将Hystrix切换为使用与调用者相同的线程

   ~~~java
   @HystrixCommand(fallbackMethod = "findByIdFallback", commandProperties = @HystrixProperty(name = "execution.isolation.strategy", value = "SEMAPHORE"))
   ~~~

   1.  THREAD：（默认）线程隔离策略，使用一个隔离的线程（不同的线程）

   2.  SEMAPHOPE：信号量，使用请求的线程，并且并发请求会受到信号量的限制（同一个线程）

4. 关键参数

   Hystrix提供几个熔断关键参数：滑动窗口大小（20）、 熔断器开关间隔（5s）、错误率（50%）

   1. 每当20个请求中，有50%失败时，熔断器就会打开，此时再调用此服务，将会直接返回失败，不再调远程服务。

   2. 直到5s钟之后，重新检测该触发条件，判断是否把熔断器关闭，或者继续打开（分流：用少数流量请求失败的服务，查看服务状态）。

### Dashboard与Turbine

使用Turbine模拟Dashboard监控集群

![](hystix\hystix01.png)

![](hystix\hystix02.png)

### 参考资料

1. [服务熔断、服务降级](https://blog.csdn.net/guwei9111986/article/details/51649240)