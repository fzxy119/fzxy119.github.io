---
title: SpringWebMvc-HandlerAdapter
date: 2021-09-08 16:42:25
tags:
- springmvc
- spring
categories:
- 技术
---

HandlerAdapter负责调用映射到请求的处理程序，HandlerAdapter主要目的是隐藏调用细节，简化DispatcherServlet，让DispatcherServlet专注于流程控制.

DispatcherServlet在处理请求的时候从维护的handler适配器列表中获取并选择合适的适配器执行器去执行，那HandlerAdapter是如何注册到DispatcherServlet中的呢 

HandlerMapping中映射着具体的处理器handler类型繁多（如HandlerMethod Controller HandlerFunction Servlet HttpRequestHandler等），功能各异，那这些处理程序的执行各有不同，执行方式也各有不同，DispatcherServlet是如何根据handler找到指定的HandlerAdapter并完成执行请求的呢

HandlerAdapter要处理的具体执行器有多重类型（如HandlerMethod Controller HandlerFunction Servlet HttpRequestHandler等）那么他们是通过哪些HandlerAdapter具体实现来处理的呢 

```java
public interface HandlerAdapter {
    /******支持处理程序检测******/
	boolean supports(Object handler);
    /******处理请求调用******/
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
    /******返回最后修改时间******/
	long getLastModified(HttpServletRequest request, Object handler);
}
```

<!-- more -->

# HandlerAdapter探测

DispatcherServlet 继承自FrameworkServlet ，并且实现了onRefresh 监听WebApplicationContext刷新方法，完成MVC各个处理节点处理策略的初始化，其中HandlerAdapter就是其中之一，

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



完成探测并维护HandlerAdapter到DispatcherServlet的handlerAdapters中：

1. 根据detectAllHandlerAdapters（是否探测所有的HandlerAdapter）属性 true 完成 从bean容器中获取所有的HandlerAdapter
2. 根据detectAllHandlerAdapters（是否探测所有的HandlerAdapter）属性 false 完成 从bean容器中获取beanName 为handlerAdapter类型为HandlerAdapter
3. 如果都没有，就从默认DispatcherServlet.properties 配置中获取，spring-webmvc 包下配置了默认的策略规则HandlerAdapter的默认策略规则为

```properties
org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
   org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
   org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
   org.springframework.web.servlet.function.support.HandlerFunctionAdapter
```

```java
private void initHandlerAdapters(ApplicationContext context) {
		this.handlerAdapters = null;
		if (this.detectAllHandlerAdapters) {
			Map<String, HandlerAdapter> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerAdapters = new ArrayList<>(matchingBeans.values());
				AnnotationAwareOrderComparator.sort(this.handlerAdapters);
			}
		}
		else {
				HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
				this.handlerAdapters = Collections.singletonList(ha);
		}

		if (this.handlerAdapters == null) {
			this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
		}
	}
```

# DispatcherServlet根据handler匹配到HandlerAdapter并完成请求执行

DispatcherServlet根据请求完成HandlerExecutionChain构建之后，DispatcherServlet已经知道了需要哪个执行器去执行并且需要执行哪些拦截器，剩下的需要Adapter接管具体的Handler执行操作。

程序从Mapping阶段 （根据请求映射到确定handler和HandlerInterceptor）进入Adapter阶段(根据具体的handler确定具体的适配执行器),在适配阶段，DispatcherServlet根据具体的Handler由HandlerAdapter 自行适配handler。并告知DispatcherServlet，是否能处理，如果HandlerAdapter 告知DispatcherServlet支持此当前请求的handler，那么DispatcherServlet将采用，并在后续的方法中将执行操作委派给HandlerAdapter 

```java
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```

```java
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter adapter : this.handlerAdapters) {
				if (adapter.supports(handler)) {
					return adapter;
				}
			}
		}
	}
```

```java
/******根据具体的handler确定具体的适配执行器******/
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
/******拦截器PreHandle ******/
if (!handlerExecutionChain.applyPreHandle(processedRequest, response)) {return;}
/******将执行操作委派给HandlerAdapter ******/
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
/******拦截器PostHandle ******/
handlerExecutionChain.applyPostHandle(processedRequest, response, mv);
```

# 处理器Handler类型及与之对应的HandlerAdapter

{% asset_img HandlerAdapterClassH.png 'HandlerAdapter结构图' %}

## Handler类型

1. Handler类型org.springframework.web.method.HandlerMethod对应AbstractHandlerMethodAdapter (RequestMappingHandlerAdapter)

   ```java
   public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {
   	public final boolean supports(Object handler) {
   		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
   	}
   }
   ```
   
2. Handler类型 org.springframework.web.servlet.mvc.Controller 对应 SimpleControllerHandlerAdapter

   ```java
   public class SimpleControllerHandlerAdapter implements HandlerAdapter {
   	public boolean supports(Object handler) {
   		return (handler instanceof Controller);
   	}
       public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)throws Exception {
   		return ((Controller) handler).handleRequest(request, response);
   	}
   }
   ```
   
3. Handler类型 org.springframework.web.servlet.function.HandlerFunction  对应 HandlerFunctionAdapter

   ```java
   public class HandlerFunctionAdapter implements HandlerAdapter, Ordered {
        public boolean supports(Object handler) {
   		return handler instanceof HandlerFunction;
   	}
       public ModelAndView handle(HttpServletRequest servletRequest,HttpServletResponse servletResponse,Object handler) throws
           Exception {
   		HandlerFunction<?> handlerFunction = (HandlerFunction<?>) handler;
   		ServerRequest serverRequest = getServerRequest(servletRequest);
   		ServerResponse serverResponse = handlerFunction.handle(serverRequest);
   		return serverResponse.writeTo(servletRequest, servletResponse,new ServerRequestContext(serverRequest));
   	}
   }
   	
   ```

   子类

   - ResourceHandlerFunction，资源文件处理器

     

4. Handler类型 javax.servlet.Servlet.Servlet  对应 SimpleServletHandlerAdapter

   ```java
   public class SimpleServletHandlerAdapter implements HandlerAdapter {
       public boolean supports(Object handler) {
   		return (handler instanceof Servlet);
   	}  
       public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)throws Exception {
   		((Servlet) handler).service(request, response);
   		return null;
   	}
   }
   ```

   

5. Handler类型 org.springframework.web.servlet.mvc.HttpRequestHandler 对应 HttpRequestHandlerAdapter

   ```java
   public class HttpRequestHandlerAdapter implements HandlerAdapter {
       public boolean supports(Object handler) {
   		return (handler instanceof HttpRequestHandler);
   	}
   	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)throws Exception {
   		((HttpRequestHandler) handler).handleRequest(request, response);
   		return null;
   	}
   }
   ```

     子类

   - ResourceHttpRequestHandler

   - DefaultServletHttpRequestHandler 

   - HttpInvokerServiceExporter 

   - HessianServiceExporter

    

# AbstractHandlerMethodAdapter如何执行

```java
protected ModelAndView handleInternal(HttpServletRequest request,HttpServletResponse response, HandlerMethod handlerMethod) throws
Exception {
		ModelAndView mav;
		checkRequest(request);
		mav = invokeHandlerMethod(request, response, handlerMethod);
		return mav;
}

protected ModelAndView invokeHandlerMethod(HttpServletRequest request,HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
```



## HandlerMethodArgumentResolver

对HandlerMethod对应方法中的参数进行逐个的解析，并赋值

## HandlerMethodReturnValueHandler

对返回的结果进行处理





