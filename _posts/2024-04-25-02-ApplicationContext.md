---
layout:     post
title:      02-ApplicationContext
subtitle:   认识ApplicationContext
date:       2024-04-25
author:     zhoujunlin
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Spring
    - 工作
---


<a name="SoiH9"></a>
# ApplicationContext具备的能力
![image.png](https://cdn.nlark.com/yuque/0/2024/png/22825798/1709637826335-ce9afb69-8e47-47d3-a0c7-11fea11342be.png#averageHue=%232e2e2e&clientId=u7ff37d2f-2568-4&from=paste&height=209&id=u47ec01d4&originHeight=261&originWidth=1487&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=29223&status=done&style=none&taskId=u3f21cb2f-5259-44ec-a9c6-a14137fea2c&title=&width=1189.6)
<a name="r7MtU"></a>
## 路径解析
```java
ConfigurableApplicationContext applicationContext = SpringApplication.run(ApplicationContextTest01.class, args);

/**
 * ResourcePatternResolver
 * classpath*:  可以加载jar包里的文件
 */
Resource[] resources = applicationContext.getResources("classpath:application.yaml");
//Resource[] resources = applicationContext.getResources("classpath*:META-INF/spring.factories");
for (Resource resource : resources) {
    System.err.println(resource);
}
```
<a name="YlW6d"></a>
## 读取环境配置
```java
String property = applicationContext.getEnvironment().getProperty("server.port");
System.err.println(property);
```
<a name="qAvCZ"></a>
## 国际化
![image.png](https://cdn.nlark.com/yuque/0/2024/png/22825798/1709637314665-b699264c-e8f7-4abe-b2f7-a58bd4a56141.png#averageHue=%234a554b&clientId=u7ff37d2f-2568-4&from=paste&height=93&id=u2809a491&originHeight=116&originWidth=359&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=6994&status=done&style=none&taskId=u17e3c7bb-a0e2-462f-b06d-dc5c7a0e7be&title=&width=287.2)<br />![image.png](https://cdn.nlark.com/yuque/0/2024/png/22825798/1709637355574-6ef61191-d179-42c1-8a6a-7c18929d5b11.png#averageHue=%2348543d&clientId=u7ff37d2f-2568-4&from=paste&height=93&id=u59384cc3&originHeight=116&originWidth=1318&originalType=binary&ratio=1.25&rotation=0&showTitle=false&size=18316&status=done&style=none&taskId=u0b09a4d4-ae80-46be-8a41-f717b8cf5c3&title=&width=1054.4)
```java
// String welcome = applicationContext.getMessage("welcome", new String[]{"zhoujunlin", DateUtil.now()}, Locale.CHINA);
String welcome = applicationContext.getMessage("welcome", new String[]{"zhoujunlin", DateUtil.now()}, Locale.US);
System.err.println(welcome);
```
<a name="nfk9g"></a>
## 事件发布
事件、事件发布者与事件监听者
```java
public class RegisterEvent extends ApplicationEvent {

    public RegisterEvent(Object source) {
        super(source);
    }
}

@Component
public class EventPublisher {

    @Resource
    private ApplicationEventPublisher applicationEventPublisher;

    public void publishEvent() {
        applicationEventPublisher.publishEvent(new RegisterEvent(this));
    }

}

@Component
public class EventConsumer {

    @EventListener
    public void handler(RegisterEvent registerEvent) {
        System.out.println("RegisterEvent by EventConsumer source:" + registerEvent.getSource());
    }

}
```
```java
/**
 * ApplicationEventPublisher
 */
//applicationContext.publishEvent(new RegisterEvent(applicationContext));
applicationContext.getBean(EventPublisher.class).publishEvent();
```
<a name="tkt7K"></a>
# BeanFactory与ApplicationContext
在没有ApplicationContext时，是怎么样将BeanDefinition加载到BeanFactory中的？ （ApplicationContext是对BeanFactory进行更进一步封装增强）
<a name="CNFQu"></a>
## 定义Bean
```java
static class Bean01 {
}

@Data
static class Bean02 {
    private Bean01 bean01;
}
```
<a name="AYYNa"></a>
## 定义bean.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="bean01" class="com.you.meet.nice.test.web.spring.applicationcontext.ApplicationContextTest02.Bean01"/>

    <bean id="bean02" class="com.you.meet.nice.test.web.spring.applicationcontext.ApplicationContextTest02.Bean02">
        <property name="bean01" ref="bean01"/>
    </bean>


    <!--
    相当于  AnnotationConfigUtils.registerAnnotationConfigProcessors(beanFactory);  此功能
    加载一些内置后处理器
    -->
    <!--    <context:annotation-config/> -->
</beans>
```
<a name="GqULF"></a>
## 加载xml中的BeanDefinition到BeanFactory
```java
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();
System.out.println("加载xml之前。。。");
for (String beanDefinitionName : beanFactory.getBeanDefinitionNames()) {
    System.out.println(beanDefinitionName);
}
XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
xmlBeanDefinitionReader.loadBeanDefinitions(new ClassPathResource("bean01.xml"));

System.out.println("加载xml之后。。。");
for (String beanDefinitionName : beanFactory.getBeanDefinitionNames()) {
    System.out.println(beanDefinitionName);
}
System.out.println(beanFactory.getBean(Bean02.class).getBean01());
```
<a name="sLPRN"></a>
# 常见的几种ApplicationContext
<a name="MSuk5"></a>
## ClassPathXmlApplicationContext
```java
// ClassPathXmlApplicationContext加载BeanDefinition到Bean容器
ClassPathXmlApplicationContext classPathXmlApplicationContext = new ClassPathXmlApplicationContext("bean01.xml");
for (String beanDefinitionName : classPathXmlApplicationContext.getBeanDefinitionNames()) {
    System.out.println(beanDefinitionName);
}
System.out.println(classPathXmlApplicationContext.getBean(Bean02.class).getBean01());
```
<a name="j1II5"></a>
## FileSystemXmlApplicationContext
```java
FileSystemXmlApplicationContext fileSystemXmlApplicationContext = new FileSystemXmlApplicationContext("/src/test/resources/bean01.xml");
for (String beanDefinitionName : fileSystemXmlApplicationContext.getBeanDefinitionNames()) {
    System.out.println(beanDefinitionName);
}
System.out.println(fileSystemXmlApplicationContext.getBean(Bean02.class).getBean01());
```
<a name="ofGMM"></a>
## AnnotationConfigApplicationContext
```java
@Configuration
static class Config {
    @Bean
    public Bean01 bean01() {
        return new Bean01();
    }

    @Bean
    public Bean02 bean02(Bean01 bean01) {
        Bean02 bean02 = new Bean02();
        bean02.setBean01(bean01);
        return bean02;
    }

}
```
```java
AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(Config.class);
/**
 * 多加载了内置处理器
 * internalConfigurationAnnotationProcessor、internalAutowiredAnnotationProcessor、internalCommonAnnotationProcessor
 * internalEventListenerProcessor、internalEventListenerFactory
 * 相当于多了此功能
 * AnnotationConfigUtils.registerAnnotationConfigProcessors(beanFactory);
 */
for (String beanDefinitionName : annotationConfigApplicationContext.getBeanDefinitionNames()) {
    System.out.println(beanDefinitionName);
}
System.out.println(annotationConfigApplicationContext.getBean(Bean02.class).getBean01());
```
<a name="meeeG"></a>
## AnnotationConfigServletWebServerApplicationContext
```java
@Configuration
static class WebConfig {

    @Bean
    public ServletWebServerFactory tomcatServletWebServerFactory() {
        return new TomcatServletWebServerFactory();
    }

    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    }

    @Bean("/hello")
    public Controller hello() {
        return (request, response) -> {
            response.getWriter().print("hello world");
            return null;
        };
    }

}
```
运行web服务
```java
AnnotationConfigServletWebServerApplicationContext webServerApplicationContext = 
    new AnnotationConfigServletWebServerApplicationContext(WebConfig.class);
```








