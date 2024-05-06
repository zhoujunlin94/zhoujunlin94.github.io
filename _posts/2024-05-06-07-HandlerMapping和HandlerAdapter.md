---
layout:     post
title:      07-HandlerMapping和HandlerAdapter
subtitle:   HandlerMapping和HandlerAdapter
date:       2024-05-06
author:     zhoujunlin
header-img: img/home-bg.jpg
catalog: true
tags:
    - SpringMVC
---

# BeanNameUrlHandlerMapping

```java

@Configuration
public class WebConfig01 {

    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        return new TomcatServletWebServerFactory(8080);
    }

    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    }

    @Bean
    public BeanNameUrlHandlerMapping beanNameUrlHandlerMapping() {
        return new BeanNameUrlHandlerMapping();
    }

    @Bean
    public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter() {
        return new SimpleControllerHandlerAdapter();
    }

    @Component("/c1")
    public static class Controller1 implements Controller {
        @Override
        public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
            response.getWriter().print("this is c1");
            // 返回null  不会渲染视图
            return null;
        }
    }

    // 不是/开头的bean 不处理
    @Component("c2")
    public static class Controller2 implements Controller {
        @Override
        public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
            response.getWriter().print("this is c2");
            return null;
        }
    }

    @Bean("/c3")
    public Controller controller3() {
        return new Controller() {
            @Override
            public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
                response.getWriter().print("this is c3");
                return null;
            }
        };
    }

}

```

## 测试

```java

public class TestBeanNameUrlHandler {

    public static void main(String[] args) {
        AnnotationConfigServletWebServerApplicationContext applicationContext =
                new AnnotationConfigServletWebServerApplicationContext(WebConfig01.class);

        /**
         * BeanNameUrlHandlerMapping 以/开头的bean名字作为映射路径
         * 这些bean本身作为handler, 需要实现Controller接口
         * SimpleControllerHandlerAdapter调用这些handler
         *
         * RequestMappingHandlerMapping 以@RequestMapping以及它的派生注解作为映射路径
         * 控制器具体方法作为handler
         * MyRequestMappingHandlerAdapter调用这些handler
         */

    }


}

```

# 自定义BeanNameUrlHandlerMapping

```java

@Configuration
public class WebConfig02 {

    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        return new TomcatServletWebServerFactory(8080);
    }

    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    }

    @Component
    static class MyBeanNameUrlHandlerMapping implements HandlerMapping {

        @Override
        public HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
            Controller controller = beanNameUrlControllerMap.get(request.getRequestURI());
            if (Objects.isNull(controller)) {
                // 404
                return null;
            }
            return new HandlerExecutionChain(controller);
        }

        @Autowired
        private ApplicationContext applicationContext;
        private Map<String, Controller> beanNameUrlControllerMap;

        @PostConstruct
        public void init() {
            Map<String, Controller> beanNameUrlControllerMap = applicationContext.getBeansOfType(Controller.class).entrySet().stream()
                    .filter(entry -> entry.getKey().startsWith("/")).collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
            System.out.println("beanNameUrlControllerMap======>");
            System.out.println(beanNameUrlControllerMap);
            this.beanNameUrlControllerMap = beanNameUrlControllerMap;
        }
    }

    static class MyControllerHandlerAdapter implements HandlerAdapter {
        @Override
        public boolean supports(Object handler) {
            return handler instanceof Controller;
        }

        @Override
        public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
            Controller controller = (Controller) handler;
            controller.handleRequest(request, response);
            // 返回null  不会渲染视图
            return null;
        }

        @Override
        public long getLastModified(HttpServletRequest request, Object handler) {
            return -1;
        }

    }

    @Bean
    public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter() {
        return new SimpleControllerHandlerAdapter();
    }

    @Component("/c1")
    public static class Controller1 implements Controller {
        @Override
        public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
            response.getWriter().print("this is c1");
            return null;
        }
    }

    @Component("c2")
    public static class Controller2 implements Controller {
        @Override
        public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
            response.getWriter().print("this is c2");
            return null;
        }
    }

    @Bean("/c3")
    public Controller controller3() {
        return new Controller() {
            @Override
            public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
                response.getWriter().print("this is c3");
                return null;
            }
        };
    }

}

```

# RouterFunctionMapping

```java

@Configuration
public class WebConfig03 {

    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        return new TomcatServletWebServerFactory(8080);
    }

    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    }

    @Bean
    public RouterFunctionMapping routerFunctionMapping() {
        return new RouterFunctionMapping();
    }

    @Bean
    public HandlerFunctionAdapter handlerFunctionAdapter() {
        return new HandlerFunctionAdapter();
    }

    @Bean
    public RouterFunction<ServerResponse> r1() {
        return RouterFunctions.route(
                RequestPredicates.GET("/r1"),
                new HandlerFunction<ServerResponse>() {
                    @Override
                    public ServerResponse handle(ServerRequest request) throws Exception {
                        return ServerResponse.ok().body("this is r1");
                    }
                });
    }

    @Bean
    public RouterFunction<ServerResponse> r3() {
        return RouterFunctions.route(
                RequestPredicates.GET("/r2"),
                new HandlerFunction<ServerResponse>() {
                    @Override
                    public ServerResponse handle(ServerRequest request) throws Exception {
                        return ServerResponse.ok().body("this is r2");
                    }
                });
    }

}

```

## 测试

```java

public class TestRouterFunctionHandler {

    public static void main(String[] args) {
        AnnotationConfigServletWebServerApplicationContext applicationContext =
                new AnnotationConfigServletWebServerApplicationContext(WebConfig03.class);

        /**
         * RouterFunctionMapping 通过 RequestPredicates 映射路径
         * handler需要实现HandlerFunction接口
         * HandlerFunctionAdapter调用这些handler
         *
         * RequestMappingHandlerMapping 以@RequestMapping以及它的派生注解作为映射路径
         * 控制器具体方法作为handler
         * MyRequestMappingHandlerAdapter调用这些handler
         */

    }


}


```

# SimpleUrlHandlerMapping

```java

@Configuration
public class WebConfig04 {

    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
        return new TomcatServletWebServerFactory(8080);
    }

    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    }

    @Bean
    public SimpleUrlHandlerMapping simpleUrlHandlerMapping(ApplicationContext applicationContext) {
        SimpleUrlHandlerMapping simpleUrlHandlerMapping = new SimpleUrlHandlerMapping();
        Map<String, ResourceHttpRequestHandler> resourceHttpRequestHandlerMap = applicationContext.getBeansOfType(ResourceHttpRequestHandler.class);
        simpleUrlHandlerMapping.setUrlMap(resourceHttpRequestHandlerMap);
        return simpleUrlHandlerMapping;
    }

    @Bean
    public HttpRequestHandlerAdapter httpRequestHandlerAdapter() {
        return new HttpRequestHandlerAdapter();
    }


    /**
     * 映射  /test1.html   /test2.html
     */
    @Bean("/**")
    public ResourceHttpRequestHandler htmlHandler() {
        ResourceHttpRequestHandler resourceHttpRequestHandler = new ResourceHttpRequestHandler();
        resourceHttpRequestHandler.setLocations(CollUtil.newArrayList(
                new ClassPathResource("static/")
        ));
        resourceHttpRequestHandler.setResourceResolvers(CollUtil.newArrayList(
                // 缓存查找
                new CachingResourceResolver(new ConcurrentMapCache("resourceCache")),
                // 压缩文件查找
                new EncodedResourceResolver(),
                // 从磁盘查找
                new PathResourceResolver()
        ));
        return resourceHttpRequestHandler;
    }

    /**
     * 映射  /image/1.png   /image/2.png
     */
    @Bean("/image/**")
    public ResourceHttpRequestHandler imageHandler() {
        ResourceHttpRequestHandler resourceHttpRequestHandler = new ResourceHttpRequestHandler();
        resourceHttpRequestHandler.setLocations(CollUtil.newArrayList(
                new ClassPathResource("image/")
        ));
        return resourceHttpRequestHandler;
    }


}

```

## 测试

```java

public class TestRouterFunctionHandler {

    public static void main(String[] args) {
        AnnotationConfigServletWebServerApplicationContext applicationContext =
                new AnnotationConfigServletWebServerApplicationContext(WebConfig03.class);

        /**
         * RouterFunctionMapping 通过 RequestPredicates 映射路径
         * handler需要实现HandlerFunction接口
         * HandlerFunctionAdapter调用这些handler
         *
         * RequestMappingHandlerMapping 以@RequestMapping以及它的派生注解作为映射路径
         * 控制器具体方法作为handler
         * MyRequestMappingHandlerAdapter调用这些handler
         */

    }


}

```
# 小结

1. HandlerMapping 负责建立请求与控制器之间的映射关系
   1. RequestMappingHandlerMapping (与@RequestMapping及其派生注解匹配)
   2. WelcomePageHandlerMapping (/)
   3. BeanNameUrlHandlerMapping (与bean 的名字匹配   以/开头)
   4. RouterFunctionMapping (函数式RequestPredicate, HandlerFunction)
   5. SimpleUrlHandlerMapping (静态资源 通配符 /**  /img/**)

HandlerMapping之间也会有顾序问题，SpringBoot中默认顺序如上

2. HandlerAdapter负责实现对各种各样的handler的适配调用
   1. RequestMappingHandlerAdapter处理: ReguestMapping方法参数解析器、返回值处理器体现了组合模式
   2. SimpleControllerHandlerAdapter处理: Controller接口
   3. HandlerFunctionAdapter处理: HandlerFunction函数式接口
   4. HttpRequestHandlerAdapter处理: HttpRequestHandler接口 (静态资源处理) 这也是典型适配器模式体现

3. ResourceHttpRequestHandler.setResourceResolvers 这是典型责任链模式体现

