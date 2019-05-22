---
title: springcloud：ribbon配置和zuul重试源码解读
date: 2018-05-29 22:33:18
type: "tags"
tags: [springcloud]
---

> 最近刚刚把springcloud的版本升级到**Finchley.RC1**，就遇到些小麻烦，分享一下解决的过程和收获。 🥕

<!--more-->

springcloud最近的版本更迭很快，为了更好的契合springboot2，我也把版本从**Finchley.M9**升级到**Finchley.RC1**，重新跑了下gateway，发现zuul重试功能莫名的失效了。因为一开始直接拿了网上的配置，也没有多去考究，正好趁着这个机会好好挖掘一下。

> 注：文章有点长，包括**ribbon**配置源码查阅的思路和**zuul**重试源码的解读，可以根据导航栏选择需要的阅读。

### ribbon 配置问题

如果要实现**zuul**重试功能是要先引入：
```
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

然后添加配置，我一开始的配置是这样的：
```
zuul:
  retryable: true
ribbon:
  connectTimeout: 500                 # 请求连接的超时时间
  readTimeout:  1000                  # 请求处理的超时时间
  maxAutoRetries: 2                   # 对当前实例的重试次数
  maxAutoRetriesNextServer: 1         # 切换实例的重试次数
  okToRetryOnAllOperations: false     # 对所有操作请求都进行重试
```

查阅源码后发现正确的写法应该是：
```
zuul:
  retryable: true
ribbon:
  ConnectTimeout: 500                 # 请求连接的超时时间
  ReadTimeout: 1000                   # 请求处理的超时时间
  MaxAutoRetries: 2                   # 对当前实例的重试次数
  MaxAutoRetriesNextServer: 1         # 切换实例的重试次数
  OkToRetryOnAllOperations: false     # 对所有操作请求都进行重试
```

上面的方式是全局的配置，如果需要局部的配置，可以这样写成
```
（serviceId）:
  ribbon:	
    ConnectTimeout: 500                 # 请求连接的超时时间
    ReadTimeout: 1000                   # 请求处理的超时时间
    MaxAutoRetries: 2                   # 对当前实例的重试次数
    MaxAutoRetriesNextServer: 1         # 切换实例的重试次数
    OkToRetryOnAllOperations: false     # 对所有操作请求都进行重试	
```

其实，就是大小写导致参数失效，而且这些在配置文件中是不给提示的，那就是代表没有对应的**@ConfigurationProperties**，那是如何生效呢，这就勾起了我的好奇心。

### ribbon 源码解读

对应这些配置的参数查阅源码，可以发现``ribbon-core``源码里有对应的``CommonClientConfigKey``类

```
public abstract class CommonClientConfigKey<T> implements IClientConfigKey<T> {
    ......

    public static final IClientConfigKey<Integer> MaxAutoRetries = new CommonClientConfigKey<Integer>("MaxAutoRetries"){};
    
    public static final IClientConfigKey<Integer> MaxAutoRetriesNextServer = new CommonClientConfigKey<Integer>("MaxAutoRetriesNextServer"){};

    public static final IClientConfigKey<Boolean> OkToRetryOnAllOperations = new CommonClientConfigKey<Boolean>("OkToRetryOnAllOperations"){};
   	
    ......
}
```

这3个参数在``DefaultLoadBalancerRetryHandler``类中被使用，而这个类在``RibbonClientConfiguration``中完成初始化的，因为**@ConditionalOnMissingBean**则不会被多次加载。

```
@Bean
@ConditionalOnMissingBean
public RetryHandler retryHandler(IClientConfig config) {
	return new DefaultLoadBalancerRetryHandler(config);
}
```

其他参数也是类似，不过被使用在其他类中。它们去获得对应的value都是使用**getProperty**方法，通过调用**getInstance**获得包装过的动态属性类，再拿到对应的value。

```
public class DefaultClientConfigImpl implements IClientConfig {
    
    /**
     * 参数获取对应属性值
     */
    protected Object getProperty(String key) {
        if (enableDynamicProperties) {
            String dynamicValue = null;
            DynamicStringProperty dynamicProperty = dynamicProperties.get(key);
            if (dynamicProperty != null) {
                dynamicValue = dynamicProperty.get();
            }
            if (dynamicValue == null) {
                dynamicValue = DynamicProperty.getInstance(getConfigKey(key)).getString();
                if (dynamicValue == null) {
                    dynamicValue = DynamicProperty.getInstance(getDefaultPropName(key)).getString();
                }
            }
            if (dynamicValue != null) {
                return dynamicValue;
            }
        }
        return properties.get(key);
    }
}

public class DynamicProperty {

    public static DynamicProperty getInstance(String propName) {
        // This is to ensure that a configuration source is registered with
        // DynamicProperty
        if (dynamicPropertySupportImpl == null) {
            DynamicPropertyFactory.getInstance();
        }
        DynamicProperty prop = ALL_PROPS.get(propName);
        if (prop == null) {
            prop = new DynamicProperty(propName);
            DynamicProperty oldProp = ALL_PROPS.putIfAbsent(propName, prop);
            if (oldProp != null) {
                prop = oldProp;
            }
        }
        return prop;
    }
｝
```

到这里其实整个向下的流程我们已经知道了，那剩余的就是看看**ALL_PROPS**是在何时被填充数据的了。这里面牵扯的类比较多，源码就不一一呈现了，捡取几个重要的。

其中在``DefaultClientConfigImpl``有个很重要的方法**loadProperties**，作用是来加载配置。

```
/**
 * 加载给定的客户端属性
 */
@Override
public void loadProperties(String restClientName){
    enableDynamicProperties = true;
    setClientName(restClientName);
    //重要：加载默认值，有手动设置的话，使用设置的值
    loadDefaultValues();
    Configuration props = ConfigurationManager.getConfigInstance().subset(restClientName);
    for (Iterator<String> keys = props.getKeys(); keys.hasNext(); ){
        String key = keys.next();
        String prop = key;
        try {
            if (prop.startsWith(getNameSpace())){
                prop = prop.substring(getNameSpace().length() + 1);
            }
            setPropertyInternal(prop, getStringValue(props, key));
        } catch (Exception ex) {
            throw new RuntimeException(String.format("Property %s is invalid", prop));
        }
    }
}
```

通过**loadDefaultValues**方法调``ConcurrentCompositeConfiguration``的**getProperty**方法，再调``PropertySourcesPropertyResolver``的**getProperty**方法。这样就从**archauus-core**包调到了**spring-core**包了。在**propertySources**中就包含了我们在yml的设置的参数。然后再通过``DefaultClientConfigImpl``类的**setPropertyInternal**方法完成一系列对**dynamicProperties**和**ALL_PROPS**的数据填充。

![propertySources](https://github.com/7le/7le.github.io/raw/master/image/springcloud/propertySources.PNG)

### zuul 源码解读

上面已经分析了配置的参数是如何被使用，但是还是有一个问题，那就是很关键的**loadProperties**的方法是何时被调用。查看源码可以发现该方法是在``RibbonClientConfiguration``类中被调用。查看所在的jar包的``spring.factories``，发现这个类并不会在服务启动时被加载。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration
```

那既然不是在服务启动的时候加载，那猜测是从**Zuul**进行路由转发的时候准备的，继续挖源码。很自然我们的目标就在``ZuulFilter``的子类上了。其中一个很重要的Filter就是``RibbonRoutingFilter``，它主要是完成请求的路由转发，很符合我们的想法，查看它的**run**方法，然后我们跟进到**forward**方法。

```
public class RibbonRoutingFilter extends ZuulFilter {

	@Override
	public Object run() {
		RequestContext context = RequestContext.getCurrentContext();
		this.helper.addIgnoredHeaders();
		try {
			RibbonCommandContext commandContext = buildCommandContext(context);
			ClientHttpResponse response = forward(commandContext);
			setResponse(response);
			return response;
		}
		catch (ZuulException ex) {
			throw new ZuulRuntimeException(ex);
		}
		catch (Exception ex) {
			throw new ZuulRuntimeException(ex);
		}
	}

	protected ClientHttpResponse forward(RibbonCommandContext context) throws Exception {
		Map<String, Object> info = this.helper.debug(context.getMethod(),
				context.getUri(), context.getHeaders(), context.getParams(),
				context.getRequestEntity());

		RibbonCommand command = this.ribbonCommandFactory.create(context);
		try {
			ClientHttpResponse response = command.execute();
			this.helper.appendDebug(info, response.getRawStatusCode(), response.getHeaders());
			return response;
		}
		catch (HystrixRuntimeException ex) {
			return handleException(info, ex);
		}

	}
｝
```

其中有2个重要的方法，分别是**create**和**execute**。很明显配置加载的工作只有可能在**create**方法中，为了验证之前的想法，继续看**create**的源码。这里的**ribbonCommandFactory**是在``RibbonCommandFactoryConfiguration``完成初始化，查看``spring.factories``，该类会在服务启动的时候加载。上面的**@import**很重要，跟进去可以发现，会先去判断是否有``ribbon.restclient.enabled``配置，再判断是否有``okhttp3.OkHttpClient``类，如果都没有设置的话，这个**ribbonCommandFactory**将会是``HttpClientRibbonCommandFactory``。

```
@Configuration
@Import({ RibbonCommandFactoryConfiguration.RestClientRibbonConfiguration.class,
		RibbonCommandFactoryConfiguration.OkHttpRibbonConfiguration.class,
		RibbonCommandFactoryConfiguration.HttpClientRibbonConfiguration.class,
		HttpClientConfiguration.class })
@ConditionalOnBean(ZuulProxyMarkerConfiguration.Marker.class)
public class ZuulProxyAutoConfiguration extends ZuulServerAutoConfiguration {

	// route filters
	@Bean
	public RibbonRoutingFilter ribbonRoutingFilter(ProxyRequestHelper helper,
			RibbonCommandFactory<?> ribbonCommandFactory) {
		RibbonRoutingFilter filter = new RibbonRoutingFilter(helper, ribbonCommandFactory,
				this.requestCustomizers);
		return filter;
	}
}
```

查看``HttpClientRibbonCommandFactory``的**create**方法:

```
public class HttpClientRibbonCommandFactory extends AbstractRibbonCommandFactory {

	@Override
	public HttpClientRibbonCommand create(final RibbonCommandContext context) {
		//提供路由发生故障时的回调（降级方法）
		FallbackProvider zuulFallbackProvider = getFallbackProvider(context.getServiceId());
		final String serviceId = context.getServiceId();
		//创建处理请求转发类
		final RibbonLoadBalancingHttpClient client = this.clientFactory.getClient(
				serviceId, RibbonLoadBalancingHttpClient.class);

		client.setLoadBalancer(this.clientFactory.getLoadBalancer(serviceId));

		return new HttpClientRibbonCommand(serviceId, client, context, zuulProperties, zuulFallbackProvider,
				clientFactory.getClientConfig(serviceId));
	}

}
```

同样的道理，准备的工作一般会在创建的时候完成，往**getClient**方法跟下去，这里涉及的类也不详细列举了，跟下去后会到``NamedContextFactory``的**createContext**方法的``context.refresh();``，刷新了context，会调用到**finishBeanFactoryInitialization**方法中的``beanFactory.preInstantiateSingletons()``，该方法会将这个上下文的bean工厂初始化，并初始化所有剩余的单例bean。这样之前的疑问就解答了，果然是在**zuul**做转发的时候完成了加载。

```
public void preInstantiateSingletons() throws BeansException {
	if (this.logger.isDebugEnabled()) {
		this.logger.debug("Pre-instantiating singletons in " + this);
	}

	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

	// Trigger initialization of all non-lazy singleton beans...
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			if (isFactoryBean(beanName)) {
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					final FactoryBean<?> factory = (FactoryBean<?>) bean;
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
										((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
			}
			else {
				getBean(beanName);
			}
		}
	}
	
	// Trigger post-initialization callback for all applicable beans...
	......
}
```

到此，**forward**中的**create**方法已经看了，问题也解决了。那把剩余的**execute**也一并看了吧。该方法最后执行的会是``AbstractRibbonCommand``的**run**方法，里面重要的就是**executeWithLoadBalancer**方法：

```
public abstract class AbstractLoadBalancerAwareClient<S extends ClientRequest, T extends IResponse> extends LoadBalancerContext implements IClient<S, T>, IClientConfigAware {

	public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
        LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

        // 主要是创建了Observable(RxJava)，并且为这个Observable设置了重试次数，最终回调了RetryableRibbonLoadBalancingHttpClient的execute方法
        // 如果没有设置重试会回调RibbonLoadBalancingHttpClient的execute方法
        try {
            return command.submit(
                new ServerOperation<T>() {
                    @Override
                    public Observable<T> call(Server server) {
                        URI finalUri = reconstructURIWithServer(server, request.getUri());
                        S requestForServer = (S) request.replaceUri(finalUri);
                        try {
                            return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                        } 
                        catch (Exception e) {
                            return Observable.error(e);
                        }
                    }
                })
                .toBlocking()
                .single();
        } catch (Exception e) {
            Throwable t = e.getCause();
            if (t instanceof ClientException) {
                throw (ClientException) t;
            } else {
                throw new ClientException(e);
            }
        }
    }

    protected LoadBalancerCommand<T> buildLoadBalancerCommand(final S request, final IClientConfig config) {
    	//创建一个RetryHandler，决定利用RxJava的Observable是否进行重试的标准
		RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, config);

		//创建一个LoadBalancerCommand，创建Observable以及根据RetryHandler来判断是否进行重试操作。
		LoadBalancerCommand.Builder<T> builder = LoadBalancerCommand.<T>builder()
				.withLoadBalancerContext(this)
				.withRetryHandler(handler)
				.withLoadBalancerURI(request.getUri());
		customizeLoadBalancerCommandBuilder(request, config, builder);
		return builder.build();
	}
}
```

然后就是``LoadBalancerCommand``的**submit**方法：

```
public Observable<T> submit(final ServerOperation<T> operation) {
        
	......

	//相同server的重试次数(去除首次请求)
        final int maxRetrysSame = retryHandler.getMaxRetriesOnSameServer();
        //集群内其他Server的重试个数
        final int maxRetrysNext = retryHandler.getMaxRetriesOnNextServer();

        // 使用负载均衡
        Observable<T> o = 
                (server == null ? selectServer() : Observable.just(server))
                .concatMap(new Func1<Server, Observable<T>>() {
                    
					// 对选择的服务器进行调用
                    @Override
                    public Observable<T> call(Server server) {
                        context.setServer(server);
                        final ServerStats stats = loadBalancerContext.getServerStats(server);
                        
                        Observable<T> o = Observable
                                .just(server)
                                .concatMap(......);

                        if (maxRetrysSame > 0) 
                            o = o.retry(retryPolicy(maxRetrysSame, true));
                        return o;
                    }
                });
         
        if (maxRetrysNext > 0 && server == null) 
            o = o.retry(retryPolicy(maxRetrysNext, false));
        
        return o.onErrorResumeNext(new Func1<Throwable, Observable<T>>() {

            //转发请求失败时，会进入此方法。通过此方法进行判断,是否超过重试次数maxRetrysSame、maxRetrysNext。
            @Override
            public Observable<T> call(Throwable e) {
            	......
            }
        });
    }
```

最后就是``RetryableRibbonLoadBalancingHttpClient``的**execute**方法：

```
@Override
public RibbonApacheHttpResponse execute(final RibbonApacheHttpRequest request, final IClientConfig configOverride) throws Exception {

   	//创建RequestConfig(请求信息)
   	final RequestConfig requestConfig = builder.build();
   	final LoadBalancedRetryPolicy retryPolicy = loadBalancedRetryPolicyFactory.create(this.getClientName(), this);
   
   	//创建RetryCallbck，实现重试时候的回调
   	RetryCallback<RibbonApacheHttpResponse, Exception> retryCallback = context -> {......}
   
   	//创建Spring-retry的模板类，RetryTemplate。
   	RetryTemplate retryTemplate = new RetryTemplate();
   
   	// 设置重试规则
   	retryTemplate.setRetryPolicy(......);
	
	//发起请求
	return retryTemplate.execute(callback, recoveryCallback);
}
```

唠唠叨叨了好多，就先到这了~

### 参考文献

http://blog.didispace.com/spring-cloud-zuul-retry-detail/


---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
