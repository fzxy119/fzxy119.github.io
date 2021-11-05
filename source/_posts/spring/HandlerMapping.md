---
title: SpringWebMvc-HandlerMapping
date: 2021-09-08 09:33:29
tags:
- springmvc
- spring
categories:
- 技术
---



   HandlerMapping 将请求映射到具体的处理程序（如HandlerMethod Controller HandlerFunction Servlet HttpRequestHandler等），并缓存映射关系，当请求进入DispatcherServlet后，DispatcherServlet 遍历 注册 HandlerMapping 根据请求情况获取HandlerExecutionChain，HandlerExecutionChain中包含了具体的处理程序和拦截器列表**，后续的执行中就直接使用了HandlerExecutionChain来处理，**此时HandlerMapping  就完成了任务，就是构建HandlerExecutionChain。

<!-- more -->

   请求是通过DispatcherServlet 来进行统一处理的，那么DispatcherServlet  是如何探测到HandlerMapping的呢，我们如何才能注册自己的HandlerMapping呢

   HandlerMapping 只定义了根据请求获取处理链的接口方法，那么HandlerMapping 主要目的就是构建处理链，提供具体的处理器和拦截器链，那么拦截器是如何注册，处理器是如何注册的，并且是如何通过url匹配到拦截器和处理器，并封装成HandlerExecutionChain 就是HandlerMapping 要做的事

   SprignMvc中能够处理请求的处理器有很多，那么都有哪些可用的线程的处理器呢，如果不使用现有的处理器类型，如何实现自己的处理器

```java
public interface HandlerMapping {
	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

# HandlerMapping探测

DispatcherServlet 继承自FrameworkServlet ，并且实现了onRefresh 监听WebApplicationContext刷新方法，完成MVC各个处理节点处理策略的初始化，其中HandlerMapping就是其中之一，

```java
protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
}
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```

具体探测步骤是

1. 根据detectAllHandlerMappings（是否探测所有的HandlerMapping）属性 true 完成 从bean容器中获取所有的HandlerMapping
2. 根据detectAllHandlerMappings（是否探测所有的HandlerMapping）属性 false 完成 从bean容器中获取beanName 为handlerMapping 类型为HandlerMapping
3. 如果都没有，就从默认DispatcherServlet.properties 配置中获取，spring-webmvc 包下配置了默认的策略规则HandlerMapping的默认策略规则为

```properties
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
   org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
   org.springframework.web.servlet.function.support.RouterFunctionMapping
```

```java
private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;
		if (this.detectAllHandlerMappings)
//探测所有的HandlerMapping
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<>(matchingBeans.values());
//执行order排序
				AnnotationAwareOrderComparator.sort(this.handlerMappings);
			}
		}
		else {
				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
		}
		if (this.handlerMappings == null) {
//默认策略配置
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
		}
	}
```



# HandlerInterceptor 探测

{% asset_img HandlerMappingClassH.png 'HandlerMapping结构图' %}

## HandlerMapping顶级实现类AbstractHandlerMapping

AbstractHandlerMapping是HandlerMapping的顶级实现类，所有的子类都直接或者间接的继承自这个类，AbstractHandlerMapping因为继承自WebApplicationObjectSupport抽象类，那么他就有了ServletContext （ServletContextAware setServletContext 接口方法）感知能力和ApplicationContext感知能力（initApplicationContext方法），**那么HandlerInterceptor 就是在AbstractHandlerMapping的initApplicationContext中直接从bean容器中检测HandlerInterceptor  类型的** 获取的类型为（**<u>MappedInterceptor</u>**）

```java
	protected void initApplicationContext() throws BeansException {
	
	//SkipPathExtensionContentNegotiation
		extendInterceptors(this.interceptors);
		
	//检测容器中的HandlerMapping 并缓存
		detectMappedInterceptors(this.adaptedInterceptors);
		
	//合并extendInterceptors和detectMappedInterceptors 到adaptedInterceptors 中进行缓存
		initInterceptors();
		
	}
	
	//检测容器中的HandlerMapping
	protected void detectMappedInterceptors(List<HandlerInterceptor> mappedInterceptors) {
		mappedInterceptors.addAll(BeanFactoryUtils.beansOfTypeIncludingAncestors(obtainApplicationContext(), 																					MappedInterceptor.class,true, false).values());
	}

```

**MappedInterceptor** 内部缓存了匹配资源和排除匹配资源，并且支持提供匹配函数来对请求的资源进行匹配判断，支持针对不同的资源进行不同的拦截，通常情况下我们使用的就是这种匹配型拦截器，定义HandlerInterceptor   并把它放入MappedInterceptor，并配置MappedInterceptor的匹配属性，进行使用。

匹配型拦截器：includePatterns ，excludePatterns interceptor，

非匹配性拦截器： interceptor

构建HandlerExecutionChain: 来收集请求对应的拦截器，如果是匹配型的就进行请求匹配，否自直接添加到处理链中

```java
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
   HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
         (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
   String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
   for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
      if (interceptor instanceof MappedInterceptor) {
         MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
          //如果是匹配型就先匹配
         if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
            chain.addInterceptor(mappedInterceptor.getInterceptor());
         }
      }
      else {
           //否自直接加入到处理器链
         chain.addInterceptor(interceptor);
      }
   }
   return chain;
}
```

* 向容器中注册拦截器 xml方式：

```xml
<mvc:interceptors>
   <!--匹配性-->
    <mvc:interceptor>
        <mvc:mapping path="/ag/*" />
        <bean class="com.ryx.credit.cms.intercept.AgentConfirmInterceptor" />
    </mvc:interceptor>
    <!--非匹配型-->
    <bean id="localeChangeInterceptor" class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor"/>
</mvc:interceptors>
```

* WebMvcConfigurer 方式：

 ```java
  public void addInterceptors(InterceptorRegistry registry) {
      //添加注册器 InterceptorRegistration
      registry.addInterceptor(new HandlerInterceptor() {
          @Override
          public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception { return true;}
  
          @Override
          public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {}
  
          @Override
          public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {}
      })    
      .addPathPatterns("/**")
      .excludePathPatterns("/a");
  }
 
 //InterceptorRegistration
 protected Object getInterceptor() {
     //如果未添加匹配路劲也未添加排除匹配就默认为非匹配型拦截器
     if (this.includePatterns.isEmpty() && this.excludePatterns.isEmpty()) {
         return this.interceptor;
     }
     String[] include = StringUtils.toStringArray(this.includePatterns);
     String[] exclude = StringUtils.toStringArray(this.excludePatterns);
     MappedInterceptor mappedInterceptor = new MappedInterceptor(include, exclude, this.interceptor);
     if (this.pathMatcher != null) {
         mappedInterceptor.setPathMatcher(this.pathMatcher);
     }
     return mappedInterceptor;
 }
 ```

# 请求映射到具体的处理程序关系初始化

## 如何注册请求处理器

- 被@Controller或者@RequestMapping 注解标注的容器Bean的内部 被@RequestMapping注解标注方法 并构建成<u>HandlerMethod</u>（class,method）
- 以"/"开头的或者别名是以"/"开头的容器内的任意Bean (<u>Controller HandlerFunction Servlet HttpRequestHandler</u>)

## AbstractHandlerMapping 有两个子类AbstractUrlHandlerMapping 和 AbstractHandlerMethodMapping

- 其中AbstractUrlHandlerMapping  维护了 **Map<String, Object> handlerMap** 映射关系并提供注册映射关系的方法，具体的注册逻辑交给了子类来处理，

  子类的注册时机是在AbstractHandlerMapping 中的 已实现了的initApplicationContext 中，子类中调用父类的initApplicationContext 后初始化映射

- 其中AbstractHandlerMethodMapping 维护了 **MappingRegistry mappingRegistry** 映射关系并提供注册映射关系的方法，并实现了注册逻辑的模板反方，并预留了是否需要注册的接口留给了子类去实现

AbstractHandlerMethodMapping 注册的模板方法 是在InitializingBean   afterPropertiesSet() 接口方法中实现的

```java

public void afterPropertiesSet() {
	initHandlerMethods();
}
protected void initHandlerMethods() {
	//获取容器中候选的beanName
    for (String beanName : getCandidateBeanNames()) {
    
		//排除scopedTarget.
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {     
			//注册        
        	processCandidateBean(beanName);
        }
    }
    
    handlerMethodsInitialized(getHandlerMethods());
}

protected void processCandidateBean(String beanName) {

 	Class<?> beanType = beanType = obtainApplicationContext().getType(beanName);  
    
	//isHandler 交给了子类去实现
    if (beanType != null && isHandler(beanType)) {
    
	//探测bean内部那些方法是handler,
     detectHandlerMethods(beanName);
     
    }
}
```





## AbstractHandlerMethodMapping子类RequestMappingHandlerMapping

- RequestMappingHandlerMapping映射那些被 Controller 或者RequestMapping 注解的类 ,并交由AbstractHandlerMethodMapping  来解析这些类下的方法为handler，并注册到 mappingRegistry

```java
protected boolean isHandler(Class<?> beanType) {
   return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
         AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}
```

- 并且解析Method  RequestMappingInfo



## AbstractUrlHandlerMapping  

AbstractUrlHandlerMapping   缓存了映射关系并提供了请求与处理器注册的方法registerHandler ，并且注册处理器过程中设置ROOThandler 和默认的 defaultHandler

```java
             if (urlPath.equals("/")) {
				setRootHandler(resolvedHandler);
			}
			else if (urlPath.equals("/*")) {
				setDefaultHandler(resolvedHandler);
			}
			else {
				this.handlerMap.put(urlPath, resolvedHandler);
			}
```



### AbstractUrlHandlerMapping  子类AbstractDetectingUrlHandlerMapping

AbstractDetectingUrlHandlerMapping 在探测处理器过程中，定义了探测的源也就是容器内的bean，并预留了抽象方法 String[] determineUrlsForHandler(String beanName) 来确定bean名称是否可作为handler   ,



```java
public void initApplicationContext() throws ApplicationContextException {
		super.initApplicationContext();
		detectHandlers();
	}
protected void detectHandlers() throws BeansException {
    ApplicationContext applicationContext = obtainApplicationContext();
    String[] beanNames = (this.detectHandlersInAncestorContexts ?
    BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext, Object.class) :
    applicationContext.getBeanNamesForType(Object.class));

    for (String beanName : beanNames) {
    
        /*********determineUrlsForHandler 方法为抽象方法，子类觉得bean的是否可以作为Handler来注册******/
        String[] urls = determineUrlsForHandler(beanName);
        
        if (!ObjectUtils.isEmpty(urls)) {
        registerHandler(urls, beanName);
        }
    }
}
```

其中有一个默认的实现类 BeanNameUrlHandlerMapping，BeanNameUrlHandlerMapping 的 方法定义了那些bean的名字可以被注册映射关系，那些以 “/”  开始 或者别名以 ‘/“ 开始的都可以注册为一个请求和处理的映射

```java
protected String[] determineUrlsForHandler(String beanName) {
		List<String> urls = new ArrayList<>();
		if (beanName.startsWith("/")) {
			urls.add(beanName);
		}
		String[] aliases = obtainApplicationContext().getAliases(beanName);
		for (String alias : aliases) {
			if (alias.startsWith("/")) {
				urls.add(alias);
			}
		}
		return StringUtils.toStringArray(urls);
	}
```

### AbstractUrlHandlerMapping  子类SimpleUrlHandlerMapping

SimpleUrlHandlerMapping 维护这一个url与处理器的映射关系  Map<String, Object> urlMap

我们常用的ResourceHandler 就采用了这个HandlerMapping，其中handler 采用的是HttpRequestHandler   , ResourceHandlerRegistry内部分代码

```java
protected AbstractHandlerMapping getHandlerMapping() {
		if (this.registrations.isEmpty()) {
			return null;
		}
		Map<String, HttpRequestHandler> urlMap = new LinkedHashMap<>();
		for (ResourceHandlerRegistration registration : this.registrations) {
			for (String pathPattern : registration.getPathPatterns()) {
				ResourceHttpRequestHandler handler = registration.getRequestHandler();
				if (this.pathHelper != null) {
					handler.setUrlPathHelper(this.pathHelper);
				}
				if (this.contentNegotiationManager != null) {
					handler.setContentNegotiationManager(this.contentNegotiationManager);
				}
				handler.setServletContext(this.servletContext);
				handler.setApplicationContext(this.applicationContext);
				try {
					handler.afterPropertiesSet();
				}
				catch (Throwable ex) {
					throw new BeanInitializationException("Failed to init ResourceHttpRequestHandler", ex);
				}
				
				//将资源映射到处理器
				urlMap.put(pathPattern, handler);
			}
		}
		//构建资源映射处理器
		return new SimpleUrlHandlerMapping(urlMap, this.order);
	}
```

### AbstractUrlHandlerMapping  子类WelcomePageHandlerMapping

WelcomePageHandlerMapping 未做任何映射，只是设置了AbstractUrlHandlerMapping   "/"   RootHandler 

### Handler处理程序

1. Controller 

   执行适配程序 SimpleControllerHandlerAdapter

   ```java
   @FunctionalInterface
   public interface Controller {
      @Nullable
      ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception;
   
   }
   
   public class SimpleControllerHandlerAdapter implements HandlerAdapter {
   
   	@Override
   	public boolean supports(Object handler) {
   		return (handler instanceof Controller);
   	}
   
   	@Override
   	@Nullable
   	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
   			throws Exception {
            //返回试图
   		return ((Controller) handler).handleRequest(request, response);
   	}
   
   	...
   
   }
   ```

2. HandlerFunction (Flux)

   执行适配程序 HandlerFunctionAdapter

   ```java
   public interface HandlerFunction<T extends ServerResponse> {
      T handle(ServerRequest request) throws Exception;
   }
   ```

3. Servlet 

   执行适配程序SimpleServletHandlerAdapter

   ```java
   public class SimpleServletHandlerAdapter implements HandlerAdapter {
   
      @Override
      public boolean supports(Object handler) {
         return (handler instanceof Servlet);
      }
   
      @Override
      @Nullable
      public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
          //未返回试图
         ((Servlet) handler).service(request, response);
         return null;
      }
   
    ...
   
   }
   ```

4. HttpRequestHandler

   执行适配程序 HttpRequestHandlerAdapter

   ```java
   public class HttpRequestHandlerAdapter implements HandlerAdapter {
   
      @Override
      public boolean supports(Object handler) {
         return (handler instanceof HttpRequestHandler);
      }
   
      @Override
      @Nullable
      public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
         //未返回试图
         ((HttpRequestHandler) handler).handleRequest(request, response);
         return null;
      }
   
      ...
   
   }
   ```

5. HandlerMethod

   



# HandlerMapping接口方法getHandler

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   //子类实现方法-通过缓存的映射关系，获取具体的处理器 Handler
   Object handler = getHandlerInternal(request);
   //获取默认的处理器
   if (handler == null) {
      handler = getDefaultHandler();
   }
   if (handler == null) {
      return null;
   }
   //如果是String类型从容器中获取bean
   if (handler instanceof String) {
      String handlerName = (String) handler;
      handler = obtainApplicationContext().getBean(handlerName);
   }
   
   /******构建HandlerExecutionChain,放入处理器，并根据请求url，匹配填充MappedInterceptor 列表******/
   HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);

   if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
      CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
      CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
      config = (config != null ? config.combine(handlerConfig) : handlerConfig);
      executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
   }
/******返回处理器链，内部包含了Handelr 和 匹配的拦截器列表******/
   return executionChain;
  }
```

```java
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
   HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
         (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

   String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
   for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
      if (interceptor instanceof MappedInterceptor) {
         MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
         /******填充匹配的拦截器列表******/
         if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
            chain.addInterceptor(mappedInterceptor.getInterceptor());
         }
      }
      else {
         chain.addInterceptor(interceptor);
      }
   }
   return chain;
}
```
