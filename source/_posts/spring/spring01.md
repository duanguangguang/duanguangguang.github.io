---
title: Spring装配bean
date: 2017-11-25 15:59:41
categories: 
 - JAVA
 - Spring
tags:
 - Spring
---

## 介绍

在Spring中，对象无需自己查找或创建与其所关联的其他对象，容器负责把需要相互协作的对象引用赋予各个对象。创建应用对象之间协作关系的行为通常称为装配（wiring），这也是依赖注入的本质。Spring提供了四种主要的装配机制：

- 隐式的bean发现机制和自动装配
- 在java中进行显示配置（javaConfig）
- 在XML中进行显示配置
- 混合配置

<!-- more -->

## 例子程序中通用的接口

CD接口：<CompactDisc>

~~~java
public interface CompactDisc {
	void play();
}
~~~

CDPlayer接口：<MediaPlayer>

~~~java
public interface MediaPlayer {
	void play();
}
~~~

## 自动化装配bean

Spring从两个角度来实现自动化装配：

- 组件扫描（component scanning）：Spring 会自动发现应用上下文中创建的bean。
- 自动装配（autowiring）：Spring自动满足bean之间的依赖。

带有@Component注解的<CompactDisc>实现类<SgtPeppers>

~~~java
/*
* @Component 
* 1. spring会为所有带@Component注解的类创建为bean
* 2. 为组件扫描的bean命名：默认为类名第一个字母小写。例：sgtPeppers
* 3. 为组件扫描的bean命名：@Component("rename"), @Named("rename")
*/
@Component 
public class SgtPeppers implements CompactDisc{

	private String title = "Lonely Hearts";
	private String artist = "The Beatles";
	
	@Override
	public void play() {
		System.out.println("Playing " + title + "by " + artist);
	}

}
~~~

@ComponentScan注解启用扫描组件

~~~java
@Configuration
/* 
* @ComponentScan
* 1. 启用组件扫描,在spring中组件扫描默认不启用
* 2. 如果没有其他配置，@ComponentScan会默认扫描与配置类相同的包,查找@Component 
* 3. 设置组件扫描基础包：@ComponentScan("spring02.assemble.automatic")
* 3. 设置组件扫描基础包：@ComponentScan(basePackages="spring02.assemble.automatic")
* 3. 设置组件扫描基础包：@ComponentScan(basePackages={"com.01", "com.02"})
* 4. 设置组件类：@ComponentScan(basePackageClasses={SgtPeppers.class, CDPlayer.class})
*/
@ComponentScan("spring02.assemble.automatic") 
public class CDPlayerConfig {
}
~~~

@Autowired注解实现自动装配：让Spring自动满足bean依赖

~~~java
@Component
public class CDPlayer implements MediaPlayer{
	private CompactDisc cd;
	/*
	* @Autowired
	* 1. @Autowired注解可以用在任意方法上
	* 2. 如果找不到bean匹配则会抛出异常，可以将@Autowired的required属性设成false避免抛出异常
	* 2. @Autowired(required=false)，匹配不上会让bean处于未装配状态
	* 3. @Autowired注解的替代注解@Inject,后者同时定义了@Named注解
	* 4. 多个bean匹配会产生自动装配的歧义性
	*/
	@Autowired
	public CDPlayer(CompactDisc cd){
		this.cd = cd;
	}
	
	@Override
	public void play() {
		cd.play();
	}
}
~~~

验证自动装配

~~~java
/*
* @RunWith
* @RunWith使用了Spring的SpringJUnit4ClassRunner，以便在测试开始的时候自动创建Spring的应用上下文
*/
@RunWith(SpringJUnit4ClassRunner.class)
/*
* @ContextConfiguration
* @ContextConfiguration会告诉测试类要在CDPlayerConfig中加载配置，因为CDPlayerConfig包含了@ComponentScan
*/
@ContextConfiguration(classes=CDPlayerConfig.class)
public class CDPlayerTest {

	@Rule
	public final StandardOutputStreamLog log = new StandardOutputStreamLog();
	
	@Autowired /*自动注解标识*/
	private CompactDisc cd;
	
	@Autowired
	private MediaPlayer player;
	
	/*
	 * 测试组件扫描能够发现CompactDisc
	 */
	@Test
	public void cdShouldNotBeNull(){
		assertNotNull(cd);
	}
	
	@Test
    public void play() {
      player.play();
      assertEquals("Playing Lonely Heartsby The Beatles\n", log.getLog());
    }
}
~~~

## 通过java代码装配bean

创建配置类：

~~~java
/*
* @Configuration
* @Configuration表明这个类是配置类，该类应该包含在Spring应用上下文中如何创建bean的细节
*/
@Configuration
public class CDPlayerConfig {
	/*
	* @Bean
	* 1. @Bean注解会返回一个Spring应用上下文中的bean
	* 2. @Bean的ID默认与方法名同名，例：compactDisc
	* 2. 重新设置@Bean的名字：@Bean(name="rename")
	*/
  	@Bean
	public CompactDisc compactDisc(){
		return new SgtPeppers();
	}
	
	@Bean
	public CDPlayer cdPlayer(CompactDisc compactDisc) {
	    return new CDPlayer(compactDisc);
	}
}
~~~

实现类<SgtPeppers>和<CDPlayer>以及测试类<CDPlayerTest>同上。

## 通过XML装配bean

XML配置规范：

~~~java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans                                http://www.springframework.org/schema/beans/spring-beans.xsd">

  <!-- configuration details go here-->

</beans>
~~~

通用实现类和测试类：

~~~java
/*
* 非字面量SgtPeppers实现类
*/
public class SgtPeppers implements CompactDisc {
  private String title = "Sgt. Pepper's Lonely Hearts Club Band";  
  private String artist = "The Beatles";
  
  public void play() {
    System.out.println("Playing " + title + " by " + artist);
  }
}

/**
 *  测试字面量实现类
 */
public class BlankDisc implements CompactDisc {
  private String title;
  private String artist;

  public BlankDisc(String title, String artist) {
    this.title = title;
    this.artist = artist;
  }

  public void play() {
    System.out.println("Playing " + title + " by " + artist);
  }
}

/**
 *  测试集合实现类
 *  collections包下
 */
public class BlankDisc implements CompactDisc {
  private String title;
  private String artist;
  private List<String> tracks;

  public BlankDisc(String title, String artist, List<String> tracks) {
    this.title = title;
    this.artist = artist;
    this.tracks = tracks;
  }

  public void play() {
    System.out.println("Playing " + title + " by " + artist);
    for (String track : tracks) {
      System.out.println("-Track: " + track);
    }
  }

/**
 *  测试属性实现类
 *  properties包下
 */
public class BlankDisc implements CompactDisc {
  private String title;
  private String artist;
  private List<String> tracks;

  public void setTitle(String title) {
    this.title = title;
  }

  public void setArtist(String artist) {
    this.artist = artist;
  }

  public void setTracks(List<String> tracks) {
    this.tracks = tracks;
  }

  public void play() {
    System.out.println("Playing " + title + " by " + artist);
    for (String track : tracks) {
      System.out.println("-Track: " + track);
    }
  }
}  

public class CDPlayer implements MediaPlayer{
	private CompactDisc cd;
	@Autowired
	public CDPlayer(CompactDisc cd){//cd构造器参数名
		this.cd = cd;
	}
	@Override
	public void play() {
		cd.play();
	}
}

/**
 *  测试属性实现类
 *  properties包下
 */
public class CDPlayer implements MediaPlayer {
  private CompactDisc compactDisc;

  @Autowired
  public void setCompactDisc(CompactDisc compactDisc) {//compactDisc属性名
    this.compactDisc = compactDisc;
  }

  public void play() {
    compactDisc.play();
  }
}
  
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="classpath:springXML02/ConstructorArgReferenceTest-context.xml")
public class ConstructorArgReferenceTest {

  @Rule
  public final StandardOutputStreamLog log = new StandardOutputStreamLog();

  @Autowired
  private MediaPlayer player;

  @Test
  public void play() {
    player.play();
    assertEquals("Playing Sgt. Pepper's Lonely Hearts Club Band by The Beatles\n", log.getLog());
  }
}
~~~

下面进行说明XML配置的几种情况：

1. 构造器注入bean引用

   ~~~java
   <bean id="compactDisc" class="spring02.assemble.xml.SgtPeppers" />
           
   <bean id="cdPlayer" class="spring02.assemble.xml.CDPlayer">
       <constructor-arg name = "compactDisc" ref="compactDisc" />
   </bean>
   ~~~

2. 将字面量注入到构造器中

   ~~~java
   <bean id="compactDisc" class="spring02.assemble.xml.BlankDisc">
       <constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
       <constructor-arg value="The Beatles" />
   </bean>
           
   <bean id="cdPlayer" class="spring02.assemble.xml.CDPlayer">
       <constructor-arg ref="compactDisc" />
   </bean>
   ~~~

3. 通过构造器装配集合

   ~~~java
   <bean id="compactDisc" class="spring02.assemble.xml.collections.BlankDisc">
       <constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
       <constructor-arg value="The Beatles" />
       <constructor-arg>
         <list>
           <value>Sgt. Pepper's Lonely Hearts Club Band</value>
           <value>With a Little Help from My Friends</value>
           <value>Lucy in the Sky with Diamonds</value>
           <value>Getting Better</value>
           <value>Fixing a Hole</value>
           <value>She's Leaving Home</value>
           <value>Being for the Benefit of Mr. Kite!</value>
           <value>Within You Without You</value>
           <value>When I'm Sixty-Four</value>
           <value>Lovely Rita</value>
           <value>Good Morning Good Morning</value>
           <value>Sgt. Pepper's Lonely Hearts Club Band (Reprise)</value>
           <value>A Day in the Life</value>
         </list>
       </constructor-arg>
   </bean>
           
   <bean id="cdPlayer" class="spring02.assemble.xml.CDPlayer">
       <constructor-arg ref="compactDisc" />
   </bean>
   ~~~

4. 构造器注入bean引用 --- c-命名模式

   ~~~java
   <!-- c-命名模式：需要在beans中引入c --> 
   xmlns:c="http://www.springframework.org/schema/c"

   <bean id="compactDisc" class="spring02.assemble.xml.SgtPeppers" />
   <!-- c-命名模式：c指前缀,cd指构造器参数名,-ref指注入bean引用,value指要注入的ID -->       
   <bean id="cdPlayer" class="spring02.assemble.xml.CDPlayer" c:cd-ref="compactDisc" />
   ~~~

5. 将字面量注入到构造器中 --- c-命名模式

   ~~~java
   <!-- c-命名模式：可以用占位符表示参数的位置，xml中不允许数字作为属性的第一个字符，所以加_前缀 --> 
   <bean id="compactDisc" class="spring02.assemble.xml.BlankDisc"
           c:_0="Sgt. Pepper's Lonely Hearts Club Band" 
           c:_1="The Beatles">
   </bean>  
   <!-- c-命名模式：当只有一个时可以只写_ -->          
   <bean id="cdPlayer" class="spring02.assemble.xml.CDPlayer" c:_-ref="compactDisc" />
   ~~~

6. 属性注入bean引用

   ~~~java
   <bean id="compactDisc" class="spring02.assemble.xml.BlankDisc">
       <constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
       <constructor-arg value="The Beatles" />
   </bean>
           
   <bean id="cdPlayer" class="spring02.assemble.xml.properties.CDPlayer">
       <property name="compactDisc" ref="compactDisc" />
   </bean>
   ~~~

7. 将字面量注入到属性中

   ~~~java
   <bean id="compactDisc"
           class="spring02.assemble.xml.properties.BlankDisc">
       <property name="title" value="Sgt. Pepper's Lonely Hearts Club Band" />
       <property name="artist" value="The Beatles" />
       <property name="tracks">
         <list>
           <value>Sgt. Pepper's Lonely Hearts Club Band</value>
           <value>With a Little Help from My Friends</value>
           <value>Lucy in the Sky with Diamonds</value>
           <value>Getting Better</value>
           <value>Fixing a Hole</value>
           <value>She's Leaving Home</value>
           <value>Being for the Benefit of Mr. Kite!</value>
           <value>Within You Without You</value>
           <value>When I'm Sixty-Four</value>
           <value>Lovely Rita</value>
           <value>Good Morning Good Morning</value>
           <value>Sgt. Pepper's Lonely Hearts Club Band (Reprise)</value>
           <value>A Day in the Life</value>
         </list>
       </property>
   </bean>
           
   <bean id="cdPlayer" class="spring02.assemble.xml.properties.CDPlayer"
           <property name="compactDisc" ref="compactDisc" />
   </bean>
   ~~~

8. 属性注入bean引用 --- p-命名模式（规则同c-命名模式）

   ~~~java
   <!-- p-命名模式：需要在beans中引入p --> 
   xmlns:p="http://www.springframework.org/schema/p"
     
   <bean id="compactDisc" class="spring02.assemble.xml.BlankDisc">
       <constructor-arg value="Sgt. Pepper's Lonely Hearts Club Band" />
       <constructor-arg value="The Beatles" />
   </bean>
    
   <!-- p-命名模式：p指前缀,compactDisc指属性名,-ref指注入bean引用,value指要注入的ID -->     
   <bean id="cdPlayer" class="spring02.assemble.xml.properties.CDPlayer"
           p:compactDisc-ref="compactDisc">
   </bean>
   ~~~

9. 将字面量注入到属性中 --- p-命名模式

   ~~~java
   <bean id="compactDisc" class="spring02.assemble.xml.properties.BlankDisc"
           p:title="Sgt. Pepper's Lonely Hearts Club Band"
           p:artist="The Beatles">
       <property name="tracks">
         <list>
           <value>Sgt. Pepper's Lonely Hearts Club Band</value>
           <value>With a Little Help from My Friends</value>
           <value>Lucy in the Sky with Diamonds</value>
           <value>Getting Better</value>
           <value>Fixing a Hole</value>
           <value>She's Leaving Home</value>
           <value>Being for the Benefit of Mr. Kite!</value>
           <value>Within You Without You</value>
           <value>When I'm Sixty-Four</value>
           <value>Lovely Rita</value>
           <value>Good Morning Good Morning</value>
           <value>Sgt. Pepper's Lonely Hearts Club Band (Reprise)</value>
           <value>A Day in the Life</value>
         </list>
       </property>
   </bean>
           
   <bean id="cdPlayer" class="spring02.assemble.xml.properties.CDPlayer"
           p:compactDisc-ref="compactDisc">
   </bean> 
   ~~~

10. 将字面量注入到属性中 --- util-命名模式

    ~~~java
    <bean id="compactDisc" class="spring02.assemble.xml.properties.BlankDisc"
            p:title="Sgt. Pepper's Lonely Hearts Club Band"
            p:artist="The Beatles"
            p:tracks-ref="trackList">
    </bean>
    <util:list id="trackList">  
        <value>Sgt. Pepper's Lonely Hearts Club Band</value>
        <value>With a Little Help from My Friends</value>
        <value>Lucy in the Sky with Diamonds</value>
        <value>Getting Better</value>
        <value>Fixing a Hole</value>
        <value>She's Leaving Home</value>
        <value>Being for the Benefit of Mr. Kite!</value>
        <value>Within You Without You</value>
        <value>When I'm Sixty-Four</value>
        <value>Lovely Rita</value>
        <value>Good Morning Good Morning</value>
        <value>Sgt. Pepper's Lonely Hearts Club Band (Reprise)</value>
        <value>A Day in the Life</value>
    </util:list>

    <bean id="cdPlayer" class="spring02.assemble.xml.properties.CDPlayer"
            p:compactDisc-ref="compactDisc" />
    </bean>
    ~~~

    Spring util-命名空间中的元素

    |           元素           |                 描述                 |
    | :--------------------: | :--------------------------------: |
    |    <util:constant>     |  引用某个类型的public static域，并将其暴露为bean  |
    |      <util:list>       | 创建一个java.util.List类型的bean，其中包含值或引用 |
    |       <util:map>       | 创建一个java.util.Map类型的bean，其中包含值或引用  |
    |   <util:properties>    |  创建一个java.util.Properties类型的bean   |
    | <util:properties-path> |   引用一个bean的属性（或内嵌属性），并将其暴露为bean    |
    |       <util:set>       | 创建一个java.util.Set类型的bean，其中包含值或引用  |

## 导入和混合配置

通用实现类：

~~~java
public class BlankDisc implements CompactDisc {
  private String title;
  private String artist;
  private List<String> tracks;

  public BlankDisc(String title, String artist, List<String> tracks) {
    this.title = title;
    this.artist = artist;
    this.tracks = tracks;
  }

  public void play() {
    System.out.println("Playing " + title + " by " + artist);
    for (String track : tracks) {
      System.out.println("-Track: " + track);
    }
  }
}

public class CDPlayer implements MediaPlayer {
  private CompactDisc cd;
  @Autowired
  public CDPlayer(CompactDisc cd) {
    this.cd = cd;
  }

  public void play() {
    cd.play();
  }
}
~~~

1. javaConfig中引入XML配置

   配置类：

   ~~~java
   /*
   * CDPlayer采用javaConfig方式
   * xml需要将cd引进来
   */
   @Configuration
   public class CDPlayerConfig { 
     @Bean
     public CDPlayer cdPlayer(CompactDisc compactDisc) {
       return new CDPlayer(compactDisc);
     }
   }

   @Configuration
   @Import(CDPlayerConfig.class)
   @ImportResource("classpath:springXML02/ImportXmlConfigTest-config.xml")
   public class SoundSystemConfig {

   }
   ~~~

   测试类：

   ~~~java
   @RunWith(SpringJUnit4ClassRunner.class)
   @ContextConfiguration(classes=SoundSystemConfig.class)
   public class ImportXmlConfigTest {
     @Autowired
     private MediaPlayer player;

     @Test
     public void play() {
       player.play();
     }
   }
   ~~~

   xml文件：

   ~~~java
   <!--  xm需要将cd引进来 -->
   <bean id="compactDisc" class="spring02.assemble.mixdconfig.BlankDisc"
           c:_0="Sgt. Pepper's Lonely Hearts Club Band"
           c:_1="The Beatles">
       <constructor-arg>
         <list>
           <value>Sgt. Pepper's Lonely Hearts Club Band</value>
           <value>With a Little Help from My Friends</value>
           <value>Lucy in the Sky with Diamonds</value>
           <value>Getting Better</value>
           <value>Fixing a Hole</value>
           <!-- ...other tracks omitted for brevity... -->
         </list>
       </constructor-arg>
   </bean>
   ~~~

2. XML配置中引入javaConfig

   配置类：

   ~~~java
   /*
   * CDConfig在XML文件中被引入进来
   */
   @Configuration
   public class CDConfig {
     @Bean
     public CompactDisc compactDisc() {
       return new SgtPeppers();
     }
   }
   ~~~

   测试类：

   ~~~java
   @RunWith(SpringJUnit4ClassRunner.class)
   @ContextConfiguration("classpath:springXML02/ImportJavaConfigSystemTest-config.xml")
   public class ImportJavaConfigSystemTest {

     @Autowired
     private MediaPlayer player;

     @Test
     public void play() {
       player.play();
     }
   }
   ~~~

   xml文件：

   ~~~java
   <!-- 使用第三方配置文件将两个组合起来，在xml中引入javaConfig  -->
     
   <bean class="spring02.assemble.mixdconfig.CDConfig" />

   <import resource="ImportJavaConfigTest-config.xml" />
     
   <!-- 第三方配置文件需要引入的xml文件ImportJavaConfigTest-config.xml  -->
     
   <bean id="cdPlayer" class="spring02.assemble.mixdconfig.CDPlayer"
           c:cd-ref="compactDisc" />
   ~~~

   ​



