---
title: Spring Boot（六）：多环境配置
date: 2018-09-01 13:46:46
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

在开发应用时，常用部署的应用是多个的，比如：开发、测试、联调、生产等不同的应用环境，这些应用环境都对应不同的配置项，比如swagger一般上在生产时是关闭的；不同环境数据库地址、端口号等都是不尽相同的，要是没有多环境的自由切换，部署起来是很繁琐也容易出错的。

<!-- more-->

### maven多环境配置

maven配置：

~~~java
<profiles>
    <profile>
       <id>dev</id>
       <activation>
             <activeByDefault>true</activeByDefault>
         </activation>
         <properties>
            <pom.port>8080</pom.port>
         </properties>
    </profile>
    <profile>
       <id>test</id>
         <properties>
            <pom.port>8888</pom.port>
         </properties>
    </profile>       
 </profiles>
 <build>
 <resources>
     <resource>
         <directory>src/main/resources</directory>
         <includes>
             <include>**/*</include>
         </includes>
     </resource>
     <resource>
         <directory>${project.basedir}/src/main/resources</directory>
         <includes>
             <include>**/*.properties</include>
         </includes>
         <!-- 加入此属性，才会进行过滤 -->
         <filtering>true</filtering>
     </resource>
 </resources>
 <plugins>
     <plugin>
         <artifactId>maven-resources-plugin</artifactId>
         <configuration>
             <encoding>utf-8</encoding>
             <!-- 需要加入，因为maven默认的是${},而springbooot 默认会把此替换成@{} -->
             <useDefaultDelimiters>true</useDefaultDelimiters>
         </configuration>
     </plugin>
     <plugin>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-maven-plugin</artifactId>
     </plugin>
 </plugins>
 </build>
~~~

然后编译时，加入-Ptest，则会替换test环境下的参数值。 完整参数：

~~~java
mvn clean install -DskipTests -Ptest
~~~

application.properties：

~~~java
server.port=${pom.port}
~~~

利用maven实现多环境配置，比较麻烦的就是每次部署新环境时，都需要再次指定环境编译打包一次。

### springboot多环境配置

Profile是Spring针对不同环境不同配置的支持。需要满足application-{profile}.properties，{profile}对应你的环境标识。如： 

- application-dev.properties：开发环境
- application-test.properties：测试环境

![](environment\environment01.png)

而指定执行哪份配置文件，只需要在application.properties配置spring.profiles.active为对应${profile}的值。

~~~java
# 指定环境为dev
spring.profiles.active=dev
~~~

还可以在命令行方式激活不同环境配置，如：

~~~java
java -jar xxx.jar --spring.profiles.active=test
~~~

在不同环境下，可能加载不同的bean时，可利用@Profile注解来动态激活：

~~~java
@Profile("dev")//支持数组:@Profile({"dev","test"})
@Configuration
@Slf4j
public class ProfileBean {
 
    @PostConstruct
    public void init() {
        log.info("dev环境下激活");
    }    
}
~~~

