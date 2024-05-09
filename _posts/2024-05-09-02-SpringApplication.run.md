---
layout:     post
title:      02-SpringApplication.run
subtitle:   SpringApplication.run重要流程
date:       2024-05-09
author:     zhoujunlin
header-img: img/post-bg-mma-5.jpg
catalog: true
tags:
    - SpringBoot
---

# 事件发布与监听

```java

SpringApplication springApplication = new SpringApplication();
// 添加事件监听器  用于监听到事件回调处理逻辑
springApplication.addListeners(event -> {
    System.out.println("事件来源: " + event.getClass());
});

// 加载配置文件中(spring.factories)的SpringApplicationRunListener
List<String> appRunListenerNames = SpringFactoriesLoader.loadFactoryNames(SpringApplicationRunListener.class, Step01.class.getClassLoader());
for (String appRunListenerName : appRunListenerNames) {
    // 目前只有一个EventPublishingRunListener
    System.out.println("publisherClass: =>" + appRunListenerName);
    // 生成对象
    Class<?> appRunListenerClazz = Class.forName(appRunListenerName);
    Constructor<?> constructor = appRunListenerClazz.getConstructor(SpringApplication.class, String[].class);
    SpringApplicationRunListener publisher = (SpringApplicationRunListener) constructor.newInstance(springApplication, args);

    // 开始发布事件
    DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
    publisher.starting(bootstrapContext); // springboot开始启动事件  ApplicationStartingEvent
    publisher.environmentPrepared(bootstrapContext, new StandardEnvironment());  // 环境信息准备完毕  ApplicationEnvironmentPreparedEvent
    GenericApplicationContext genericApplicationContext = new GenericApplicationContext();
    publisher.contextPrepared(genericApplicationContext);  // 在spring容器创建并调用初始化器之后  发送此事件  ApplicationContextInitializedEvent
    publisher.contextLoaded(genericApplicationContext);    //  所有BeanDefinition加载完成  ApplicationPreparedEvent
    genericApplicationContext.refresh();
    publisher.started(genericApplicationContext, Duration.ofSeconds(10));   // 容器初始化完成后  refresh调用完成  ApplicationStartedEvent
    publisher.ready(genericApplicationContext, Duration.ofSeconds(11));  // springboot启动完毕    ApplicationReadyEvent

    publisher.failed(genericApplicationContext, new Exception("故意抛出的异常"));  // springboot启动出错
}

```

# 准备Bean

```java

static class Bean4 {
}

static class Bean5 {
}

static class Bean6 {
}

@Configuration
static class Config {

    @Bean
    public Bean5 bean5() {
        return new Bean5();
    }

    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        return new TomcatServletWebServerFactory();
    }

    @Bean
    public CommandLineRunner commandLineRunner() {
        return new CommandLineRunner() {
            @Override
            public void run(String... args) throws Exception {
                System.out.println("\tCommandLineRunner..." + Arrays.toString(args));
            }
        };
    }

    @Bean
    public ApplicationRunner applicationRunner() {
        return new ApplicationRunner() {
            @Override
            public void run(ApplicationArguments args) throws Exception {
                System.out.println("\tApplicationRunner..." + Arrays.toString(args.getSourceArgs()));
                System.out.println("\tNonOptionArgs:" + args.getNonOptionArgs());
                System.out.println("\tOptionNames:" + args.getOptionNames());
                System.out.println("\tOptionArg:server.port: " + args.getOptionValues("server.port"));
            }
        };
    }

}

@Component
public class Bean10 {

}

```

```xml

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">


  <bean id="bean4" class="com.you.meet.nice.test.web.springboot.Step02.Bean4"/>

</beans>
```
# new SpringApplication
```java
SpringApplication springApplication = new SpringApplication();
springApplication.addInitializers((applicationContext) -> {
    System.out.println("\t执行初始化器回调增强容器");
});

```

# 创建容器

```java

private static GenericApplicationContext createApplicationContext(WebApplicationType webApplicationType) {
    GenericApplicationContext applicationContext = null;
    switch (webApplicationType) {
        case SERVLET:
            applicationContext = new AnnotationConfigServletWebServerApplicationContext();
            break;
        case REACTIVE:
            applicationContext = new AnnotationConfigReactiveWebServerApplicationContext();
            break;
        case NONE:
            applicationContext = new AnnotationConfigApplicationContext();
            break;
        default:
            break;
    }
    return applicationContext;
}


```

# 准备容器

```java

System.out.println("8. 创建容器");
GenericApplicationContext applicationContext = createApplicationContext(WebApplicationType.SERVLET);

System.out.println("9. 准备容器");
for (ApplicationContextInitializer initializer : springApplication.getInitializers()) {
    initializer.initialize(applicationContext);
}

```

# 加载BeanDefinition

```java

System.out.println("10. 加载BeanDefinition");
// 注解配置类中读取Bean
DefaultListableBeanFactory beanFactory = applicationContext.getDefaultListableBeanFactory();
AnnotatedBeanDefinitionReader annotatedBeanDefinitionReader = new AnnotatedBeanDefinitionReader(beanFactory);
annotatedBeanDefinitionReader.registerBean(Config.class);
// 从xml中读取
XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
xmlBeanDefinitionReader.loadBeanDefinitions(new ClassPathResource("b02.xml"));
// 扫包获取
ClassPathBeanDefinitionScanner classPathBeanDefinitionScanner = new ClassPathBeanDefinitionScanner(beanFactory);
classPathBeanDefinitionScanner.scan("com.you.meet.nice.test.web.springboot.sub");

```

# refresh

```java

System.out.println("11. refresh容器");
applicationContext.refresh();

```

# 执行runner

```java

System.out.println("12. 执行runner");
for (CommandLineRunner commandLineRunner : applicationContext.getBeansOfType(CommandLineRunner.class).values()) {
    commandLineRunner.run(args);
}
DefaultApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
for (ApplicationRunner applicationRunner : applicationContext.getBeansOfType(ApplicationRunner.class).values()) {
    applicationRunner.run(applicationArguments);
}

for (String beanDefinitionName : applicationContext.getBeanDefinitionNames()) {
    System.out.println("beanName:" + beanDefinitionName + " 来源=>" + applicationContext.getBeanFactory().getBeanDefinition(beanDefinitionName).getResourceDescription());
}
applicationContext.close();

```

# 环境

1. StandardEnvironment是spring中标准的环境类
2. ApplicationEnvironment是springboot中的,但是它是default修饰的,所以必须在同一个包下测试,这里建了个org.springframework.boot包
   1. systemProperties:  VM OPTIONS   -DJAVA_HOME=abc
   2. systemEnvironment 系统环境配置

## 准备step03.properties

```properties

user.first-name=zhou
user.middleName=jun
user.lase_name=lin

```
## 自定义添加配置来源

```java

// 初始构造时加载了   系统属性(systemProperties  jvm属性配置)系统环境变量(systemEnvironment)
ApplicationEnvironment applicationEnvironment = new ApplicationEnvironment();
// 放到最后  优先级最低
applicationEnvironment.getPropertySources().addLast(new ResourcePropertySource(new ClassPathResource("application.properties")));
applicationEnvironment.getPropertySources().addLast(new ResourcePropertySource(new ClassPathResource("step03.properties")));

// 解决属性匹配  _ , - , 驼峰 均能匹配到数据
ConfigurationPropertySources.attach(applicationEnvironment);

for (PropertySource<?> propertySource : applicationEnvironment.getPropertySources()) {
    // 配置获取优先级
    System.out.println(propertySource);
}

System.out.println(applicationEnvironment.getProperty("JAVA_HOME"));
System.out.println(applicationEnvironment.getProperty("server.port"));
// 默认属性名完全一样才能获取到配置
System.out.println(applicationEnvironment.getProperty("user.first-name"));
System.out.println(applicationEnvironment.getProperty("user.middle-name"));
System.out.println(applicationEnvironment.getProperty("user.lase-name"));

```

## 配置文件与随机数加载到环境后置处理器

```java

SpringApplication springApplication = new SpringApplication();
ApplicationEnvironment applicationEnvironment = new ApplicationEnvironment();
System.out.println("=======>增强前");
for (PropertySource<?> propertySource : applicationEnvironment.getPropertySources()) {
    System.out.println(propertySource);
}

// 添加application.properties  application.yaml配置文件
ConfigDataEnvironmentPostProcessor configDataEnvironmentPostProcessor = new ConfigDataEnvironmentPostProcessor(new DeferredLogs(), new DefaultBootstrapContext(), null);
configDataEnvironmentPostProcessor.postProcessEnvironment(applicationEnvironment, springApplication);

// 添加随机数 random.int  random.uuid   RandomValuePropertySource
RandomValuePropertySourceEnvironmentPostProcessor randomValuePropertySourceEnvironmentPostProcessor = new RandomValuePropertySourceEnvironmentPostProcessor(new DeferredLog());
randomValuePropertySourceEnvironmentPostProcessor.postProcessEnvironment(applicationEnvironment, springApplication);

System.out.println("=======>增强后");
for (PropertySource<?> propertySource : applicationEnvironment.getPropertySources()) {
    System.out.println(propertySource);
}

System.out.println(applicationEnvironment.getProperty("JAVA_HOME"));
System.out.println(applicationEnvironment.getProperty("server.port"));

```
## 监听器的方式增强环境

```java

SpringApplication springApplication = new SpringApplication();
springApplication.addListeners(new EnvironmentPostProcessorApplicationListener());

// 可以对环境进行增强的EnvironmentPostProcessor 执行时机是:EnvironmentPostProcessorApplicationListener事件监听
/*List<String> environmentPostProcessorNames = SpringFactoriesLoader.loadFactoryNames(EnvironmentPostProcessor.class, Step04.class.getClassLoader());
for (String environmentPostProcessorName : environmentPostProcessorNames) {
    System.out.println(environmentPostProcessorName);
}*/

ApplicationEnvironment applicationEnvironment = new ApplicationEnvironment();
System.out.println("=======>增强前");
for (PropertySource<?> propertySource : applicationEnvironment.getPropertySources()) {
    System.out.println(propertySource);
}
// 触发事件 -> EnvironmentPostProcessorApplicationListener监听回调 -> 执行所有EnvironmentPostProcessor
EventPublishingRunListener publisher = new EventPublishingRunListener(springApplication, args);
publisher.environmentPrepared(new DefaultBootstrapContext(), applicationEnvironment);

System.out.println("=======>增强后");
for (PropertySource<?> propertySource : applicationEnvironment.getPropertySources()) {
    System.out.println(propertySource);
}

```
## 配置绑定
### 准备对象

```java

@Data
static class User {
    private String firstName;
    private String middleName;
    private String lastName;
}

```

### 准备application.properties

```java

spring.main.lazy-initialization=true
spring.main.banner-mode=off

```

### 绑定新对象

```java

ApplicationEnvironment applicationEnvironment = new ApplicationEnvironment();
applicationEnvironment.getPropertySources().addLast(new ResourcePropertySource(new ClassPathResource("application.properties")));
applicationEnvironment.getPropertySources().addLast(new ResourcePropertySource(new ClassPathResource("step03.properties")));

// 绑定新对象  user是前缀
User user = Binder.get(applicationEnvironment).bind("user", User.class).get();
System.out.println(user);

```

### 绑定已存在的对象

```java

User user = new User();
user.setFirstName("zzzz");
Binder.get(applicationEnvironment).bind("user", Bindable.ofInstance(user));
System.out.println(user);

```

### 绑定SpringApplication

```java

SpringApplication springApplication = new SpringApplication();
System.out.println(springApplication);
Binder.get(applicationEnvironment).bind("spring.main", Bindable.ofInstance(springApplication));
// debug查看字段变更
System.out.println(springApplication);

```

# banner

```java

SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter(
        new DefaultResourceLoader(),
        new SpringBootBanner()
);
ApplicationEnvironment applicationEnvironment = new ApplicationEnvironment();

// 文字banner
applicationEnvironment.getPropertySources().addLast(new MapPropertySource("custom", ImmutableMap.of("spring.banner.location", "banner1.txt")));

// 图片banner
//applicationEnvironment.getPropertySources().addLast(new MapPropertySource("custom", ImmutableMap.of("spring.banner.image.location", "banner1.png")));

System.out.println("SpringBootVersion:" + SpringBootVersion.getVersion());
bannerPrinter.print(applicationEnvironment, Step04.class, System.out);

```
