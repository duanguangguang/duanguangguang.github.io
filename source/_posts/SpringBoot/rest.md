---
title: Spring Boot（二十）：调用REST服务
date: 2018-09-09 15:08:16
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

pom文件引入：

~~~java
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
</dependency>
~~~

<!-- more -->

代码实现：

~~~java
@RestController
@RequestMapping(value = "/rest", method = RequestMethod.POST)
public class RestController {

	@Autowired
	private UserLogCache userLogCache;

	@RequestMapping(value = "/update")
	public UserLog update(@RequestBody JsonNode jsonNode) {
		System.out.println("jsonNode=" + jsonNode);
		UserLog bean = userLogCache.selectById(jsonNode.get("id").asInt(1));
		if(bean == null){
			bean = new UserLog();
		}
		bean.setUserName("测试");
		bean.setCreateTime(new Date());
		bean.setUserIp("192.168.1.1");
		userLogCache.updateById(bean);
		return bean;
	}

	@RequestMapping(value = "/update/{id}", method = RequestMethod.GET)
	public UserLog update2(@PathVariable(value = "id") Integer id) {
		UserLog bean = userLogCache.selectById(id);
		if(bean == null){
			bean = new UserLog();
		}
		bean.setUserName("测试");
		bean.setCreateTime(new Date());
		bean.setUserIp("192.168.1.1");
		userLogCache.updateById(bean);
		return bean;
	}

}
~~~

测试：

~~~java
@Autowired
//springboot会自动注入RestTemplateBuilder
private RestTemplateBuilder restTemplateBuilder;

/**
  * get请求
  */
@Test
public void getForObject() {
    UserLog bean = 			restTemplateBuilder.build().getForObject("http://localhost:8080/rest/update/{id}", UserLog.class, 6);
    System.out.println(bean);
    Map<String,Object> map = new HashMap<String,Object>();
    map.put("id", 7);
    bean=restTemplateBuilder.build().postForObject("http://localhost:8080/rest/update", 													map, UserLog.class);
    System.out.println(bean);
}
~~~

### 代理调用

实现类：

~~~java
class ProxyCustomizer implements RestTemplateCustomizer {
        @Override
        public void customize(RestTemplate restTemplate) {
            String proxyHost = "119.29.177.120"; //代理ip
            int proxyPort = 1080;                //代理端口

            HttpHost proxy = new HttpHost(proxyHost, proxyPort);
            HttpClient httpClient = HttpClientBuilder.create().setRoutePlanner(new DefaultProxyRoutePlanner(proxy) {
                @Override
                public HttpHost determineProxy(HttpHost target, HttpRequest request, HttpContext context) throws HttpException {
                    System.out.println(target.getHostName());//对hostneme做过滤
                    return super.determineProxy(target, request, context);
                }
            }).build();
            HttpComponentsClientHttpRequestFactory httpComponentsClientHttpRequestFactory = new HttpComponentsClientHttpRequestFactory(httpClient);
            // 连接超时时间
            httpComponentsClientHttpRequestFactory.setConnectTimeout(10000);
            // 读超时时间
            httpComponentsClientHttpRequestFactory.setReadTimeout(60000);
            restTemplate.setRequestFactory(httpComponentsClientHttpRequestFactory);
        }
    }
~~~

测试：

~~~java
@Test
public void getForObject() {
    String result = restTemplateBuilder.additionalCustomizers(new ProxyCustomizer()).build().getForObject("http://www.baidu.com", String.class);
    System.out.println(result);
}
~~~

在线代理：`http://ip.zdaye.com/`

