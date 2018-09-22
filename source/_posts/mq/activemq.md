---
title: JMS之ActiveMQ
date: 2018-09-22 09:41:46
tags:
 - ActiveMQ
categories: 
 - 消息队列
---

### 介绍

**JMS**：**Java消息服务**（**Java Message Service**，**JMS**）

- Java消息服务是一个与具体平台无关的API；
- Java消息服务的规范包括两种消息模式，点对点和发布者／订阅者；
- Java消息服务支持同步和异步的消息处理；
- Java消息服务支持面向事件的方法接收消息，[事件驱动的程序设计](https://zh.wikipedia.org/wiki/%E4%BA%8B%E4%BB%B6%E9%A9%85%E5%8B%95%E7%A8%8B%E5%BC%8F%E8%A8%AD%E8%A8%88)。

**ActiveMQ**：**Apache ActiveMQ**

- 开源[消息中间件](https://zh.wikipedia.org/w/index.php?title=%E6%B6%88%E6%81%AF%E4%B8%AD%E9%96%93%E4%BB%B6&action=edit&redlink=1)；
- 纯Java程序
- ActiveMQ管理控制台：http://127.0.0.1:8161/admin/ (admin/admin)

<!-- more -->

### 点对点消息实现

消息生产者：

~~~java
public class Producer {

    private static final String USERNAME=ActiveMQConnection.DEFAULT_USER; // 默认的连接用户名
    private static final String PASSWORD=ActiveMQConnection.DEFAULT_PASSWORD; // 默认的连接密码
    private static final String BROKEURL=ActiveMQConnection.DEFAULT_BROKER_URL; // 默认的连接地址
    private static final int SENDNUM=10; // 发送的消息数量

    public static void main(String[] args) {
        ConnectionFactory connectionFactory; // 连接工厂
        Connection connection = null; // 连接
        Session session; // 会话 接受或者发送消息的线程
        Destination destination; // 消息的目的地
        MessageProducer messageProducer; // 消息生产者

        // 实例化连接工厂
        connectionFactory=new ActiveMQConnectionFactory(Producer.USERNAME, Producer.PASSWORD, Producer.BROKEURL);

        try {
            connection=connectionFactory.createConnection(); // 通过连接工厂获取连接
            connection.start(); // 启动连接
            session=connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE); // 创建Session, true表示开始事物
            destination=session.createQueue("FirstQueue"); // 创建消息队列
            messageProducer=session.createProducer(destination); // 创建消息生产者
            sendMessage(session, messageProducer); // 发送消息
            session.commit();
        } catch (Exception e) {
            e.printStackTrace();
        } finally{
            if(connection!=null){
                try {
                    connection.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 发送消息
     * @param session
     * @param messageProducer
     * @throws Exception
     */
    public static void sendMessage(Session session,MessageProducer messageProducer)throws Exception{
        for(int i=0;i<Producer.SENDNUM;i++){
            TextMessage message=session.createTextMessage("ActiveMQ 发送的消息"+i);
            System.out.println("发送消息："+"ActiveMQ 发送的消息"+i);
            messageProducer.send(message);
        }
    }
}
~~~

消息消费方式一：Receive

~~~java
public class ConsumerReceive {
    private static final String USERNAME=ActiveMQConnection.DEFAULT_USER; // 默认的连接用户名
    private static final String PASSWORD=ActiveMQConnection.DEFAULT_PASSWORD; // 默认的连接密码
    private static final String BROKEURL=ActiveMQConnection.DEFAULT_BROKER_URL; // 默认的连接地址

    public static void main(String[] args) {
        ConnectionFactory connectionFactory; // 连接工厂
        Connection connection = null; // 连接
        Session session; // 会话 接受或者发送消息的线程
        Destination destination; // 消息的目的地
        MessageConsumer messageConsumer; // 消息的消费者

        // 实例化连接工厂
        connectionFactory=new ActiveMQConnectionFactory(ConsumerReceive.USERNAME, ConsumerReceive.PASSWORD, ConsumerReceive.BROKEURL);

        try {
            connection=connectionFactory.createConnection();  // 通过连接工厂获取连接
            connection.start(); // 启动连接
            session=connection.createSession(Boolean.FALSE, Session.AUTO_ACKNOWLEDGE); // 创建Session
            destination=session.createQueue("FirstQueue");  // 创建连接的消息队列
            messageConsumer=session.createConsumer(destination); // 创建消息消费者
            while(true){
                TextMessage textMessage=(TextMessage)messageConsumer.receive(100000);
                if(textMessage!=null){
                    System.out.println("收到的消息："+textMessage.getText());
                }else{
                    break;
                }
            }
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
~~~

- **Session.AUTO_ACKNOWLEDGE**：当客户成功的从receive方法返回的时候，或者从MessageListener.onMessage 方法成功返回的时候，会话自动确认客户收到的消息。
- **Session.CLIENT_ACKNOWLEDGE**： 客户通过消息的 acknowledge 方法确认消息。需要注意的是，在这种模 式中，确认是在会话层上进行：确认一个被消费的消息将自动确认所有已被会话消 费的消息。例如，如果一 个消息消费者消费了 10 个消息，然后确认第 5 个消息，那么所有 10 个消息都被确认。  
- **Session.DUPS_ACKNOWLEDGE** ： 该选择只是会话迟钝第确认消息的提交。如果 JMS provider 失败，那么可 能会导致一些重复的消息。如果是重复的消息，那么 JMS provider 必须把消息头的 JMSRedelivered 字段设置 为 true。  

消息消费方式二：Listener

~~~java
public class ConsumerListener {
    private static final String USERNAME=ActiveMQConnection.DEFAULT_USER; // 默认的连接用户名
    private static final String PASSWORD=ActiveMQConnection.DEFAULT_PASSWORD; // 默认的连接密码
    private static final String BROKEURL=ActiveMQConnection.DEFAULT_BROKER_URL; // 默认的连接地址

    public static void main(String[] args) {
        ConnectionFactory connectionFactory; // 连接工厂
        Connection connection = null; // 连接
        Session session; // 会话 接受或者发送消息的线程
        Destination destination; // 消息的目的地
        MessageConsumer messageConsumer; // 消息的消费者

        // 实例化连接工厂
        connectionFactory=new ActiveMQConnectionFactory(ConsumerListener.USERNAME, ConsumerListener.PASSWORD, ConsumerListener.BROKEURL);

        try {
            connection=connectionFactory.createConnection();  // 通过连接工厂获取连接
            connection.start(); // 启动连接
            session=connection.createSession(Boolean.FALSE, Session.AUTO_ACKNOWLEDGE); // 创建Session
            destination=session.createQueue("FirstQueue");  // 创建连接的消息队列
            messageConsumer=session.createConsumer(destination); // 创建消息消费者
            messageConsumer.setMessageListener(new Listener()); // 注册消息监听
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
~~~

Listener方式消息监听：

~~~java
public class Listener implements MessageListener {
    @Override
    public void onMessage(Message message) {
        try {
            System.out.println("收到的消息："+((TextMessage)message).getText());
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
~~~

结果：

![](activemq\mq02.png)

### 发布-订阅消息实现

先订阅才能收到发布的消息。

消息发布者：

~~~java
public class ProfucerIssue {
    private static final String USERNAME=ActiveMQConnection.DEFAULT_USER; // 默认的连接用户名
    private static final String PASSWORD=ActiveMQConnection.DEFAULT_PASSWORD; // 默认的连接密码
    private static final String BROKEURL=ActiveMQConnection.DEFAULT_BROKER_URL; // 默认的连接地址
    private static final int SENDNUM=10; // 发送的消息数量

    public static void main(String[] args) {

        ConnectionFactory connectionFactory; // 连接工厂
        Connection connection = null; // 连接
        Session session; // 会话 接受或者发送消息的线程
        Destination destination; // 消息的目的地
        MessageProducer messageProducer; // 消息生产者

        // 实例化连接工厂
        connectionFactory=new ActiveMQConnectionFactory(ProfucerIssue.USERNAME, ProfucerIssue.PASSWORD, ProfucerIssue.BROKEURL);

        try {
            connection=connectionFactory.createConnection(); // 通过连接工厂获取连接
            connection.start(); // 启动连接
            session=connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE); // 创建Session
            destination=session.createTopic("FirstTopic"); // 创建消息队列
            messageProducer=session.createProducer(destination); // 创建消息生产者
            sendMessage(session, messageProducer); // 发送消息
            session.commit();
        } catch (Exception e) {
            e.printStackTrace();
        } finally{
            if(connection!=null){
                try {
                    connection.close();
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 发送消息
     * @param session
     * @param messageProducer
     * @throws Exception
     */
    public static void sendMessage(Session session,MessageProducer messageProducer)throws Exception{
        for(int i=0;i<ProfucerIssue.SENDNUM;i++){
            TextMessage message=session.createTextMessage("ActiveMQ 发布的消息"+i);
            System.out.println("发送消息："+"ActiveMQ 发布的消息"+i);
            messageProducer.send(message);
        }
    }
}
~~~

消息订阅者一：

~~~java
public class ConsumerSubscribe1 {

    private static final String USERNAME=ActiveMQConnection.DEFAULT_USER; // 默认的连接用户名
    private static final String PASSWORD=ActiveMQConnection.DEFAULT_PASSWORD; // 默认的连接密码
    private static final String BROKEURL=ActiveMQConnection.DEFAULT_BROKER_URL; // 默认的连接地址

    public static void main(String[] args) {
        ConnectionFactory connectionFactory; // 连接工厂
        Connection connection = null; // 连接
        Session session; // 会话 接受或者发送消息的线程
        Destination destination; // 消息的目的地
        MessageConsumer messageConsumer; // 消息的消费者

        // 实例化连接工厂
        connectionFactory=new ActiveMQConnectionFactory(ConsumerSubscribe1.USERNAME, ConsumerSubscribe1.PASSWORD, ConsumerSubscribe1.BROKEURL);

        try {
            connection=connectionFactory.createConnection();  // 通过连接工厂获取连接
            connection.start(); // 启动连接
            session=connection.createSession(Boolean.FALSE, Session.AUTO_ACKNOWLEDGE); // 创建Session
            destination=session.createTopic("FirstTopic"); // 创建连接的消息队列
            messageConsumer=session.createConsumer(destination); // 创建消息消费者
            messageConsumer.setMessageListener(new Listener1()); // 注册消息监听
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
~~~

消息订阅者二：

~~~java
public class ConsumerSubscribe2 {

    private static final String USERNAME=ActiveMQConnection.DEFAULT_USER; // 默认的连接用户名
    private static final String PASSWORD=ActiveMQConnection.DEFAULT_PASSWORD; // 默认的连接密码
    private static final String BROKEURL=ActiveMQConnection.DEFAULT_BROKER_URL; // 默认的连接地址

    public static void main(String[] args) {
        ConnectionFactory connectionFactory; // 连接工厂
        Connection connection = null; // 连接
        Session session; // 会话 接受或者发送消息的线程
        Destination destination; // 消息的目的地
        MessageConsumer messageConsumer; // 消息的消费者

        // 实例化连接工厂
        connectionFactory=new ActiveMQConnectionFactory(ConsumerSubscribe2.USERNAME, ConsumerSubscribe2.PASSWORD, ConsumerSubscribe2.BROKEURL);

        try {
            connection=connectionFactory.createConnection();  // 通过连接工厂获取连接
            connection.start(); // 启动连接
            session=connection.createSession(Boolean.FALSE, Session.AUTO_ACKNOWLEDGE); // 创建Session
            destination=session.createTopic("FirstTopic"); // 创建连接的消息队列
            messageConsumer=session.createConsumer(destination); // 创建消息消费者
            messageConsumer.setMessageListener(new Listener2()); // 注册消息监听
        } catch (JMSException e) {
            e.printStackTrace();
        }
    }
}
~~~

结果：

![](activemq\mq01.png)



