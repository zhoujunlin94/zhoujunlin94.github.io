---
layout:     post
title:      01-BeanFactory
subtitle:   认识Bean工厂
date:       2024-04-25
author:     zhoujunlin
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Spring
---


<a name="LIoTb"></a>
# 准备Bean
```java
interface BaseBean {
}

class Bean1 implements BaseBean {

    public Bean1() {
        System.out.println("构造Bean1()");
    }

}

class Bean2 {

    @Autowired
    @Resource(name = "bean1")
    private BaseBean bean3;

    public Bean2() {
        System.out.println("构造Bean2()");
    }

    public BaseBean getBean() {
        return bean3;
    }
}

class Bean3 implements BaseBean {

    public Bean3() {
        System.out.println("构造Bean3()");
    }

}

@Configuration
class BeanConfig {

    @Bean
    public Bean1 bean1() {
        return new Bean1();
    }

    @Bean
    public Bean2 bean2() {
        return new Bean2();
    }

    @Bean
    public Bean3 bean3() {
        return new Bean3();
    }

}
```
<a name="DUIEb"></a>
# 对BeanFactory操作
<a name="bN4we"></a>
## 注册配置类Bean
```java
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

// Bean的定义  class  scope
AbstractBeanDefinition configBeanDefinition = BeanDefinitionBuilder.genericBeanDefinition(BeanConfig.class)
.setScope(BeanDefinition.SCOPE_SINGLETON).getBeanDefinition();
beanFactory.registerBeanDefinition("beanConfig", configBeanDefinition);
// 到这里 Bean容器中只有beanConfig一个bean  也就是说@Bean并未生效  
```
<a name="by6yC"></a>
## 注册一些注解相关的BeanFactoryProcessor和BeanPostProcessor
```java
/**
 * 给BeanFactory添加后置处理器 例如:
 * BeanFactoryPostProcessor: internalConfigurationAnnotationProcessor(处理注解bean配置)   
 * BeanPostProcessor: internalAutowiredAnnotationProcessor(处理@Autowired注入  优先级高于@Resource)
 * BeanPostProcessor: internalCommonAnnotationProcessor(处理@Resource注入)
 */
AnnotationConfigUtils.registerAnnotationConfigProcessors(beanFactory);

// 到这里 Bean容器中只有beanConfig以及上面的内置bean
```
<a name="TEuWo"></a>
## 执行BeanFactoryProcessor补充BeanDefinition
```java
 // 应用已经加入的BeanFactory后置处理器  补充一些Bean定义
beanFactory.getBeansOfType(BeanFactoryPostProcessor.class).values().forEach(beanFactoryPostProcessor -> {
    beanFactoryPostProcessor.postProcessBeanFactory(beanFactory);
});

// 到这里已经有了Bean1和Bean2了  但是注入类的注解这里还没有生效即 @Autowired 和@Resource未生效
for (String beanDefinitionName : beanFactory.getBeanDefinitionNames()) {
    System.out.println("beanDefinitionName=====>" + beanDefinitionName);
}
```
<a name="Fs9X8"></a>
# 对Bean操作
```java
// 给BeanFactory添加bean后置处理器 针对bean的各个生命周期进行扩展  例如@Autowired @Resource注解处理
beanFactory.getBeansOfType(BeanPostProcessor.class).values().stream()
        // 默认@Autowired优先级高  这里排序目的将@Resource优先级提在了@Autowired之前了
        //.sorted(AnnotationAwareOrderComparator.INSTANCE)
        .forEach(beanFactory::addBeanPostProcessor);
// 此时@Autowired @Resource注解生效

// 默认getBean的时候才会实例化Bean   这里可以调用此方法提前准备单例bean
beanFactory.preInstantiateSingletons();

if (beanFactory.containsBean("bean2")) {
    System.out.println("--------->");
    System.out.println(beanFactory.getBean(Bean2.class).getBean());
}
```
<a name="iZwpB"></a>
# 小结
BeanFactory：

- 不会主动注册并调用BeanFactory后置处理器
- 不会主动注册Bean后置处理器   在BeanFactory初始化Bean的时候调用
- 不会主动实例化Bean
- 不会解析#{}  ${}

BeanFactoryPostProcessor：发现处理并添加BeanDefinition<br />BeanPostProcessor：对Bean进行扩展增强



