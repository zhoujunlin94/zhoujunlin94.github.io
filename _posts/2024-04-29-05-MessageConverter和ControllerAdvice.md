---
layout:     post
title:      05-MessageConverter和ControllerAdvice
subtitle:   MessageConverter和ControllerAdvice
date:       2024-04-29
author:     zhoujunlin
header-img: img/home-bg.jpg
catalog: true
tags:
    - SpringMVC
---


准备对象
```java
@Data
static class User {
    private String name;
    private int age;

    @JsonCreator   // 默认jackson会使用无参构造器反序列化  这里强制使用当前带参构造器
    public User(@JsonProperty("name") String name, @JsonProperty("age") int age) {
        this.name = name;
        this.age = age;
    }
}
```
# MessageConverter
## 序列化-JSON
```java
@SneakyThrows
private static void test1() {
    MockHttpOutputMessage outputMessage = new MockHttpOutputMessage();
    MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();

    if (converter.canWrite(User.class, MediaType.APPLICATION_JSON)) {
        converter.write(new User("张三", 30), MediaType.APPLICATION_JSON, outputMessage);
        System.out.println("outputMessage.getBodyAsString() = " + outputMessage.getBodyAsString());
    }
}
```
## 序列化-XML
```java
@SneakyThrows
private static void test2() {
    MockHttpOutputMessage outputMessage = new MockHttpOutputMessage();
    MappingJackson2XmlHttpMessageConverter converter = new MappingJackson2XmlHttpMessageConverter();

    if (converter.canWrite(User.class, MediaType.APPLICATION_XML)) {
        converter.write(new User("张三", 30), MediaType.APPLICATION_XML, outputMessage);
        System.out.println("outputMessage.getBodyAsString() = " + outputMessage.getBodyAsString());
    }
}
```
## 序列化优先级
```java
@ResponseBody
// @RequestMapping(produces = MediaType.APPLICATION_JSON_VALUE)
public User user() {
    return null;
}


@SneakyThrows
public static void test4() {
    // 模拟请求
    MockHttpServletRequest request = new MockHttpServletRequest();
    MockHttpServletResponse response = new MockHttpServletResponse();
    ServletWebRequest servletWebRequest = new ServletWebRequest(request, response);

    // 配置请求头 以及 响应类型
    request.addHeader("Accept", "application/xml");
    response.setContentType("application/json");

    // 设置ResponseBody处理的类型转换器
    RequestResponseBodyMethodProcessor requestResponseBodyMethodProcessor = new RequestResponseBodyMethodProcessor(
            CollUtil.newArrayList(
                    new MappingJackson2HttpMessageConverter(),
                    new MappingJackson2XmlHttpMessageConverter()
            )
    );

    requestResponseBodyMethodProcessor.handleReturnValue(
            new User("张三", 30),
            new MethodParameter(TestMessageConverter.class.getMethod("user"), -1),
            new ModelAndViewContainer(),
            servletWebRequest
    );
        
    System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));
}
   
```

@ResponseBody 是由返回值类型处理器完成解析的，具体转换工作有消息转换器完成  MappingJackson2HttpMessageConverter<br />具体转换为json还是xml?

- 响应头里有没有设置response.setContentType 或者 @RequestMapping(produces = MediaType.APPLICATION_JSON_VALUE)
- request的请求头里Accept参数
- RequestResponseBodyMethodProcessor中消息转换器的顺序
## 反序列化

```java

@SneakyThrows
public static void test3() {
    MockHttpInputMessage inputMessage = new MockHttpInputMessage("{\"name\":\"张三\", \"age\": 30}".getBytes(StandardCharsets.UTF_8));

    MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
    if (converter.canRead(User.class, MediaType.APPLICATION_JSON)) {
        User readUser = (User) converter.read(User.class, inputMessage);
        System.out.println(readUser);
    }

}

```

# ControllerAdvice
准备对象

```java

@Data
@NoArgsConstructor
@AllArgsConstructor
public static class User {
    private String name;
    private int age;
}

```

准备控制器与ControllerAdvice
```java
@Controller
public static class MyController {

    @ResponseBody
    public User user() {
        return new User("王五", 30);
    }

}

@ControllerAdvice
public static class MyControllerAdvice implements ResponseBodyAdvice<Object> {

    // 满足条件才转换
    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    // 将返回对象转换为统一格式
    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        if (body instanceof JsonResponse) {
            return body;
        }
        return JsonResponse.ok(body);
    }
}
```
# MVC执行流程
```java

// 1. 准备容器
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(WebConfig.class);

// 2. 准备请求
MockHttpServletRequest request = new MockHttpServletRequest();
MockHttpServletResponse response = new MockHttpServletResponse();
ServletWebRequest servletWebRequest = new ServletWebRequest(request, response);

// 3. 准备servletInvocableHandlerMethod
ServletInvocableHandlerMethod servletInvocableHandlerMethod = new ServletInvocableHandlerMethod(
        applicationContext.getBean(WebConfig.MyController.class),
        WebConfig.MyController.class.getDeclaredMethod("user"));
// 这里没有特殊参数绑定  入参传null
ServletRequestDataBinderFactory servletRequestDataBinderFactory = new ServletRequestDataBinderFactory(Collections.emptyList(), null);
servletInvocableHandlerMethod.setDataBinderFactory(servletRequestDataBinderFactory);
servletInvocableHandlerMethod.setParameterNameDiscoverer(new DefaultParameterNameDiscoverer());
servletInvocableHandlerMethod.setHandlerMethodArgumentResolvers(argumentResolverComposite(applicationContext));
servletInvocableHandlerMethod.setHandlerMethodReturnValueHandlers(returnValueHandlerComposite(applicationContext));

// 4. 准备ModelAndViewContainer容器
ModelAndViewContainer modelAndViewContainer = new ModelAndViewContainer();

// 5. 方法执行
servletInvocableHandlerMethod.invokeAndHandle(
        servletWebRequest,
        modelAndViewContainer);

System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));

// 6. 关闭容器
applicationContext.close();

```

参数解析器与之前一致，主要是返回值处理器中的RequestResponseBodyMethodProcessor 添加自定义ControllerAdvice

```java
private static HandlerMethodReturnValueHandlerComposite returnValueHandlerComposite(ApplicationContext applicationContext) {
    MappingJackson2HttpMessageConverter jackson2HttpMessageConverter = new MappingJackson2HttpMessageConverter();
    HandlerMethodReturnValueHandlerComposite returnValueHandlerComposite = new HandlerMethodReturnValueHandlerComposite();
    returnValueHandlerComposite.addHandlers(CollUtil.newArrayList(
            // 返回值：ModelAndView
            new ModelAndViewMethodReturnValueHandler(),
            // 返回值字符串：视图名
            new ViewNameMethodReturnValueHandler(),
            // 返回模型数据  带@ModelAttribute注解
            new ServletModelAttributeMethodProcessor(false),
            // 返回值：HttpEntity
            new HttpEntityMethodProcessor(CollUtil.newArrayList(jackson2HttpMessageConverter)),
            // 返回值：HttpHeaders
            new HttpHeadersReturnValueHandler(),
            // 返回值json: 带@ResponseBody注解     添加自定义ControllerAdvice
            new RequestResponseBodyMethodProcessor(CollUtil.newArrayList(jackson2HttpMessageConverter),
                    CollUtil.newArrayList(applicationContext.getBean(WebConfig.MyControllerAdvice.class))),
            // 返回模型数据  不带@ModelAttribute注解
            new ServletModelAttributeMethodProcessor(true)

    ));
    return returnValueHandlerComposite;
}

```

关键代码<br />![image.png](https://cdn.jsdelivr.net/gh/zhoujunlin94/picture_bed/blog/1711288096547-a103820f-8a17-4c61-9a9a-a4436b819780.png)



