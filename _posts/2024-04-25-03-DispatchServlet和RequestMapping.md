---
layout:     post
title:      01-DispatchServlet和RequestMapping
subtitle:   认识DispatchServlet和RequestMapping
date:       2024-04-25
author:     zhoujunlin
header-img: img/home-bg.jpg
catalog: true
tags:
    - SpringMVC
---


<a name="UquAQ"></a>
# SpringMVC启动
```properties
server.port=8090
spring.mvc.servlet.load-on-startup=1
```
```java
@Configuration
@ComponentScan
//  引入配置
@PropertySource("classpath:application.properties")
@EnableConfigurationProperties({ServerProperties.class, WebMvcProperties.class})
public class WebConfig {

    /**
     * 内嵌web容器工厂
     */
    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory(ServerProperties serverProperties) {
        return new TomcatServletWebServerFactory(serverProperties.getPort());
    }

    @Bean
    public DispatcherServlet dispatcherServlet() {
        // initStrategies 初始化逻辑
        return new DispatcherServlet();
    }

    /**
     * 注册DispatchServlet  springmvc入口
     */
    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(
        DispatcherServlet dispatcherServlet,
        WebMvcProperties webMvcProperties
    ) {
        DispatcherServletRegistrationBean registrationBean = new DispatcherServletRegistrationBean(dispatcherServlet, "/");
        // 默认DispatcherServlet是在第一次访问应用时初始化的  这里优先级设置为1  让DispatcherServlet启动时初始化
        registrationBean.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
        return registrationBean;
    }

}

```
```java
public static void main(String[] args) {
    AnnotationConfigServletWebServerApplicationContext applicationContext =
            new AnnotationConfigServletWebServerApplicationContext(WebConfig.class);

}
```
<a name="xlMrt"></a>
# HandlerMethod 
<a name="bExRZ"></a>
## 准备自定义注解
```java
/**
 * 自定义参数注解：用于将请求头中的token参数解析出来并放入方法入参（参数解析器）
 */
@Target({ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Token {
}

/**
 * 自定义方法返回值注解  用户测试自定义方法返回值处理器
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Yml {
}

```
<a name="FO6jy"></a>
## 准备控制器
```java
@Slf4j
@Controller
public class TestController {


    /**
     * 测试get请求
     *
     * @return
     */
    @GetMapping("/test1")
    public ModelAndView test1() {
        log.debug("test1");
        return null;
    }

    /**
     * 测试带入参注解的post请求
     *
     * @param name
     * @return
     */
    @PostMapping("/test2")
    public ModelAndView test2(@RequestParam String name) {
        log.debug("test2({})", name);
        return null;
    }

    /**
     * 测试带自定义注解的入参处理器   put请求
     *
     * @param token
     * @return
     */
    @PutMapping("/test3")
    public ModelAndView test3(@Token String token) {
        log.debug("test3({})", token);
        return null;
    }

    /**
     * 不定义请求类型  自定义返回类型处理器
     *
     * @return
     */
    @RequestMapping("/test4")
    @Yml
    public User test4() {
        log.debug("test4()");
        return new User("张三", 30);
    }

}

@Data
@AllArgsConstructor
class User {
    private String name;
    private int age;

    public static void main(String[] args) {
        String userYaml = new Yaml().dump(new User("张三", 30));
        System.out.println(userYaml);
    }

}
```
<a name="btlYd"></a>
## 准备RequestMappingHandlerMapping 
```java
/**
 * 如果使用DispatcherServlet.properties配置文件中的默认RequestMappingHandlerMapping，其不会作为spring的bean
 * 不方便后续测试  这里new一个RequestMappingHandlerMapping并存入spring容器  方便后续测试
 */
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping() {
    return new RequestMappingHandlerMapping();
}

```
<a name="Dcdpq"></a>
## 测试
```java
public static void testHandlerMethod(ApplicationContext applicationContext) {
    // 作用: 解析@RequestMapping以及其派生注解  生成请求路径与处理方法的映射关系  在初始化时就生成了
    RequestMappingHandlerMapping handlerMapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
    Map<RequestMappingInfo, HandlerMethod> handlerMethods = handlerMapping.getHandlerMethods();

    handlerMethods.forEach((k, v) -> {
        System.out.println(k + "=" + v);
    });

    /**
     * {GET [/test1]}=com.you.meet.nice.test.web.springmvc.dispatchservlet.TestController#test1()
     * {POST [/test2]}=com.you.meet.nice.test.web.springmvc.dispatchservlet.TestController#test2(String)
     * { [/test4]}=com.you.meet.nice.test.web.springmvc.dispatchservlet.TestController#test4()
     * {PUT [/test3]}=com.you.meet.nice.test.web.springmvc.dispatchservlet.TestController#test3(String)
     */

}
```
<a name="U73Xr"></a>
# RequestMappingHandlerAdapter 
<a name="MClct"></a>
## 准备自定义RequestMappingHandlerAdapter  Bean
```java
/**
 * @author zhoujunlin
 * @date 2024/3/14 22:12
 * @desc 继承RequestMappingHandlerAdapter，重新invokeHandlerMethod方法
 * 因为此方法是protected的，外部无法调用  继承后修改为public便于测试
 */
public class MyRequestMappingHandlerAdapter extends RequestMappingHandlerAdapter {

    @Override
    public ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        return super.invokeHandlerMethod(request, response, handlerMethod);
    }
}


/**
 * 继承RequestMappingHandlerAdapter，并放入Bean容器  替换DispatcherServlet.properties配置中的默认RequestMappingHandlerAdapter
 */
@Bean
public MyRequestMappingHandlerAdapter myRequestMappingHandlerAdapter() {
    MyRequestMappingHandlerAdapter handlerAdapter = new MyRequestMappingHandlerAdapter();
    return handlerAdapter;
}
```
<a name="osNXj"></a>
## test1
```java
@SneakyThrows
public static void test1(ApplicationContext applicationContext) {
    RequestMappingHandlerMapping handlerMapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
    MockHttpServletRequest request = new MockHttpServletRequest("GET", "/test1");
    MockHttpServletResponse response = new MockHttpServletResponse();
    // 获取Mock请求对应的处理器方法  放回执行器链对象
    HandlerExecutionChain handlerExecutionChain = handlerMapping.getHandler(request);
    System.out.println("handlerExecutionChain = " + handlerExecutionChain);
    /**
     * 对应一个处理器方法和0个拦截器
     * HandlerExecutionChain with [com.you.meet.nice.test.web.springmvc.dispatchservlet.TestController#test1()] and 0 interceptors
     */
    System.out.println("========>");
    MyRequestMappingHandlerAdapter handlerAdapter = applicationContext.getBean(MyRequestMappingHandlerAdapter.class);
    handlerAdapter.invokeHandlerMethod(request, response, ((HandlerMethod) handlerExecutionChain.getHandler()));
}

```
<a name="EoZcq"></a>
## test2
```java
@SneakyThrows
public static void test2(ApplicationContext applicationContext) {
    RequestMappingHandlerMapping handlerMapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
    MockHttpServletRequest request = new MockHttpServletRequest("POST", "/test2");
    request.addParameter("name", "zhou");
    MockHttpServletResponse response = new MockHttpServletResponse();
    HandlerExecutionChain handlerExecutionChain = handlerMapping.getHandler(request);
    System.out.println("========>");
    MyRequestMappingHandlerAdapter handlerAdapter = applicationContext.getBean(MyRequestMappingHandlerAdapter.class);
    handlerAdapter.invokeHandlerMethod(request, response, ((HandlerMethod) handlerExecutionChain.getHandler()));
}
```
<a name="WKJXK"></a>
## 获取所有参数解析器
```java
public static void testAllArgumentResolver(ApplicationContext applicationContext) {
    // 获取所有参数解析器
    MyRequestMappingHandlerAdapter handlerAdapter = applicationContext.getBean(MyRequestMappingHandlerAdapter.class);
    handlerAdapter.getArgumentResolvers().forEach(System.out::println);
}

```
<a name="JXimG"></a>
## 自定义入参解析器
```java
/**
 * @author zhoujunlin
 * @date 2024/3/14 22:23
 * @desc 自定义Token注解对应的入参解析器
 */
public class TokenArgumentResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        // 只有入参中有Token注解才支持解析
        Token token = parameter.getParameterAnnotation(Token.class);
        return Objects.nonNull(token);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        return webRequest.getHeader("token");
    }

}


/**
 * 继承RequestMappingHandlerAdapter，并放入Bean容器  替换DispatcherServlet.properties配置中的默认RequestMappingHandlerAdapter
 */
@Bean
public MyRequestMappingHandlerAdapter myRequestMappingHandlerAdapter() {
    MyRequestMappingHandlerAdapter handlerAdapter = new MyRequestMappingHandlerAdapter();

    TokenArgumentResolver tokenArgumentResolver = new TokenArgumentResolver();
    handlerAdapter.setCustomArgumentResolvers(CollUtil.newArrayList(tokenArgumentResolver));

    return handlerAdapter;
}

```
<a name="SosA3"></a>
## test3
```java
@SneakyThrows
public static void test3(ApplicationContext applicationContext) {
    RequestMappingHandlerMapping handlerMapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
    MockHttpServletRequest request = new MockHttpServletRequest("PUT", "/test3");
    request.addParameter("token", "混淆视听");
    request.addHeader("token", "某个token");
    MockHttpServletResponse response = new MockHttpServletResponse();
    HandlerExecutionChain handlerExecutionChain = handlerMapping.getHandler(request);
    System.out.println("========>");
    MyRequestMappingHandlerAdapter handlerAdapter = applicationContext.getBean(MyRequestMappingHandlerAdapter.class);
    handlerAdapter.invokeHandlerMethod(request, response, ((HandlerMethod) handlerExecutionChain.getHandler()));
}

```
<a name="jihJg"></a>
## 自定义返回值处理器
```java
public class YmlReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        Yml yml = returnType.getMethodAnnotation(Yml.class);
        // 只有方法上存在Yml注解才解析
        return Objects.nonNull(yml);
    }

    @Override
    public void handleReturnValue(Object returnValue, MethodParameter returnType, ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        // 1. 对象转yml
        String ret = new Yaml().dump(returnValue);

        // 2. 将结果写入响应
        HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
        response.setContentType("text/plain;charset=utf-8");
        response.getWriter().print(ret);

        // 3. 设置请求已处理完毕
        mavContainer.setRequestHandled(true);

    }
}

/**
 * 继承RequestMappingHandlerAdapter，并放入Bean容器  替换DispatcherServlet.properties配置中的默认RequestMappingHandlerAdapter
 */
@Bean
public MyRequestMappingHandlerAdapter myRequestMappingHandlerAdapter() {
    MyRequestMappingHandlerAdapter handlerAdapter = new MyRequestMappingHandlerAdapter();

    TokenArgumentResolver tokenArgumentResolver = new TokenArgumentResolver();
    handlerAdapter.setCustomArgumentResolvers(CollUtil.newArrayList(tokenArgumentResolver));

    YmlReturnValueHandler ymlReturnValueHandler = new YmlReturnValueHandler();
    handlerAdapter.setReturnValueHandlers(CollUtil.newArrayList(ymlReturnValueHandler));

    return handlerAdapter;
}

```
<a name="e5wx8"></a>
## test4
```java
@SneakyThrows
public static void test4(ApplicationContext applicationContext) {
    RequestMappingHandlerMapping handlerMapping = applicationContext.getBean(RequestMappingHandlerMapping.class);
    MockHttpServletRequest request = new MockHttpServletRequest("DELETE", "/test4");
    request.addParameter("token", "混淆视听");
    request.addHeader("token", "某个token");
    MockHttpServletResponse response = new MockHttpServletResponse();
    HandlerExecutionChain handlerExecutionChain = handlerMapping.getHandler(request);
    System.out.println("========>");
    MyRequestMappingHandlerAdapter handlerAdapter = applicationContext.getBean(MyRequestMappingHandlerAdapter.class);
    handlerAdapter.invokeHandlerMethod(request, response, ((HandlerMethod) handlerExecutionChain.getHandler()));

    byte[] ret = response.getContentAsByteArray();
    System.out.println(new String(ret, StandardCharsets.UTF_8));

}
```
