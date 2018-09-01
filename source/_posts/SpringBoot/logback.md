---
title: Spring Boot（三）：日志配置详解
date: 2018-08-25 15:29:50
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

Spring Boot在所有内部日志中使用[Commons Logging](http://commons.apache.org/proper/commons-logging/)，但是默认配置也提供了对常用日志的支持，如：[Java Util Logging](http://docs.oracle.com/javase/7/docs/api/java/util/logging/package-summary.html)，[Log4J](http://logging.apache.org/log4j/), [Log4J2](http://logging.apache.org/log4j/)和[Logback](http://logback.qos.ch/)。每种Logger都可以通过配置使用控制台或者文件输出日志内容。 日志格式如下：

![](logback\logback01.png)

从上图可以看到，日志输出内容元素具体如下： 

- 时间日期 — 精确到毫秒
- 日志级别 — ERROR, WARN, INFO, DEBUG or TRACE
- 进程ID
- 分隔符 — `---` 标识实际日志的开始
- 线程名 — 方括号括起来（可能会截断控制台输出）
- Logger名 — 通常使用源代码的类名
- 日志内容

<!-- more -->

### 日志系统分类

1. [SLF4J](http://www.slf4j.org/):Simple Logging Facade For Java，它是一个针对于各类Java日志框架的统一[Facade](https://en.wikipedia.org/wiki/Facade_pattern)抽象。Java日志框架众多——常用的有`java.util.logging`, `log4j`, `logback`，`commons-logging`, Spring框架使用的是`Jakarta Commons Logging API (JCL)`。而SLF4J定义了统一的日志抽象接口，而真正的日志实现则是在运行时决定的，它提供了各类日志框架的binding。 
2. [Logback](http://logback.qos.ch/)是log4j框架的作者开发的新一代日志框架，它效率更高、能够适应诸多的运行环境，同时天然支持SLF4J。 

默认情况下，spring boot默认会加载`classpath:logback-spring.xml`或者`classpath:logback-spring.groovy`，Spring Boot会用Logback来记录日志，并用INFO级别输出到控制台。 

### 默认配置logback

1. 添加日志依赖

   spring-boot-starter其中包含了 spring-boot-starter-logging，该依赖内容就是 Spring Boot 默认的日志框架 logback，所以我们不需要直接添加该依赖 。

2. 控制台输出

   日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL ，Spring Boot中默认配置`ERROR`、`WARN`和`INFO`级别的日志输出到控制台。 以下两种方式开启调试模式：

   - 在运行命令后加入`--debug`标志，如：`$ java -jar springTest.jar --debug`。
   - 在`application.properties`中配置`debug=true`，该属性置为true的时候，核心Logger（包含嵌入式容器、hibernate、spring）会输出更多内容，但是你自己应用的日志并不会输出为DEBUG级别。

3. 文件输出

   如果要编写除控制台输出之外的日志文件，则需在application.properties中设置logging.file或logging.path属性：

   - logging.file，设置文件，可以是绝对路径，也可以是相对路径。如：`logging.file=my.log`
   - logging.path，设置目录，会在该目录下创建spring.log文件，并写入日志内容，如：`logging.path=/var/log`

   如果只配置 logging.file，会在项目的当前路径下生成一个 xxx.log 日志文件。 如果只配置 logging.path，在 /var/log文件夹生成一个日志文件为 spring.log ，二者不能同时使用，如若同时使用，则只有logging.file生效 

4. 级别控制

   所有支持的日志记录系统都可以在Spring环境中设置记录级别（例如在application.properties中） 格式为：logging.level.* = LEVEL：

   - `logging.level`：日志级别控制前缀，`*`为包名或Logger名
   - `LEVEL`：选项TRACE, DEBUG, INFO, WARN, ERROR, FATAL, OFF

#### 自定义日志配置

根据不同的日志系统，你可以按如下规则组织配置文件名，就能被正确加载： 

1. Logback：`logback-spring.xml`, `logback-spring.groovy`, `logback.xml`, `logback.groovy`
2. Log4j：`log4j-spring.properties`, `log4j-spring.xml`, `log4j.properties`, `log4j.xml`
3. Log4j2：`log4j2-spring.xml`, `log4j2.xml`
4. JDK (Java Util Logging)：`logging.properties`

**Spring Boot官方推荐优先使用带有-spring的文件名作为你的日志配置（如使用logback-spring.xml，而不是logback.xml），命名为logback-spring.xml的日志配置文件，spring boot可以为它添加一些spring boot特有的配置项。**上面是默认的命名规则，并且放在`src/main/resources`下面即可。自定义日志文件名，可以在`application.properties`配置文件里面通过logging.config属性指定自定义的名字： 

~~~java
logging.config=classpath:logging-config.xml
~~~

####logback-spring.xml 示例

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="60 seconds" debug="false">
	<contextName>logback</contextName>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>
    <!-- 定义项目名称及日志文件的存储地址 注意: 此处需要定义项目名 -->
    <property name="APP_NAME" value="demo"/>
    <!-- 项目日志路径固定为以下路径 -->
    <property name="LOG_HOME" value="/dodd/logs"/>
    
    <!--输出到控制台-->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
       <!-- <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>-->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%date{yyyy-MM-dd HH:mm:ss.SSS} [%+22thread] %-5level %+30logger{5} %n--> %msg%n%n</pattern>
        </encoder>
    </appender>
    
    <!-- 按照每天生成日志文件 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--日志文件输出的文件名 -->
            <FileNamePattern>${LOG_HOME}/${APP_NAME}/${APP_NAME}.%d{yyyy-MM-dd}.log.zip</FileNamePattern>
            <!--日志文件保留天数 -->
            <MaxHistory>15</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%date{yyyy-MM-dd HH:mm:ss.SSS} [%+22thread] %-5level %+30logger{5} %n--> %msg%n%n</pattern>
        </encoder>
    </appender>

    <!-- 按照每天生成日志文件 -->
    <appender name="SQL" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!--日志文件输出的文件名 -->
            <FileNamePattern>${LOG_HOME}/${APP_NAME}/${APP_NAME}.%d{yyyy-MM-dd}.sql.zip</FileNamePattern>
            <!--日志文件保留天数 -->
            <MaxHistory>15</MaxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%date{yyyy-MM-dd HH:mm:ss.SSS} [%+22thread] %-5level %+30logger{5} %n--> %msg%n%n</pattern>
        </encoder>
    </appender>

    <!--项目代码打印INFO级别日志  注意: 此处需要自定义包名-->
    <logger name="com.dodd.demo" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="SQL"/>
        <appender-ref ref="CONSOLE"/>
    </logger>

    <!--其它代码打印WARN级别日志-->
    <root level="WARN">
        <appender-ref ref="FILE"/>
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
~~~

#### 根节点configuration

- scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true
- scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟
- debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false

1. 属性一：设置上下文名称`<contextName>`

   每个logger都关联到logger上下文，默认上下文名称为“default”。但可以使用设置成其他名字，用于区分不同应用程序的记录。一旦设置，不能修改,可以通过%contextName来打印日志上下文名称

2. 属性二：设置变量`<property>`

   用来定义变量值的标签， 有两个属性，name和value；其中name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文中。定义变量后，可以使“${}”来使用变量

#### 子节点一：appender

appender用来格式化日志输出节点，有俩个属性name和class，class用来指定哪种输出策略，常用就是控制台输出策略和文件输出策略。 

`<encoder>`表示对日志进行编码：

- `%d{HH: mm:ss.SSS}`——日志输出时间
- `%thread`——输出日志的进程名字，这在Web应用以及异步任务处理中很有用
- `%-5level`——日志级别，并且使用5个字符靠左对齐
- `%logger{36}`——日志输出者的名字
- `%msg`——日志消息
- `%n`——平台的换行符

ThresholdFilter为系统定义的拦截器，例如我们用ThresholdFilter来过滤掉ERROR级别以下的日志不输出到文件中。如果不用记得注释掉，不然你控制台会发现没日志~

####  子节点二：root

root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性，用来设置打印级别，大小写无关`TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `ALL` 和`OFF`，不能设置为INHERITED或者同义词NULL。 默认是DEBUG

####  子节点三：logger

`<logger>`用来设置某一个包或者具体的某一个类的日志打印级别、以及指定`<appender>`。`<logger>`仅有一个name属性，一个可选的level和一个可选的addtivity属性:

- `name`:用来指定受此logger约束的某一个包或者具体的某一个类
- `level`:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，还有一个特俗值INHERITED或者同义词NULL，代表强制执行上级的级别。如果未设置此属性，那么当前logger将会继承上级的级别
- `addtivity`:是否向上级logger传递打印信息。默认是true。

#### 打印sql语句

使用mybatis的时候，sql语句是debug下才会打印，所以想要查看sql语句的话，有以下两种操作： 

- 第一种把`<root level="info">`改成`<root level="DEBUG">`这样就会打印sql

- 第二种就是单独给dao下目录配置debug模式，代码如下，这样配置sql语句会打印，其他还是正常info级别：

  ~~~xml
  <logger name="com.dodd.dao" level="DEBUG" additivity="false">
      <appender-ref ref="console" />
  </logger>
  ~~~

#### 多环境日志输出

logback-spring.xml（想使用spring扩展profile支持，要以logback-spring.xml命名 ），方法如下： 

~~~java
<!-- 测试环境+开发环境. 多个使用逗号隔开. -->
<springProfile name="test,dev">
    <logger name="com.dodd.controller" level="info" />
</springProfile>
<!-- 生产环境. -->
<springProfile name="prod">
    <logger name="com.dodd.controller" level="ERROR" />
</springProfile>
~~~

根据不同环境（prod:生产环境，test:测试环境，dev:开发环境）来定义不同的日志输出：使用方法如下：

application.properties

~~~java
#配置文件环境配置
spring.profiles.active = dev
#配置成dev则会去找<springProfile name="dev">的环境日志配置
~~~

logback多环境配置示例：

~~~java
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 文件输出格式 -->
    <property name="PATTERN" value="%-12(%d{yyyy-MM-dd HH:mm:ss.SSS}) |-%-5level [%thread] %c [%L] -| %msg%n" />
    <!-- test文件路径 -->
    <property name="TEST_FILE_PATH" value="d:/dodd/springboot/test/logs" />
    <!-- pro文件路径 -->
    <property name="PRO_FILE_PATH" value="/dodd/springboot/pro/logs" />

    <!-- 开发环境 -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>${PATTERN}</pattern>
            </encoder>
        </appender>

        <logger name="springbootdemo.springboot" level="debug"/>

        <root level="info">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>

    <!-- 测试环境 -->
    <springProfile name="test">
        <!-- 每天产生一个文件 -->
        <appender name="TEST-FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <!-- 文件路径 -->
            <file>${TEST_FILE_PATH}</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <!-- 文件名称 -->
                <fileNamePattern>${TEST_FILE_PATH}/info.%d{yyyy-MM-dd}.log</fileNamePattern>
                <!-- 文件最大保存历史数量 -->
                <MaxHistory>100</MaxHistory>
            </rollingPolicy>

            <layout class="ch.qos.logback.classic.PatternLayout">
                <pattern>${PATTERN}</pattern>
            </layout>
        </appender>

        <root level="info">
            <appender-ref ref="TEST-FILE" />
        </root>
    </springProfile>

    <!-- 生产环境 -->
    <springProfile name="prod">
        <appender name="PROD_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>${PRO_FILE_PATH}</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
                <fileNamePattern>${PRO_FILE_PATH}/warn.%d{yyyy-MM-dd}.log</fileNamePattern>
                <MaxHistory>100</MaxHistory>
            </rollingPolicy>
            <layout class="ch.qos.logback.classic.PatternLayout">
                <pattern>${PATTERN}</pattern>
            </layout>
        </appender>

        <root level="warn">
            <appender-ref ref="PROD_FILE" />
        </root>
    </springProfile>
</configuration>
~~~

### log4j2

#### pom引用：

~~~java
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
<!-- 去除logback的依赖包 -->
<exclusions>
	<exclusion>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-logging</artifactId>
	</exclusion>
</exclusions>
~~~

在classpath添加log4j2.xml或者log4j2-spring.xml（spring boot 默认加载）。

#### log4j2多环境配置：

可以结合多环境配置的配置文件，利用`logging.config=classpath:log4j2-dev.xml`，结构如下：

![](logback\log02.png)

application.properties

~~~java
#配置文件环境配置
spring.profiles.active = dev
~~~

application-dev.properties

~~~java
logging.config=classpath:log4j2-dev.xml
~~~

log4j2-dev.xml

~~~java
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <properties>
        <!-- 文件输出格式 -->
        <property name="PATTERN">%d{yyyy-MM-dd HH:mm:ss.SSS} |-%-5level [%thread] %c [%L] -| %msg%n</property>
    </properties>

    <appenders>
        <Console name="CONSOLE" target="system_out">
            <PatternLayout pattern="${PATTERN}" />
        </Console>
    </appenders>

    <loggers>
        <logger name="com.dodd.demo" level="debug" />
        <root level="info">
            <appenderref ref="CONSOLE" />
        </root>
    </loggers>

</configuration>
~~~

### 总结

性能比较：Log4J2 和 Logback 都优于 log4j（不推荐使用）

配置方式：Logback最简洁，spring boot默认，推荐使用