---
title: Spring Boot（八）：Servlets、Filters、listeners
date: 2018-08-31 16:48:16
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

Web 开发使用 Controller 基本上可以完成大部分需求， 但是我们还可能会用到 Servlet、 Filter、 Listener 等，并且在 spring boot 中的三种实现方式  。

<!-- more -->

### 方法一：注册Bean

自定义 servlet  

~~~java
public class CustomServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        System.out.println("servlet get method");
        doPost(request, response);
    }
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        System.out.println("servlet post method");
        response.getWriter().write("hello world");
    }
}
~~~

自定义 filter  

~~~java
public class CustomFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("init filter");
    }
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)throws IOException, ServletException {
        System.out.println("do filter");
        chain.doFilter(request, response);
    }
    @Override
    public void destroy() {
        System.out.println("destroy filter");
    }
}
~~~

自定义 listener  

~~~java
public class CustomListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("contextInitialized");
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("contextDestroyed");
    }
}
~~~

注册 bean  

~~~java
@SpringBootApplication
public class SpringbootApplication {
    private static Logger logger = LoggerFactory.getLogger(SpringbootApplication.class);
    @Bean
    public ServletRegistrationBean servletRegistrationBean() {
        return new ServletRegistrationBean(new CustomServlet(), "/dodd");
    }
    @Bean
    public FilterRegistrationBean filterRegistrationBean() {
        // servletRegistrationBean 配置的拦截的路径
        return new FilterRegistrationBean(new CustomFilter(), servletRegistrationBean());
    }

    @Bean
    public ServletListenerRegistrationBean<CustomListener> servletListenerRegistrationBean() {
        return new ServletListenerRegistrationBean<CustomListener>(new CustomListener());
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringbootApplication.class, args);
        logger.info("SpringbootApplication start successed>>>");
    }
}
~~~

### 方法二：通过实现 ServletContextInitializer 接口直接注册

~~~java
@SpringBootApplication
public class SpringbootApplication implements ServletContextInitializer {
    private static Logger logger = LoggerFactory.getLogger(SpringbootApplication.class);
    
    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        servletContext.addServlet("customServlet", new CustomServlet()).addMapping("/dodd");
        servletContext.addFilter("customFilter", new CustomFilter())
                .addMappingForServletNames(EnumSet.of(DispatcherType.REQUEST), true, "customServlet");
        servletContext.addListener(new CustomListener());
    }

    public static void main(String[] args) {
        SpringApplication.run(SpringbootApplication.class, args);
        logger.info("SpringbootApplication start successed>>>");
    }
}
~~~

### 方法三：注解方式

在 SpringBootApplication 上使用`@ServletComponentScan `注解后，直接通过`@WebServlet`、 `@WebFilter`、 `@WebListener `注解自动注册  

`@ServletComponentScan `

~~~java
@ServletComponentScan
@SpringBootApplication
public class SpringbootApplication {
    private static Logger logger = LoggerFactory.getLogger(SpringbootApplication.class);
    
    public static void main(String[] args) {
        SpringApplication.run(SpringbootApplication.class, args);
        logger.info("SpringbootApplication start successed>>>");
    }
}
~~~

`@WebServlet`

~~~java
@WebServlet(name = "customServlet", urlPatterns = "/dodd")
public class CustomServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    
    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        System.out.println("servlet get method");
        doPost(request, response);
    }
    
    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        System.out.println("servlet post method");
        response.getWriter().write("hello world");
    }
}
~~~

`@WebFilter`

~~~java
@WebFilter(filterName = "customFilter", urlPatterns = "/*")
public class CustomFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("init filter");
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        System.out.println("do filter");
        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {
        System.out.println("destroy filter");
    }
}
~~~

`@WebListener `

~~~java
@WebListener
public class CustomListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("contextInitialized");
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("contextDestroyed");
    }
}
~~~

