---
title: Spring Boot（二十一）：发送邮件（多账号轮询发送）
date: 2018-09-09 15:18:12
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
	<artifactId>spring-boot-starter-mail</artifactId>
</dependency>
~~~

<!-- more -->

配置：

~~~java
spring.mail.host = 
spring.mail.username = 
spring.mail.password =
spring.mail.properties.mail.smtp.auth = true
~~~

多账号实现：

~~~java
@Configuration
@EnableConfigurationProperties(MailProperties.class)
public class DoddJavaMailSenderImpl extends JavaMailSenderImpl implements JavaMailSender {

    private ArrayList<String> usernameList;
    private ArrayList<String> passwordList;
    private int currentMailId = 0;

    private final MailProperties properties;

    public DoddJavaMailSenderImpl(MailProperties properties) {
        this.properties = properties;

        // 初始化账号
        if (usernameList == null)
            usernameList = new ArrayList<String>();
        String[] userNames = this.properties.getUsername().split(",");
        if (userNames != null) {
            for (String user : userNames) {
                usernameList.add(user);
            }
        }

        // 初始化密码
        if (passwordList == null)
            passwordList = new ArrayList<String>();
        String[] passwords = this.properties.getPassword().split(",");
        if (passwords != null) {
            for (String pw : passwords) {
                passwordList.add(pw);
            }
        }
    }

    @Override
    protected void doSend(MimeMessage[] mimeMessage, Object[] object) throws MailException {

        super.setUsername(usernameList.get(currentMailId));
        super.setPassword(passwordList.get(currentMailId));

        // 设置编码和各种参数
        super.setHost(this.properties.getHost());
        super.setDefaultEncoding(this.properties.getDefaultEncoding().name());
        super.setJavaMailProperties(asProperties(this.properties.getProperties()));
        super.doSend(mimeMessage, object);

        // 轮询
        currentMailId = (currentMailId + 1) % usernameList.size();
    }

    private Properties asProperties(Map<String, String> source) {
        Properties properties = new Properties();
        properties.putAll(source);
        return properties;
    }

    @Override
    public String getUsername() {
        return usernameList.get(currentMailId);
    }

}
~~~

实现类：

~~~java
@Component
public class DoddJavaMailComponent {
    private static final String template = "mail/dodd.ftl";

    @Autowired
    private FreeMarkerConfigurer freeMarkerConfigurer;
    @Autowired
    private DoddJavaMailSenderImpl javaMailSender;

    public void sendMail(String email) {
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("email", email);
        try {
            String text = getTextByTemplate(template, map);
            send(email, text);
        } catch (IOException | TemplateException e) {
            e.printStackTrace();
        } catch (MessagingException e) {
            e.printStackTrace();
        }
    }

    //使用模板方式
    private String getTextByTemplate(String template, Map<String, Object> model) throws TemplateNotFoundException, MalformedTemplateNameException, ParseException, IOException, TemplateException {
        return FreeMarkerTemplateUtils.processTemplateIntoString(freeMarkerConfigurer.getConfiguration().getTemplate(template), model);
    }

    private String send(String email, String text) throws MessagingException, UnsupportedEncodingException {
        MimeMessage message = javaMailSender.createMimeMessage();
        MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
        InternetAddress from = new InternetAddress();
        from.setAddress(javaMailSender.getUsername());
        from.setPersonal("Dodd", "UTF-8");
        helper.setFrom(from);
        helper.setTo(email);
        helper.setSubject("测试邮件");
        helper.setText(text, true);
        javaMailSender.send(message);
        return text;
    }

}
~~~

controller：

~~~java
@RestController
@RequestMapping("/api")
public class MailController {

    @Autowired
    private DoddJavaMailComponent component;

    @RequestMapping(value = "mail")
    public String mail(String email) {
        component.sendMail(email);
        return "success";
    }
}
~~~

