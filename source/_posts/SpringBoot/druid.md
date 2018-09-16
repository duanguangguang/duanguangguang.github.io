---
title: Spring Boot（二十四）：springBoot集成Druid
date: 2018-09-09 17:00:06
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

Druid为监控而生的数据库连接池，主要功能对比：

|                          | Druid | BoneCP | DBCP | C3P0 | Proxool | JBoss | Tomcat-Jdbc |
| ------------------------ | ----- | ------ | ---- | ---- | ------- | ----- | ----------- |
| LUR                      | √     |        | √    |      | √       | √     | ？          |
| PSCache                  | √     | √      | √    | √    |         |       | √           |
| PSCache-Oracle-Optimized | √     |        |      |      |         |       |             |
| ExceptionSorter          | √     |        |      |      |         | √     |             |
| 更新维护                 | √     |        |      |      |         | ？    | √           |

[文档](https://github.com/alibaba/druid)

<!-- more -->

pom文件引入：

~~~java
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid</artifactId>
	<version>1.0.26</version>
</dependency>
~~~

Druid配置：

~~~java
@Configuration
public class DruidConfiguration {
	
    @ConditionalOnClass(DruidDataSource.class)
    @ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.alibaba.druid.pool.DruidDataSource", matchIfMissing = true)
    static class Druid extends DruidConfiguration {
        @Bean
        @ConfigurationProperties("spring.datasource.druid")
        public DruidDataSource dataSource(DataSourceProperties properties) {
            DruidDataSource druidDataSource = (DruidDataSource) properties.initializeDataSourceBuilder().type(DruidDataSource.class).build();
            DatabaseDriver databaseDriver = DatabaseDriver.fromJdbcUrl(properties.determineUrl());
            String validationQuery = databaseDriver.getValidationQuery();
            if (validationQuery != null) {
                druidDataSource.setValidationQuery(validationQuery);
            }
            return druidDataSource;
        }
	}
}
~~~

监控：

1. 配置servlet

   ~~~java
   @WebServlet(urlPatterns = { "/druid/*" }, initParams = { @WebInitParam(name = "loginUsername", value = "dodd"), @WebInitParam(name = "loginPassword", value = "dodd") })
   public class DruidStatViewServlet extends StatViewServlet {
   	private static final long serialVersionUID = 1L;
   }
   ~~~

2. filter

   ~~~java
   @WebFilter(filterName = "druidWebStatFilter", urlPatterns = "/*", initParams = { @WebInitParam(name = "exclusions", value = "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*") })
   public class DruidWebStatFilter extends WebStatFilter {
       
   }
   ~~~

配置文件：

~~~java
# mysql
spring.datasource.url=
spring.datasource.username=
spring.datasource.password=
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
#spring.datasource.type=org.apache.tomcat.jdbc.pool.DataSource
#初始化连接大小
spring.datasource.druid.initial-size=8
#最小空闲连接数
spring.datasource.druid.min-idle=5
#最大连接数
spring.datasource.druid.max-active=10
#查询超时时间
spring.datasource.druid.query-timeout=6000
#事务查询超时时间
spring.datasource.druid.transaction-query-timeout=6000
#关闭空闲连接超时时间
spring.datasource.druid.remove-abandoned-timeout=1800
~~~

测试：[http://localhost:8080/druid/index.html ](http://localhost:8080/druid/index.html )

sql监控配置：

~~~java
#filter类名:stat,config,encoding,logging
spring.datasource.druid.filter-class-names=stat
spring.datasource.druid.filters=stat,config
~~~

spring监控配置：

~~~java
@ImportResource(locations = { "classpath:druid-bean.xml" })
~~~

- druid-bean.xml

  ~~~xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
  	xmlns:aop="http://www.springframework.org/schema/aop" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  	xsi:schemaLocation="
          http://www.springframework.org/schema/beans
          http://www.springframework.org/schema/beans/spring-beans.xsd
  		http://www.springframework.org/schema/aop
  		http://www.springframework.org/schema/aop/spring-aop.xsd">
  
  	<!-- 配置_Druid和Spring关联监控配置 -->
  	<bean id="druid-stat-interceptor"
  		class="com.alibaba.druid.support.spring.stat.DruidStatInterceptor"></bean>
  
  	<!-- 方法名正则匹配拦截配置 -->
  	<bean id="druid-stat-pointcut" class="org.springframework.aop.support.JdkRegexpMethodPointcut"
  		scope="prototype">
  		<property name="patterns">
  			<list>
  				<value>com.dodd.mapper.*</value>
  			</list>
  		</property>
  	</bean>
  
  	<aop:config proxy-target-class="true">
  		<aop:advisor advice-ref="druid-stat-interceptor"
  			pointcut-ref="druid-stat-pointcut" />
  	</aop:config>
  
  </beans>
  ~~~
