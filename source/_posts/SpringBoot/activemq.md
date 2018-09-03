---
title: Spring Boot（十八）：异步消息JMS（ActiveMQ）
date: 2018-09-02 19:05:04
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
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
~~~

启用JMS注解：`@EnableJms`

<!-- more -->

mq配置：

~~~java
import javax.jms.Queue;

@Configuration
public class JmsConfiguration {
    @Bean
    public Queue queue() {
        return new ActiveMQQueue("dodd.queue");
    }
}
~~~

实现类：

~~~java
@Component
public class JmsComponent {
    @Autowired
    private JmsMessagingTemplate jmsMessagingTemplate;

    @Autowired
    private Queue queue;

    public void send(String msg) {
        this.jmsMessagingTemplate.convertAndSend(this.queue, msg);
    }

    @JmsListener(destination = "dodd.queue")
    public void receiveQueue(String text) {
        System.out.println("接受到：" + text);
    }
}
~~~

配置文件：

~~~java
#放在缓存中，不用连接真正的mq
spring.activemq.in-memory=true
~~~

测试类：

~~~java
@Autowired
private JmsComponent jmsComponent;

@Test
public void send() {
    jmsComponent.send("hello world");
}
~~~

