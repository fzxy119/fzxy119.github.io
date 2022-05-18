---
title: 客户端RestTemplate
date: 2022-05-15 18:47:55
tags:
- SpringCloud负载均衡
categories:
- 技术
---
# Spring Cloud 负载均衡框架描述

服务的治理：

服务治理的参与方有 服务提供方，服务消费方，注册中心，服务提供方将自己作为服务提供者注册到注册中心，服务消费方从注册中心获取订阅的服务列表，注册中心完成服务的健康检查
服务停止注册中心就停止租约，服务启动重新在注册中心续租。

客服端服务端调用：

客户端服务调用是在客户端持有服务列表并与注册中心保持通信，并获取注册中心的服务对应主机列表，***然后通过请求的过程中完成客户单请求的信息重写***，完成负载均衡功能。

# 客户端调用服务端模板工具 spring-web包 RestTemplate

RestTemplate 是同步调用HTTP请求的模板方法工具类，是通过底层HTTP客户端库实现，他定义了我们请求的通用过程，是一个模板工具。支持POST GET PUT等http请求方法。并提供了不同的结构类型。
如 getForEntity 返回http请求结果对象，其中包含了响应的头，响应状态等请求相关数据 getForObject 返回指定的数据类型，既将结果转化为给定的对象。
其底层都调用了模板方法 doExecute

{% asset_img RestTemplate.png 类图结构 %}

1. HttpAccessor http访问器 是RestTemplate 类机构中最顶层的类，内部成员及方法如下，其包含了基本的客户端请求工厂 并包含通用的请求初始化器，对所有请求有效, 其包含了创建请求的能力，并使用请求初始器对请求进行初始化。

```java
public abstract class HttpAccessor {
  
// 创建Request工厂
	private ClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
//对创建的请求进行初始化
	private final List<ClientHttpRequestInitializer> clientHttpRequestInitializers = new ArrayList<>();

	 //...

//通过 请求url method 创建请求，内部受保护的方法，底层使用 ClientHttpRequestFactory 完成request的创建
	protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
        //获取工厂并完成请求创建
		ClientHttpRequest request = getRequestFactory().createRequest(url, method);
		initialize(request);
		if (logger.isDebugEnabled()) {
			logger.debug("HTTP " + method.name() + " " + url);
		}
		return request;
	}

	private void initialize(ClientHttpRequest request) {
		this.clientHttpRequestInitializers.forEach(initializer -> initializer.initialize(request));
	}

}
```

2. InterceptingHttpAccessor 继承http访问器HttpAccessor  并做了扩展，持有拦截器列表，并重写了 父类的getRequestFactory 方法，此类返回的是InterceptingClientHttpRequestFactory 工厂类，并内部持有父类的 SimpleClientHttpRequestFactory

```java
public abstract class InterceptingHttpAccessor extends HttpAccessor {
//拦截器列表
	private final List<ClientHttpRequestInterceptor> interceptors = new ArrayList<>();
//拦截器工厂私有成员
	@Nullable
	private volatile ClientHttpRequestFactory interceptingRequestFactory;

	@Override
	public ClientHttpRequestFactory getRequestFactory() {
       //获取拦截器
		List<ClientHttpRequestInterceptor> interceptors = getInterceptors();
		if (!CollectionUtils.isEmpty(interceptors)) {
			ClientHttpRequestFactory factory = this.interceptingRequestFactory;
			if (factory == null) {
                //构建请求工厂，内部持有拦截器列表
				factory = new InterceptingClientHttpRequestFactory(super.getRequestFactory(), interceptors);
				this.interceptingRequestFactory = factory;
			}
			return factory;
		}
		else {
			return super.getRequestFactory();
		}
	}

}
```

3. RestTemplate 继承http访问器InterceptingHttpAccessor ，RestTemplate默认具有拦截器能力，创建请求的能力***重写了父类的能力***，初始化请求的能力，
   自带 消息转换 messageConverters 异常处理 errorHandler 结果提取能力 headersExtractor。

```java
public class RestTemplate extends InterceptingHttpAccessor implements RestOperations{
//消息转换器列表
    private final List<HttpMessageConverter<?>> messageConverters = new ArrayList<>();
//异常处理器
	private ResponseErrorHandler errorHandler = new DefaultResponseErrorHandler();
//url处理模板
	private UriTemplateHandler uriTemplateHandler;
//结果提取器
	private final ResponseExtractor<HttpHeaders> headersExtractor = new HeadersExtractor();

//模板方法
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,@Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {
		ClientHttpResponse response = null;
//创建请求命令  通过分析 createRequest 这里拿到的是 InterceptingClientHttpRequest
			ClientHttpRequest request = createRequest(url, method);
			if (requestCallback != null) {
				requestCallback.doWithRequest(request);
			}
//请求执行 InterceptingClientHttpRequest.execute()
			response = request.execute();
//异常处理
			handleResponse(url, method, response);
//结果提取
			return (responseExtractor != null ? responseExtractor.extractData(response) : null);

		//...
	}
    //创建请求命令
    protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
//调用 父类 InterceptingHttpAccessor 方法 得到 InterceptingClientHttpRequestFactory(内部持有SimpleClientHttpRequestFactory) 创建  ClientHttpRequest(InterceptingClientHttpRequest)
		ClientHttpRequest request = getRequestFactory().createRequest(url, method);
//使用HttpAccessor 方法完成初始化
		initialize(request);
		if (logger.isDebugEnabled()) {
			logger.debug("HTTP " + method.name() + " " + url);
		}
		return request;
	}
}
```

RestTemplate类及父类都是通过ClientHttpRequestFactory工厂来创建请求，并在不同的父类中，工厂类有所不同，InterceptingHttpAccessor 中有ClientHttpRequestFactory (InterceptingClientHttpRequestFactory) 创建 InterceptingClientHttpRequest 有了拦截器能力，HttpAccessor中有ClientHttpRequestFactory （SimpleClientHttpRequestFactory)有通过HttpURLConnection发送http请求发送能力

> 那么 ClientHttpRequestFactory 如何创建 request 并且ClientHttpRequest 是如何在执行的时候，执行拦截器，并完成最后的http请求的呢？

{% asset_img ClientHttpRequestFactory.png 类图结构 %}

{% asset_img HttpRequest.png 类图结构 %}

# ClientHttpRequestFactory 通过URI和HttpMethod创建请求

```java
public interface ClientHttpRequestFactory {
	ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException;
}

/**
 *包装类， 内部持有实际工厂
 **/
public abstract class AbstractClientHttpRequestFactoryWrapper implements ClientHttpRequestFactory {
	private final ClientHttpRequestFactory requestFactory;
	@Override
	public final ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod) throws IOException {
//调用内部方法
		return createRequest(uri, httpMethod, this.requestFactory);
	}
//由子类创建请求，并将包装的工厂，传递给子类
	protected abstract ClientHttpRequest createRequest( URI uri, HttpMethod httpMethod, ClientHttpRequestFactory requestFactory) throws IOException;
}

/**
 * 带有拦截器的工厂类
 **/
public class InterceptingClientHttpRequestFactory extends AbstractClientHttpRequestFactoryWrapper {
// 拦截器列表
	private final List<ClientHttpRequestInterceptor> interceptors;
//持有包装的工厂类 和 拦截器构造方法
	public InterceptingClientHttpRequestFactory(ClientHttpRequestFactory requestFactory,
			@Nullable List<ClientHttpRequestInterceptor> interceptors) {
		super(requestFactory);
		this.interceptors = (interceptors != null ? interceptors : Collections.emptyList());
	}

//这里实现了 父类 AbstractClientHttpRequestFactoryWrapper 的 createRequest 参数 requestFactory 
//正式父类持有的包装类 也就是InterceptingClientHttpRequestFactory 构造函数传递进去的被包装的类，
//根据 InterceptingHttpAccessor 分析 此处被包装的类是 HttpAccessor 类中的 SimpleClientHttpRequestFactory
	@Override
	protected ClientHttpRequest createRequest(URI uri, HttpMethod httpMethod, ClientHttpRequestFactory requestFactory) {
		return new InterceptingClientHttpRequest(requestFactory, this.interceptors, uri, httpMethod);
	}

}
```

# ClientHttpRequest 完成拦截器功能及底层的http请求

{% asset_img HTTPmessage.png 类图结构 %}
{% asset_img HttpRequest.png 类图结构 %}

```java
/**
 *通用消息定义
 **/
public interface HttpMessage {
	HttpHeaders getHeaders();
}

/**
 *输出消息
 **/
public interface HttpOutputMessage extends HttpMessage {
	OutputStream getBody() throws IOException;

}
/**
 *输入消息
 **/
public interface HttpInputMessage extends HttpMessage {
	InputStream getBody() throws IOException;
}

/**
 *http请求
 **/
public interface HttpRequest extends HttpMessage {
//请求方法
	@Nullable
	default HttpMethod getMethod() {
		return HttpMethod.resolve(getMethodValue());
	}
//请求方法
	String getMethodValue();
//请求URI
	URI getURI();
}

/**
 * 抽象的客户端http请求 继承了 HttpRequest HttpOutputMessage 命令模式可被执行（包含http命令的内容 body 包含http headers 包含 请求方法 请求URI)
 **/
public interface ClientHttpRequest extends HttpRequest, HttpOutputMessage {
	ClientHttpResponse execute() throws IOException;
}
/**
 * 抽象的客户端http请求 默认的抽象ClientHttpRequest实现类，持有HttpHeaders成员变量
 * 实现 execute 方法，生成 executeInternal抽象方法    扩展execute 新增Httpheaders参数，提供子类执行http请求必要数据
 * 实现 getBody 方法， 生成  getBodyInternal抽象方法  扩展getBody 新增Httpheaders参数，提供子类执行getBody请求必要数据
 **/
public abstract class AbstractClientHttpRequest implements ClientHttpRequest {
	private final HttpHeaders headers = new HttpHeaders();
	private boolean executed = false;
	@Override
	public final HttpHeaders getHeaders() {
		return (this.executed ? HttpHeaders.readOnlyHttpHeaders(this.headers) : this.headers);
	}
	@Override
	public final OutputStream getBody() throws IOException {
		assertNotExecuted();
		return getBodyInternal(this.headers);
	}
	@Override
	public final ClientHttpResponse execute() throws IOException {
		assertNotExecuted();
		ClientHttpResponse result = executeInternal(this.headers);
		this.executed = true;
		return result;
	}
	protected abstract OutputStream getBodyInternal(HttpHeaders headers) throws IOException;
	protected abstract ClientHttpResponse executeInternal(HttpHeaders headers) throws IOException;

}
/**
 * 抽象的客户端http请求 实现了抽象类AbstractClientHttpRequest，持有消息体 
 * 实现 executeInternal 并扩展  executeInternal 除了HttpHeaders 新增bufferedOutput参数，提供子类执行http请求必要数据
 **/
abstract class AbstractBufferingClientHttpRequest extends AbstractClientHttpRequest {
	private ByteArrayOutputStream bufferedOutput = new ByteArrayOutputStream(1024);
	@Override
	protected OutputStream getBodyInternal(HttpHeaders headers) throws IOException {
		return this.bufferedOutput;
	}
	@Override
	protected ClientHttpResponse executeInternal(HttpHeaders headers) throws IOException {
		byte[] bytes = this.bufferedOutput.toByteArray();
//添加内容长度
		if (headers.getContentLength() < 0) {
			headers.setContentLength(bytes.length);
		}
		ClientHttpResponse result = executeInternal(headers, bytes);
		this.bufferedOutput = new ByteArrayOutputStream(0);
		return result;
	}
//子类实现 此类提供请求头和请求体
	protected abstract ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput)throws IOException;
}


/**
 * 客户端http请求 实现了拦截器链功能的请求 实现了抽象类AbstractBufferingClientHttpRequest，持有消息体，持有消息头，并且完成了 executeInternal实现，是一个具体可被执行的http请求
 *  1：执行委派给了拦截器处理链
 *  2：拦截器链顺序执行 拦截器列表，拦截器完成对请求对象的相关处理 如负载均衡
 *  3：拦截器链执行完毕后最终执行 SimpleClientHttpRequestFactory 创建请求，完成真正的请求执行
 **/
class InterceptingClientHttpRequest extends AbstractBufferingClientHttpRequest {
    //有包装工厂 InterceptingClientHttpRequestFactory 提供的  SimpleClientHttpRequestFactory
	private final ClientHttpRequestFactory requestFactory;
    //有包装工厂 InterceptingClientHttpRequestFactory 提供的  拦截器处理链
	private final List<ClientHttpRequestInterceptor> interceptors;
    //请求URI及请求方法
	private HttpMethod method;
	private URI uri;
    //...

   //具体执行委派给了处理链
	@Override
	protected final ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
		InterceptingRequestExecution requestExecution = new InterceptingRequestExecution();
		//拦截器链执行
		return requestExecution.execute(this, bufferedOutput);
	}

    //拦截器链执行器
 	private class InterceptingRequestExecution implements ClientHttpRequestExecution {
		private final Iterator<ClientHttpRequestInterceptor> iterator;
		public InterceptingRequestExecution() {
            //处理链持有 InterceptingClientHttpRequest 中的 拦截器迭代器
			this.iterator = interceptors.iterator();
		}

		@Override
		public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
		    //执行拦截器，拦截器中持有处理链，并在拦截器链中继续执行拦截器链
			if (this.iterator.hasNext()) {
				ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
				return nextInterceptor.intercept(request, body, this);
			}
            //处理链执行完毕执行真正的请求
			else {
				HttpMethod method = request.getMethod();
                //将经过一系列处理的请求，最终数据委派SimpleClientHttpRequestFactory 创建请求来执行
				ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
                //将请求数据设置到委派的请求中
				request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
				if (body.length > 0) {
					if (delegate instanceof StreamingHttpOutputMessage) {
						StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
						streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
					}
					else {
						StreamUtils.copy(body, delegate.getBody());
					}
				}
				return delegate.execute();
			}
		}
	}
}
```

# 总 结

1. RestTemplate 通过模板方法来控制整个执行过程
2. RestTemplate 包含了 父类 HttpAccessor (创建基础HTTP请求能力-SimpleClientHttpRequestFactory) 和 父类InterceptingHttpAccessor（创建带有拦截器功能的HTTP请求的能力-InterceptingClientHttpRequestFactory)
3. RestTemplate createRequest 使用 InterceptingClientHttpRequestFactory 创建了 InterceptingClientHttpRequest。InterceptingClientHttpRequestFactory 是一个装饰类 里边装饰了 SimpleClientHttpRequestFactory，并且持有拦截器列表，并创建InterceptingClientHttpRequest请求，InterceptingClientHttpRequest中持有了SimpleClientHttpRequestFactory，这使得InterceptingClientHttpRequest完成处理拦截器链的请求执行完成后可执行基础的HTTP请求。
4. InterceptingClientHttpRequest 内部类 InterceptingRequestExecution 使用拦截器列表迭代器，顺序执行拦截器列表，并在最后使用SimpleClientHttpRequestFactory创建请求完成基础HTTP请求

### spring-cloud-commons 包负载均衡自动化配置

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.client.loadbalancer.AsyncLoadBalancerAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.reactive.LoadBalancerBeanPostProcessorAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.reactive.ReactorLoadBalancerClientAutoConfiguration,\
org.springframework.cloud.client.loadbalancer.reactive.ReactiveLoadBalancerAutoConfiguration,\
org.springframework.cloud.client.serviceregistry.ServiceRegistryAutoConfiguration,\
org.springframework.cloud.commons.httpclient.HttpClientConfiguration,\
org.springframework.cloud.commons.util.UtilAutoConfiguration,\
org.springframework.cloud.configuration.CompatibilityVerifierAutoConfiguration,\
org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration
```

### spring-cloud-commons 包

SmartInitializingSingleton
LoadBalancerRequestFactory
LoadBalancerClient 接口
LoadBalancerInterceptor ClientHttpRequestInterceptor
RestTemplateCustomizer

### spring-web 包

RestTemplate
ClientHttpRequestFactory
ClientHttpRequestInterceptor
HttpMessage -> HttpOutputMessage ->  ClientHttpRequest(HttpRequest)
HttpMessage -> HttpInputMessage  ->  ClientHttpResponse
HttpRequest -> HttpRequestWrapper
UriTemplateHandler
RequestCallback
HttpMessageConverter
ResponseErrorHandler
ResponseExtractor

### spring-cloud-netflix-ribbon

RibbonAutoConfiguration
RibbonLoadBalancerClient
RestTemplateCustomizer
RibbonClientHttpRequestFactory

### SpringClientFactory 服务子容器内默认配置

```java
public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification> {

	static final String NAMESPACE = "ribbon";

	public SpringClientFactory() {
        //子容器相关类默认配置RibbonClientConfiguration 默认控制及命名空间
		super(RibbonClientConfiguration.class, NAMESPACE, "ribbon.client.name");
	}
}
```

#### 属性工厂

支持配置文件控制ribbon 相关元素  ILoadBalancer IPing IRule ServerList ServerListFilter

* 从环境变量中获取相关服务  如：orderapplication.ribbon.NFLoadBalancerClassName
* 从环境变量中获取相关服务  如：orderapplication.ribbon.NFLoadBalancerPingClassName
* 从环境变量中获取相关服务  如：orderapplication.ribbon.NFLoadBalancerRuleClassName
* 从环境变量中获取相关服务  如：orderapplication.ribbon.NIWSServerListClassName
* 从环境变量中获取相关服务  如：orderapplication.ribbon.NIWSServerListFilterClassName

```java
public class PropertiesFactory {

	@Autowired
	private Environment environment;

	private Map<Class, String> classToProperty = new HashMap<>();

	public PropertiesFactory() {
		classToProperty.put(ILoadBalancer.class, "NFLoadBalancerClassName");
		classToProperty.put(IPing.class, "NFLoadBalancerPingClassName");
		classToProperty.put(IRule.class, "NFLoadBalancerRuleClassName");
		classToProperty.put(ServerList.class, "NIWSServerListClassName");
		classToProperty.put(ServerListFilter.class, "NIWSServerListFilterClassName");
	}

	public boolean isSet(Class clazz, String name) {
		return StringUtils.hasText(getClassName(clazz, name));
	}

	public String getClassName(Class clazz, String name) {
		if (this.classToProperty.containsKey(clazz)) {
			String classNameProperty = this.classToProperty.get(clazz);
            //从环境变量中获取相关服务  如：orderapplication.ribbon.NFLoadBalancerClassName
			String className = environment.getProperty(name + "." + NAMESPACE + "." + classNameProperty);
			return className;
		}
		return null;
	}

	@SuppressWarnings("unchecked")
	public <C> C get(Class<C> clazz, IClientConfig config, String name) {
		String className = getClassName(clazz, name);
		if (StringUtils.hasText(className)) {
			try {
				Class<?> toInstantiate = Class.forName(className);
				return (C) SpringClientFactory.instantiateWithConfig(toInstantiate,
						config);
			}
			catch (ClassNotFoundException e) {
				throw new IllegalArgumentException("Unknown class to load " + className
						+ " for class " + clazz + " named " + name);
			}
		}
		return null;
	}

}
```

#### 子容器相关类默认配置（RibbonClientConfiguration)

默认每个子容器都配置了Ribbon需要的组件

* IClientConfig  默认配置 DefaultClientConfigImpl
* IRule  默认配置 ZoneAvoidanceRule
* IPing  默认配置 DummyPing
* ServerList `<Server>` 默认配置ConfigurationBasedServerList
* ServerListUpdater 默认配置 PollingServerListUpdater
* ILoadBalancer 默认配置 ZoneAwareLoadBalancer
* ServerListFilter `<Server>` 默认配置 ZonePreferenceServerListFilter
* RibbonLoadBalancerContext 默认配置 RibbonLoadBalancerContext
* RetryHandler 默认配置 DefaultLoadBalancerRetryHandler
* ServerIntrospector 默认配置 DefaultServerIntrospector

```java
public class RibbonClientConfiguration {

	/**
	 * Ribbon client default connect timeout.
	 */
	public static final int DEFAULT_CONNECT_TIMEOUT = 1000;

	/**
	 * Ribbon client default read timeout.
	 */
	public static final int DEFAULT_READ_TIMEOUT = 1000;

	/**
	 * Ribbon client default Gzip Payload flag.
	 */
	public static final boolean DEFAULT_GZIP_PAYLOAD = true;

	@RibbonClientName
	private String name = "client";

	// TODO: maybe re-instate autowired load balancers: identified by name they could be
	// associated with ribbon clients

	@Autowired
	private PropertiesFactory propertiesFactory;

	@Bean
	@ConditionalOnMissingBean
	public IClientConfig ribbonClientConfig() {
		DefaultClientConfigImpl config = new DefaultClientConfigImpl();
//加载给定服务的属性
		config.loadProperties(this.name);
		config.set(CommonClientConfigKey.ConnectTimeout, DEFAULT_CONNECT_TIMEOUT);
		config.set(CommonClientConfigKey.ReadTimeout, DEFAULT_READ_TIMEOUT);
		config.set(CommonClientConfigKey.GZipPayload, DEFAULT_GZIP_PAYLOAD);
		return config;
	}

	@Bean
	@ConditionalOnMissingBean
	public IRule ribbonRule(IClientConfig config) {
//从配置文件中加载IRule
		if (this.propertiesFactory.isSet(IRule.class, name)) {
			return this.propertiesFactory.get(IRule.class, config, name);
		}
		ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
		rule.initWithNiwsConfig(config);
		return rule;
	}

	@Bean
	@ConditionalOnMissingBean
	public IPing ribbonPing(IClientConfig config) {
//从配置文件中加载IRule
		if (this.propertiesFactory.isSet(IPing.class, name)) {
			return this.propertiesFactory.get(IPing.class, config, name);
		}
		return new DummyPing();
	}

	@Bean
	@ConditionalOnMissingBean
	@SuppressWarnings("unchecked")
	public ServerList<Server> ribbonServerList(IClientConfig config) {
		if (this.propertiesFactory.isSet(ServerList.class, name)) {
			return this.propertiesFactory.get(ServerList.class, config, name);
		}
		ConfigurationBasedServerList serverList = new ConfigurationBasedServerList();
		serverList.initWithNiwsConfig(config);
		return serverList;
	}

	@Bean
	@ConditionalOnMissingBean
	public ServerListUpdater ribbonServerListUpdater(IClientConfig config) {
		return new PollingServerListUpdater(config);
	}

	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}

	@Bean
	@ConditionalOnMissingBean
	@SuppressWarnings("unchecked")
	public ServerListFilter<Server> ribbonServerListFilter(IClientConfig config) {
		if (this.propertiesFactory.isSet(ServerListFilter.class, name)) {
			return this.propertiesFactory.get(ServerListFilter.class, config, name);
		}
		ZonePreferenceServerListFilter filter = new ZonePreferenceServerListFilter();
		filter.initWithNiwsConfig(config);
		return filter;
	}

	@Bean
	@ConditionalOnMissingBean
	public RibbonLoadBalancerContext ribbonLoadBalancerContext(ILoadBalancer loadBalancer,
			IClientConfig config, RetryHandler retryHandler) {
		return new RibbonLoadBalancerContext(loadBalancer, config, retryHandler);
	}

	@Bean
	@ConditionalOnMissingBean
	public RetryHandler retryHandler(IClientConfig config) {
		return new DefaultLoadBalancerRetryHandler(config);
	}

	@Bean
	@ConditionalOnMissingBean
	public ServerIntrospector serverIntrospector() {
		return new DefaultServerIntrospector();
	}

	@PostConstruct
	public void preprocess() {
		setRibbonProperty(name, DeploymentContextBasedVipAddresses.key(), name);
	}
}
```
