---
title: 全局异常处理器HandlerExceptionResolver
date: 2021-09-19 22:28:36
tags:
- springmvc
- spring
categories:
- 技术
---

处理器异常解析器，是 DispatcherServlet 在请求转发**处理过程中出现异常**（请求映射处理器，处理链，处理适配器，处理器执行）的时候对异常进行处理的最后一步，，**如果抛出的异常没有被处理器异常解析器HandlerExceptionResolver**处理并且解析成**ModelAndView**，那么异常 将被抛出并交给上层容器来处理，如：我们在容器中配置的错误码映射视图

```xml
<web-app>
<!-- 根据状态码 -->
<error-page>
    <error-code>500</error-code>
    <location>/500.jsp</location>
</error-page>

<!-- 根据异常类型 -->
<error-page>
	<exception-type>java.lang.RuntimeException</exception-type>
	<location>/500.jsp</location>
</error-page>
    <welcome-file-list>
        <welcome-file>/</welcome-file>
        <welcome-file>/index</welcome-file>
    </welcome-file-list>
    
</web-app>
```

# SpringMvc异常处理方式

1. handler内部捕获处理，不往上层程序抛异常，单独处理

   ```java
   @RequestMapping("testExp")
   public String testExp()throws Exception{
       try {
           throw new Exception("a");
       } catch (Exception e) {
           e.printStackTrace();
           return "";
       }
   }
   ```

2. @ExceptionHandler注解方式。注解方式也有两种用法： 

   - 使用在`Controller`内部 ，可以在Controller**局部**异常统一处理

   ```java
   @RestController
   public class HalloWordController {
     //可以对HalloWordController 内部所有映射的处理器进行异常捕获，他只处理HalloWordController内部抛出的异常
       @ExceptionHandler(Exception.class)
       public String handlerException(Exception e){
           return "";
       }
   
   }
   ```

   - 配置`@ControllerAdvice`一起使用实现**全局**处理,或者**部分区域**异常处理，**取决于扫描包**

   ```java
   @ControllerAdvice(basePackages={"com.cx.user.controller"})
   public class HalloWordController {
       //可以对com.cx.user.controller包 内部所有映射的处理器进行异常捕获，他只处理HalloWordController内部抛出的异常
       @ExceptionHandler(Exception.class)
       public String handlerException(Exception e){
           return "";
       }
   
   }
   ```

   

3. 实现视图与异常进行映射处理SimpleMappingExceptionResolver

4. 实现`HandlerExceptionResolver`方式实现**全局**处理

# 全局异常处理器HandlerExceptionResolver

1. doDispatch请求处理程序抛出异常，包含了整个请求处理边界

2. 非ModelAndViewDefiningException异常的所有可捕获异常都将交给全局异常处理器里处理

3. 异常处理器**集合**返回view，并将view渲染给用户,如果所有的异常处理器都未给出要渲染的view，那MVC框架将不再处理这个异常，那这个异常将抛出到上层容器来处理。

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ...
    try {
      ModelAndView mv = null;
      Exception dispatchException = null;
      try {
          //请求处理器映射
          //处理链构造
          //处理链前置执行
          //执行handler
          mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
          //处理链后置执行
      }
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
      //这里对整个的处理记过进行处理,如果MVC内部处理流程出现异常，那么将在processDispatchResult 来处理
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {
     //...
   }
}
```

```java
//处理转发结果，异常解析，视图渲染
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
      @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
      @Nullable Exception exception) throws Exception {

   boolean errorView = false;
    //如果抛出异常将在这里处理
   if (exception != null) {
      if (exception instanceof ModelAndViewDefiningException) {
         mv = ((ModelAndViewDefiningException) exception).getModelAndView();
      }
      else {
         Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
          //异常处理并解析成对应的view，如果handler没有返回mv，执行的过程中抛出了异常，那么这里的异常处理器必须返回view，
          //如果不返回，异常将抛到上层容器，此处返回错误异常界面，后续程序将对异常界面进行渲染
         mv = processHandlerException(request, response, handler, exception);
         errorView = (mv != null);
      }
   }

   if (mv != null && !mv.wasCleared()) {
      render(mv, request, response);
      if (errorView) {
         WebUtils.clearErrorRequestAttributes(request);
      }
   }
   ......
}
```

```java
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
      @Nullable Object handler, Exception ex) throws Exception {
   ModelAndView exMv = null;
   if (this.handlerExceptionResolvers != null) {
       //遍历处理异常解析器，并进行处理，直到有一个返回了view
      for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
         exMv = resolver.resolveException(request, response, handler, ex);
         if (exMv != null) {
            break;
         }
      }
   }
   if (exMv != null) {
      //返回view
      return exMv;
   }
   //抛出异常,
   throw ex;
}
```

# 常用实现以及如何探测

- SimpleMappingExceptionResolver

  ​    根据异常类和视图之间的映射配置来判断是否处理异常，匹配成功就返回对应的视图和对应的响应吗

- ExceptionHandlerExceptionResolver

  ​	对处理器是HandlerMethod类型处理器异常进行处理，只处理一下两种情况

  ​		  1. 内部@ExceptionHandler注解的异常处理器，支持处理器内部区域内异常处理

  ​		  2. @ControllerAdvice支持全局或者部分区域（包扫描）内@ExceptionHandler注解的异常处理器

- DefaultHandlerExceptionResolver

  ​	处理mvc框架内部异常的处理

- ResponseStatusExceptionResolver

  ​	根据国际化资源对ResponseStatusException异常进行处理

- DefaultErrorAttributes

  ​     异常存储处理器不返回任何错误视图

## SpringBoot自动装配MVC中ExceptionResolver的装配

```properties
#异常处理自动装配
org.springframework.boot.autoconfigure.web.servlet.error.ErrorMvcAutoConfiguration
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
```

### 装配DefaultErrorAttributes和BasicErrorController

```java
//异常信息及解析器记录异常
@Bean
@ConditionalOnMissingBean(value = ErrorAttributes.class, search = SearchStrategy.CURRENT)
public DefaultErrorAttributes errorAttributes() {
		return new DefaultErrorAttributes();
}

@Bean
@ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
public BasicErrorController basicErrorController(ErrorAttributes errorAttributes, ObjectProvider<ErrorViewResolver> errorViewResolvers) {
   return new BasicErrorController(errorAttributes, this.serverProperties.getError(),errorViewResolvers.orderedStream().collect(Collectors.toList()));
}
```





### 装配ExceptionHandlerExceptionResolver和ResponseStatusExceptionResolver

```java
//WebMvcAutoConfiguration ->EnableWebMvcConfiguration ->DelegatingWebMvcConfiguration ->WebMvcConfigurationSupport
protected final void addDefaultHandlerExceptionResolvers(List<HandlerExceptionResolver> exceptionResolvers,
      ContentNegotiationManager mvcContentNegotiationManager) {

   ExceptionHandlerExceptionResolver exceptionHandlerResolver = createExceptionHandlerExceptionResolver();
   exceptionHandlerResolver.setContentNegotiationManager(mvcContentNegotiationManager);
   exceptionHandlerResolver.setMessageConverters(getMessageConverters());
   exceptionHandlerResolver.setCustomArgumentResolvers(getArgumentResolvers());
   exceptionHandlerResolver.setCustomReturnValueHandlers(getReturnValueHandlers());
   if (jackson2Present) {
      exceptionHandlerResolver.setResponseBodyAdvice(
            Collections.singletonList(new JsonViewResponseBodyAdvice()));
   }
   if (this.applicationContext != null) {
      exceptionHandlerResolver.setApplicationContext(this.applicationContext);
   }
   exceptionHandlerResolver.afterPropertiesSet();
    
    //注册ExceptionHandlerExceptionResolver，
    //处理@ExceptionHandler注解的内部异常处理器
    //@ControllerAdvice内@ExceptionHandler注解的异常处理器，支持全局或者部分区域内部（包扫描）异常处理，
   exceptionResolvers.add(exceptionHandlerResolver);

   ResponseStatusExceptionResolver responseStatusResolver = new ResponseStatusExceptionResolver();
   responseStatusResolver.setMessageSource(this.applicationContext);
   exceptionResolvers.add(responseStatusResolver);
   //注册ResponseStatusExceptionResolver
   exceptionResolvers.add(new DefaultHandlerExceptionResolver());
}
```

# 自定义全区异常处理器



### SimpleMappingExceptionResolver映射异常处理器

```java
@Order(Integer.MAX_VALUE)
@Component
public class MvcConfig implements WebMvcConfigurer , InitializingBean {

    private Properties exceptionMappings;

    private Properties statusCodes;


    //加载映射配置
    @Override
    public void afterPropertiesSet() throws Exception {
        exceptionMappings = new Properties();
        statusCodes = new Properties();
        exceptionMappings.load(new ClassPathResource("exceptionMappings.properties").getInputStream());
        statusCodes.load(new ClassPathResource("statusCodes.properties").getInputStream());
    }

  
    @Override
    public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        SimpleMappingExceptionResolver simpleMappingExceptionResolver = new SimpleMappingExceptionResolver();
        simpleMappingExceptionResolver.setExceptionMappings(exceptionMappings);
        simpleMappingExceptionResolver.setStatusCodes(statusCodes);
        simpleMappingExceptionResolver.setDefaultErrorView("error");
        simpleMappingExceptionResolver.setDefaultStatusCode(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
        resolvers.add(simpleMappingExceptionResolver);
    }
}
```

异常视图映射配置：exceptionMappings.properties

```properties
异常对应视图
java.lang.Exception=500
```

视图响应码配置：statusCodes.properties

```properties
#视图对应响应码
500=500
```

### 自定义实现HandlerExceptionResolver异常处理

```java
@Component
public class GobalExceptionResolver implements HandlerExceptionResolver {
    
    
public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response,
                                     Object handler,Exception ex) {
        String msg = "服务暂停";
        //如果是HandlerMethod，那么检查是否是返回ResponseBody，确定是返回界面还是json数据
        if(ex!=null && handler instanceof HandlerMethod){
            HandlerMethod hdlm =  (HandlerMethod)handler;
            ResponseBody have_rb = AnnotationUtils.findAnnotation(hdlm.getBeanType(),ResponseBody.class);
            //返回json视图
            if(hdlm.hasMethodAnnotation(ResponseBody.class) || have_rb!=null){
                ModelAndView mv =   new ModelAndView();
                MappingJackson2JsonView mappingJackson2JsonView = new MappingJackson2JsonView();
                mv.setView(mappingJackson2JsonView);
                mv.addObject("msg",msg);
                return mv;
           //返回view视图 html     
            }else{
                return new ModelAndView("error");
            }
        }
        //返回模式视图
        return new ModelAndView("error");
    }
}
```
