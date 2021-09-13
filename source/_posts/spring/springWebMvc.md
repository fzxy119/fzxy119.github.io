---
title: SpringWebMvc
date: 2021-09-07 16:31:18
tags:
- java
- springmvc
- spring
categories:
- 技术
---

SpringWebMVC是构建在ServletAPI上的最初的Web框架，从一开始就包含在Spring框架中。正式名称“SpringWebMVC”来自其源模块(Spring-webmvc)的名称，但它更常见的名称是“SpringMVC”。与SpringWebMVC并行，SpringFramework5.0引入了一个反应性堆栈Web框架，它的名称“SpringWebFlux”也是基于它的源模块(SpringWebFlux)。

与许多其他Web框架一样，SpringMVC是围绕前端控制器模式设计的，其中一个中心servlet DispatcherServlet为请求处理提供了一个共享算法，而实际工作则由可配置的委托组件执行。该模型具有灵活性，支持多种工作流。与任何servlet一样，DispatcherServlet需要使用Java配置或web.xml根据servlet规范进行声明和映射。反过来，DispatcherServlet使用Spring配置来发现请求映射、视图解析、异常处理等所需的委托组件。

SpringBoot遵循不同的初始化顺序。SpringBoot没有连接到servlet容器的生命周期，而是使用Spring配置引导自身和嵌入式servlet容器。`Filter`和`Servlet`声明在Spring配置中,容器检测到并在servlet容器中注册. 

传统的Spring和Springmvc依赖于servlet容器的生命周期，通过org.springframework.web.context.ContextLoaderListener 容器监听器和org.springframework.web.servlet.DispatcherServlet 注册 来完成容器的启动 这与SpringBoot 通过容器初始化，通过工厂方式创建WebServer ，然后监听ServletContext ,并回调

# 上下文层次结构

DispatcherServlet 可以直接与 ServletContext 进项关联 xml，或者借助 WebApplicationContext 与 ServletContext 进项关联(JavaConfig) ，如果应用程序想要获取WebApplicationContext，可通过RequestContextUtils 工具类获取到它.通常情况下WebApplicationContext 有一个上下文层次结构，应用程序环境有父子容器关系，根`ApplicationContext`通常包含基础设施bean（扫描service包）,这些bean实际上是继承的，可以在WebApplicationContext （扫描controller包）特定的子级中重写(即重新声明),下图显示了这种关系，

![](/images/mvc-context-hierarchy.png)



# DispatcherServlet

DispatcherServlet 就像Servlet 链接到ServletContext 有多重方式，支持web.xml,或者JAVA 配置方式 来进项链接.（Tomcat容器方式）

## web.xml 方式类链接 DispatcherServlet 

```
<web-app>
     <!--ROOTApplicationContext-->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring.xml,classpath:dubbo-consumer.xml</param-value>
    </context-param>
      <!--注册DispatcherServlet-->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>

</web-app>
```

## WebApplicationInitializer 方式类链接 DispatcherServlet 

现在JavaConfig配置方式在逐步取代xml配置方式。而WebApplicationInitializer可以看做是Web.xml的替代，它是一个接口。通过实现WebApplicationInitializer，在其中可以添加servlet，listener等，在加载Web项目的时候会加载这个接口实现类，从而起到web.xml相同的作用

```
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {
    
        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        //可采用注解的方式替代xml配置
        context.register(AppConfig.class);
        
        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/*");
    }
}
```

或者 采用xml

```
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring-mvc.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/*");
    }
}
```



# 特殊的Bean

DispatcherServlet接受请求并强请求委派给SpringMvc中定义的相应功能Bean进项处理，这些Bean担任不同的角色，有者不同的处理能力，这些通常带有内置的契约，我们可以自定义他们的属性或者替换他们来完成我们的业务需求

| Bean                                 | 说明                                                         |
| :----------------------------------- | :----------------------------------------------------------- |
| HandlerMapping                       | 将请求映射到处理程序以及生成拦截器处理链。映射是基于一些标准，其细节取决于HandlerMapping实现。两主HandlerMapping实现是RequestMappingHandlerMapping(支持@RequestMapping)和SimpleUrlHandlerMapping(它维护对处理程序的URI路径模式的显式注册)。 |
| HandlerAdapter                       | 调用映射到请求的处理程序，可适配多种具体的执行方式，如处理程序为handlerMethod Controller HandlerFunction  Servlet HttpRequestHandler，HandlerAdapter主要目的是隐藏调用细节，简化DispatcherServlet，让DispatcherServlet专注于流程控制 |
| HandlerExceptionResolver             | mvc中解决异常的策略，可以将程序中的执行异常，映射到处理程序，HTML试图，或者其他结果，常见的HandlerExceptionResolver实现有，SimpleMappingExceptionResolver，DefaultHandlerExceptionResolver，ResponseStatusExceptionResolver，ExceptionHandlerExceptionResolver |
| ViewResolver                         | 如果处理程序返回视图数据模型，那么ViewResolver负责对视图数据模型进行解析,可根据具体的请求接受类型，返回最适合的视图View，并由View进行视图的渲染 |
| LocaleResolver,LocaleContextResolver | 解析Locale客户端正在使用并可能使用他们的时区，以便能够提供国际化的视图 |
| ThemeResolver                        | 解决您的Web应用程序可以使用的主题                            |
| MultipartResolver                    | 抽象用于解析多部分请求(例如，浏览器表单文件上传)             |
| FlashMapManager                      | 存储和检索“输入”和“输出”FlashMap它可以用于将属性从一个请求传递到另一个请求 |

# doDispatch执行流程:

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
         .....
		try {
			ModelAndView mv = null;
			Exception dispatchException = null;
			try {
			    //-------MultipartResolver 解析并装饰 request
				processedRequest = checkMultipart(request);
				//-------遍历HandlerMapping 并找到匹配的 处理链 HandlerExecutionChain
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {noHandlerFound(processedRequest, response);return;}
				//-------遍历HandlerAdapter 找到支持执行程序的适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
				//-------applyPreHandle 拦截器前置处理执行
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {return;}
				//-------执行器执行
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
				 //-------applyPostHandle 拦截器后置处理执行
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {dispatchException = ...;}
			catch (Throwable err) {dispatchException =...;}
			 //-------异常解析，视图解析 ，视图渲染  拦截器afterCompletion处理执行
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
		     //-------拦截器afterCompletion处理执行
			triggerAfterCompletion
		}
		catch (Throwable err) {
		   //-------拦截器afterCompletion处理执行
			triggerAfterCompletion
		}
}
```

# 问题：

1. HandlerMapping实现都有哪些，何时被初始化  如何使用扩展
2. HandlerAdapter实现都有哪些，何时被初始化 如何使用扩展
3. HandlerExceptionResolver 实现都有哪些，何时被初始化  如何使用扩展            
4. ViewResolver   实现都有哪些，何时被初始化   如何使用扩展      
5. LocaleResolver,LocaleContextResolver        实现都有哪些，何时被初始化 如何使用扩展            
6. ThemeResolver 实现都有哪些，何时被初始化 如何使用扩展          
7. MultipartResolver   实现都有哪些，何时被初始化 如何使用扩展          
8. FlashMapManager 实现都有哪些，何时被初始化 如何使用扩展          
