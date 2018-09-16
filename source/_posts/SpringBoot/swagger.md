---
title: Spring Boot（二十五）：springBoot集成Swagger
date: 2018-09-09 17:01:42
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

**Swagger** 是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务。

 http://swagger.io/ 

**Springfox** 的前身是 swagger-springmvc，是一个开源的 API doc 框架，可以将我们的 Controller 的 方法以文档的形式展现，基于 Swagger。 

http://springfox.github.io/springfox/  

<!--more -->

pom文件引入：

~~~java
<!-- Swagger -->
<dependency>
	<groupId>io.springfox</groupId>
	<artifactId>springfox-swagger-ui</artifactId>
	<version>2.6.0</version>
</dependency>
<dependency>
	<groupId>io.springfox</groupId>
	<artifactId>springfox-swagger2</artifactId>
	<version>2.6.0</version>
</dependency>
~~~

Swagger配置类：

~~~java
@Configuration
@EnableSwagger2
public class Swagger2Configuration {
	@Bean
	public Docket accessToken() {
		return new Docket(DocumentationType.SWAGGER_2).groupName("api")// 定义组
				.select() // 选择那些路径和api会生成document
				.apis(RequestHandlerSelectors.basePackage("com.dodd.controller")) // 拦截的包路径
				.paths(regex("/api/.*"))// 拦截的接口路径
				.build() // 创建
				.apiInfo(apiInfo()); // 配置说明
	}

	private ApiInfo apiInfo() {
		return new ApiInfoBuilder()//
				.title("dodd")// 标题
				.description("spring boot swagger")// 描述
				.termsOfServiceUrl("http://www.baidu.com")//
				.contact(new Contact("dodd", "http://www.baidu.com", "1234567@qq.com"))// 联系
				//.license("Apache License Version 2.0")// 开源协议
				//.licenseUrl("https://github.com/springfox/springfox/blob/master/LICENSE")// 地址
				.version("1.0")// 版本
				.build();
	}
}
~~~

自定义注解的使用：

1. @ApiIgnore 

   忽略暴露的 api 。例：

   ~~~java
   @ApiIgnore
   @RequestMapping(value = "/delete", method = RequestMethod.GET)
   public int delete(@RequestParam(defaultValue = "1") Integer id) {
       return doddMappper.deleteByPrimaryKey(id);
   }
   ~~~

2. @ApiOperation(value = "查找", notes = "根据用户 ID 查找用户")

    添加说明  。例：

   ~~~java
   @ApiOperation(value = "查找", notes = "根据用户ID查找用户")
   @RequestMapping(value = "/select", method = RequestMethod.GET)
   public RoncooUser get(@RequestParam(defaultValue = "1") Integer id) {
       return doddMappper.selectByPrimaryKey(id);
   }
   ~~~

其他注解：

1. @Api

    用在类上，说明该类的作用 

2. @ApiImplicitParams

    用在方法上包含一组参数说明 

3. @ApiResponses

    用于表示一组响应  

4. @ApiResponse

    用在@ApiResponses 中，一般用于表达一个错误的响应信息 code：数字，例如 400 message：信息，例如"请求参数没填好" response：抛出异常的类 

5. @ApiModel

    描述一个 Model 的信息（这种一般用在 post 创建的时候，使用@RequestBody 这样的场 景，请求参数无法使用@ApiImplicitParam 注解进行描述的时候） 

6. @ApiModelProperty

    描述一个 model 的属性  

[更多](https://github.com/swagger-api/swagger-core/wiki/Annotations  )