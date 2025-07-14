---
title: "small-spring 学习总结"
description: "第一次感受到编程的魅力"
publishDate: "2025-07-14"
draft: false
tags: ["脚手架学习"]
---

# 爱上编程的第一步

> 如果以后我真的可以全心全意地爱上编程的话，那这个项目就是丘比特的箭了！

## 写在开头

跟着教程一步一步地把这个简陋的 Spring 框架敲出来了，DI 容器、Bean 生命周期管理、Xml 配置、Aware、Event、AOP、注解与包扫描、多级缓存、类型转换，全都涉猎了一遍。

这是我的第一个正经学完的编程项目，回想本科期间，可能只有编译原理课上的简单编译器可以与之媲美，但是编译原理课的助教团体太过负责，制作了茫茫多的非公开测试用例，导致项目体感更类似 OJ 而非工程……往事不要再提。

还记得把多级缓存搞明白的那个晚上，我是 8 点 50 离开公司座位（那是周六，所以没在摸鱼啦，为了周末有饭吃就到公司去了），走到楼下，朗月清风。我骑着共享单车绕着小区转了好多好多圈之后才回家，唱的是伍佰的《亏欠》和《一生最爱的人》，也不知道唱这两首歌的时候心里想的是谁，希望那天早睡的人没有被我吵醒……

如果你也感兴趣，可以参考我的学习路径总结：[my-small-spring 项目地址](https://github.com/WWWeedss/my-small-spring)

下方的内容则是我自己对 Spring 学习的一个回顾，练一练总结和抓重点脉络的能力。

## 为什么要有 Spring

为了能这么写后端：

```java
@Autowired
UserService userService;
```

Spring 就是基础设施，除了帮我们管理好实例之间的依赖，还内置了 event、AOP 等等功能。我已经无法想象我无法想象没有 SpringBoot 或类似 DI 容器的后端开发流程，就像我无法想象没有 ChatAI 的工作生活。

## 一起搭一个积木：从使用者的角度看 Spring 功能的逐步丰满 

### 最简单的容器

BeanFactory 里存储了一个 HashMap，这就是全部的秘密。

```java
beanFactory.register("userService", new UserService());
UserService userService = beanFactory.get("userService");
```

### 用反射来替代 new

我们增加一个 BeanDefinition 的定义，在 BeanDefinition 内存储 BeanName 和 Class 信息。这就可以用反射来避免用户直接 new 对象了。

```java
BeanDefinition beanDefinition = new BeanDefinition(UserService.class);
beanFactory.registerBeanDefinition("userService", beanDefinition);
UserService userService = beanFactory.getBean("userService");
```

有参构造 Bean 与属性注入

在 getBean 的时候传入构造参数，使用反射匹配符合参数的构造函数，完成 bean 的有参构造。

```java
BeanDefinition beanDefinition = new BeanDefinition(UserService.class);
beanFactory.registerBeanDefinition("userService", beanDefinition);
// UserService 有一个 name 属性，接受一个 String 参数
UserService userService = beanFactory.getBean("userService", "WWWeeds");
```

用户不仅可以在 getBean 的时候传参数，还可以在 getBean 之前配置属性，这就用到了属性注入。

没有什么神秘的，只是 BeanDefinition 中存储了 PropertyValues，每一个 PropertyValue 都有字段名和对应的值，在 createBean 的时候使用反射进行 setField 而已。当然我们也需要一个额外的类型来表示引用，这就是 BeanReference，注入 BeanReference 时要使用 getBean。 这里我们忽略循环依赖和字段类型不匹配的情况。

```java
beanFactory.registerBeanDefinition("userDao", new BeanDefinition(UserDao.class));

PropertyValues propertyValues = new PropertyValues();
propertyValues.addPropertyValue(new PropertyValue("uId", "10001"));
propertyValues.addPropertyValue(new PropertyValue("userDao", new BeanReference("userDao")));

BeanDefinition beanDefinition = new BeanDefinition(UserService.class, propertyValues);
beanFactory.registerBeanDefinition("userService", beanDefinition);

// 框架会先 getBean("userDao")，然后将取出的 userDao 注入 userService 的字段中；对于 uId 这个字段直接注入就好。
UserService userService = (UserService) beanFactory.getBean("userService");
```

### 使用 XML 进行配置

Xml 就是一个树状元素，把我们在代码里写的东西写到 xml 文件中，仅此而已。为了兼容各种路径（classpath、普通路径、网络 url），Spring 框架使用不同的 Reader 实现了同一个接口方法：输入路径，输出字节流。这是经典的策略模式或者简单工厂的应用场景。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>

    <bean id="userDao" class="bean.UserDao"/>

    <bean id="userService" class="bean.UserService">
        <property name="uId" value="10001"/>
        <property name="userDao" ref="userDao"/>
    </bean>

</beans>
```

> 就是把我们的代码翻译了一遍，没有任何失真，除了 value 只能是字符串

已经有点像魔法了，它自动生成了 UserDefinition，自动注入了 userDao，我们只要拿出来用就可以。当然前提是……xml 文件不是你填的。

```java
UserService userService = beanFactory.getBean("userService");
```

### 增加 ApplicationContext 控制 Bean 的生命周期

**BeanPostprocessor**

我们希望在创建 Bean 的过程中对 BeanDefinition 或者 Bean 进行一些特殊处理，这就是 PostProcessor。

实现方式就是新建一个 ApplicationContext 类持有 BeanFactory，在 ApplicationContext 中定义 Bean 的全生命周期。在注册 BeanDefinition 之后执行 BeanFactoryPostProcessor 之类。

像 BeanFactoryPostProcessor  本身也是 Bean，我们实现之后用 Xml 注册进来，容器就能自动通过类型识别取出调用了。

```java
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        BeanDefinition beanDefinition = beanFactory.getBeanDefinition("userService");
        PropertyValues propertyValues = beanDefinition.getPropertyValues();

        propertyValues.addPropertyValue(new PropertyValue("company", "改为：字节跳动"));
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans>

    <bean id="userDao" class="bean.UserDao"/>

    <bean id="userService" class="bean.UserService1">
        <property name="uId" value="10001"/>
        <property name="company" value="腾讯"/>
        <property name="location" value="深圳"/>
        <property name="userDao" ref="userDao"/>
    </bean>

    <bean class="common.MyBeanPostProcessor"/>
    <bean class="common.MyBeanFactoryPostProcessor"/>

</beans>
```

```java
// 1.初始化 BeanFactory
ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:springPostProcessor.xml");

// 2. 获取Bean对象调用方法
UserService userService = applicationContext.getBean("userService", UserService.class);
String result = userService.queryUserInfo();
System.out.println("测试结果：" + result);
```

BeanPostProcessor 的具体执行对用户是无感知的，后面的 Aware、Event、AOP 都会用到它。

**初始化与销毁**

有两种注册初始化与销毁方法的方式，一种是类似 BeanPostProcessors，透出接口，在特定位点取出实现了接口的 Bean 执行，另一种就是 xml 配置，通过反射取出特定方法，在生命周期的特定位置执行。其中销毁方法是用虚拟机关闭时的钩子触发的。

initialzingBean.afterProperties() 会先于 init-method 执行，二者均配置时顺序执行。

disposableBean.destroy() 会先于 destroy.method 执行，二者均配置时只执行 destroy，避免二次销毁。

xml 配置如下：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans>

    <bean id="userDao" class="bean.UserDao" init-method="initDataMethod" destroy-method="destroyDataMethod"/>

    <bean id="userService" class="bean.UserService1">
        <property name="uId" value="10001"/>
        <property name="company" value="淘天"/>
        <property name="location" value="杭州"/>
        <property name="userDao" ref="userDao"/>
    </bean>

</beans>
```

```java
package bean;

import springframework.beans.factory.DisposableBean;
import springframework.beans.factory.InitializingBean;

public class UserService implements InitializingBean, DisposableBean {

    private String uId;
    private String company;
    private String location;
    private UserDao userDao;

    @Override
    public void destroy() throws Exception {
        System.out.println("执行：UserService.destroy()");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("执行：UserService.afterPropertiesSet()");
    }
}
```

### 感知 Aware

所谓感知，就是让 Bean 知道创建自己的 BeanName、ClassLoader、BeanFactory、ApplicationContext，来进行一些反向操作。框架的实现非常简单，通过注册一个 BeanPostProcessor，透出一些 Aware 接口，这个 BeanPostProcessor 会对实现了 Aware 接口的 Bean 进行注入（调用 set 方法）。

其中一个 Aware 接口：

```java
public interface ApplicationContextAware extends Aware {
    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

用户可以这么去定义 Bean：

```java
public class UserService implements BeanNameAware, ApplicationContextAware, BeanFactoryAware {

    private ApplicationContext applicationContext;
    private BeanFactory beanFactory;

    private String uId;
    private String company;
    private String location;
    private UserDao userDao;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void setBeanName(String name) {
        System.out.println("Bean Name is：" + name);
    }

}
```

### Spring Event

发送消息是一个经典的解耦需求，在这里我们暂且不考虑消息队列。Spring 框架内部注册一个 ApplicationEventMulticaster Bean，从 ApplicationContext 向外暴露一个 PublishEvent 方法。

当用户使用 ApplicationContext.publishEvent() 时，ApplicationEventMulticaster 就会通知所有订阅该事件的 Listener。

Event 和自定义 Listener 由用户实现接口，通过泛型指定订阅的消息类型，框架会抛出几个特定事件，比如容器刷新完成和关闭。

```java
public class CustomEvent extends ApplicationContextEvent {

    private Long id;
    private String message;

    public CustomEvent(Object source, Long id, String message) {
        super(source);
        this.id = id;
        this.message = message;
    }

    public Long getId() {
        return id;
    }

    public String getMessage() {
        return message;
    }
}

```

```java
public class CustomEventListener implements ApplicationListener<CustomEvent> {
    @Override
    public void onApplicationEvent(CustomEvent event) {
        System.out.println("CustomEventListener received event: " + event.getMessage() + " with ID: " + event.getId());
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
    <beans>

        <bean class="event.ContextRefreshedEventListener"/>

        <bean class="event.CustomEventListener"/>

        <bean class="event.ContextClosedEventListener"/>

    </beans>
```

```java
public class ApiTest {
    @Test
    public void test_event(){
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext(("classpath:spring.xml"));
        applicationContext.publishEvent(new CustomEvent(applicationContext, 1001L, "Hello, this is a custom event!"));
    }
}
```

### AOP

首先你需要了解动态代理：

![image-20250703073925782](https://typora-images-wwweeds.oss-cn-hangzhou.aliyuncs.com/image-20250703073925782.png)

动态代理持有着 targetObject 的引用，通过反射的方式调用其中的方法，并在调用前后增加额外操作。那么如何对用户屏蔽代理的存在呢？JDK 的动态代理是让代理和 targetObject 实现同一个接口，用户只能调用接口中的方法，Cglib 的动态代理是让代理类去继承 targetObject，形成增强子类。

AOP 也很好理解，在 Bean 初始化完成之后，通过 AspectJ 切点表达式（切点表达式标注了）判断这个 Bean 是否需要代理，如果需要就增强一下。后续 getBean 出来都是这个代理类，用户的调用都会被代理转发，代理的 preprocessor 和 postprocessor 都暴露出来可以给用户自定义，去打日志之类的。

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans>

    <bean id="userService" class="bean.UserService1"/>

    <bean class="springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>

    <bean id="beforeAdvice" class="bean.UserServiceBeforeAdvice"/>

    <bean id="methodInterceptor" class="springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor">
        <property name="advice" ref="beforeAdvice"/>
    </bean>

    <bean id="pointcutAdvisor" class="springframework.aop.aspectj.AspectJExpressionPointcutAdvisor">
        <property name="expression" value="execution(* bean.IUserService.*(..))"/>
        <property name="advice" ref="methodInterceptor"/>
    </bean>

</beans>
```

beforeAdvice 定义了方法的前置操作，在 methodInterceptor 和原方法执行一起被封装到 Invoke 中。pointcutAdvisor 定义了切点表达式，告知了 proxy 要匹配哪些方法。

Proxy 接到调用之后，就检查切点表达式是否能匹配到，能就调用 methodInterceptor 的 invoke，否则直接执行。

```java
public class UserServiceBeforeAdvice implements MethodBeforeAdvice {
    @Override
    // 在目标方法之前执行
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("拦截方法：" + method.getName());
    }
}
```

```java
public class ApiTest {
    @Test
    public void test_aop() throws NoSuchMethodException {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:spring.xml");

        IUserService userService = applicationContext.getBean("userService", IUserService.class);
        System.out.println(userService.queryUserInfo());
    }
}
```

### 注解与包扫描

这是一个非常经典的用户体验优化。经过这一步，我们就能写出这样的类了：
```java
@Component("userService")
public class UserService {
    @Autowired
    UserDao userDao;
    
    @Value("helloWorld");
    String token;
}
```

保留到运行时的注解会和修饰类绑定，我们可以通过反射拿到指定包内被修饰的类，这就可以对 @Component 修饰的类进行 Bean 注册，再拿到被注解修饰的字段，进行属性注入即可。

## Bean 的生命周期

![image-20250714190004691](https://typora-images-wwweeds.oss-cn-hangzhou.aliyuncs.com/image-20250714190004691.png)

> 对于 Prototype 类型的 Bean 来说，它根本不进入缓存，创建完直接返回，不归 Spring 框架管，没有什么讨论的价值。

每两个步骤之间，都可以添加 PostProcessors 去修改 BeanDefinition 或者 Bean，Spring 框架提供了切口给我们，我们只需要继承某一个特定类或者实现某一个特定接口就可以了。

![image-20250714230832198](https://typora-images-wwweeds.oss-cn-hangzhou.aliyuncs.com/image-20250714230832198.png)



## 其他细节

### 关于 FactoryBean 

FactoryBean 是 Bean 的一种。它存储在 Spring 框架内，但是无法被 getBean 获取实例，getBean 会调用 FactoryBean.getObject() 方法，这个方法的返回值才是 getBean 的结果。

FactoryBean 实际是把 Bean 的实例化抛回给用户，如果你的对象需要复杂的配置，无法用 get、set、依赖注入解决，或者说每次 getBean 都需要根据 xx 因素改变一下，或者 FactoryBean 本身有一些额外作用，那就用 FactoryBean 吧。

### 循环依赖的解决

基本方案是缓存。

```java
A a = new A();
B b = new B();
a.setB(b);
b.setA(a);
```

Spring 就是把上述代码中的 a 刚 new 出来就放入了缓存以供 b 引用，通过维持引用一致，在 a 初始化之后，b 持有的 a 也初始化完成。

至于为什么要有三级缓存，缺乏了 ApplicationContext 以及 Spring AOP 如何实现的前置知识，是无法理解的。可以顺着项目走，走到那里就懂了。

### 类型转换的解决

@Value 只能写字符串值，那么 Spring 如何将字符串值转换为其他类型并注入字段呢。这就用到了 Convertor。你可以简单地理解为 Spring 的容器中存储了一组 Convertors，注入字段时 Spring 会尝试去匹配，如果有类型符合的 Convertor 就使用。

Spring 内置了字符串到数字值的转换器，用户也可以自己实现并注册成为 Bean。

下方的配置文件中就使用了 FactoryBean，当我们 getBean(conversionService) 时，它就会返回 converters，而 getBean(converters)，返回的就是我们定义的各种转换器了。

```xml
<beans>
    <bean id="conversionService" class="springframework.context.support.ConversionServiceFactoryBean">
        <property name="converters" ref="converters"/>
    </bean>

    <bean id="converters" class="converter.ConvertersFactoryBean"/>
    
    <component-scan base-package="bean"/>
</beans>
```



## 写在最后

将日常使用的东西逐渐解构，这种对世界理解加深的感觉真是太棒了；不考虑面试、不考虑就业、不考虑任何东西，只凭借单纯的好奇与 Spring 代码本身的优雅结构（也许只是比我写的代码优雅？），就可以收获很多很多乐趣。在以后的日子里，我想要更多地学习日常接触的东西的原理，并且开发我自己日常可以使用的软件。什么都离不开生活，软件也是。

**用软件来改善生活，是二十一世纪魔法师的最终使命。**

