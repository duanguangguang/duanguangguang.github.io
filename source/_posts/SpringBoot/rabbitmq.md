---
title: Spring Boot（十九）：异步消息AMQP（RabbitMQ）
date: 2018-09-02 19:05:24
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
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
~~~

注解启用：@EnableRabbit

<!-- more -->

AmqpConfiguration：

~~~java
import org.springframework.amqp.core.Queue;

@Configuration
public class AmqpConfiguration {
    @Bean
    public Queue queue() {
        return new Queue("dodd.queue");
    }
}
~~~

AmqpComponent：

~~~java
@Component
public class AmqpComponent {
    @Autowired
    private AmqpTemplate amqpTemplate;

    public void send(String msg) {
        this.amqpTemplate.convertAndSend("dodd.queue", msg);
    }

    @RabbitListener(queues = "dodd.queue")
    public void receiveQueue(String text) {
        System.out.println("接受到：" + text);
    }
}
~~~

测试类：

~~~java
@Autowired
private AmqpComponent amqpComponent;

@Test
public void send() {
    amqpComponent.send("hello world");
}
~~~