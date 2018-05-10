---
title: springcloud：实现zuul的动态路由和集群通知
date: 2018-04-18 20:13:08
type: "tags"
tags: [springcloud]
---

> springboot2.0出来也有一小段时间了，之前一直钟情于vert.x的响应式编程，现在springboot也支持了，那就结合springcloud全家桶玩起来~

<!--more-->

微服务中网关至关重要，相应的解决方案也比较多，像 Nginx+ Lua ，Spring Cloud Zuul， Spring Cloud Gateway（spring不再继续兼容zuul2.0，自研的网关中间件）。

因为Spring Cloud Gateway尚在观望，就先用zuul来做网关，比较坑的是它的很多功能需要自行实现。

### 动态路由

实现动态路由的方案一般有两种：

* DiscoveryClientRouteLocator的重新覆盖
* 实现了RefreshableRouteLocator接口 

两种方案选了第二种去尝试，具体实现有大佬已经写过文章了，就不赘述了 [Zuul动态路由](https://blog.csdn.net/u013815546/article/details/68944039)。看了下相应的源码，是通过事件来刷新路由。为什么要这样实现呢，其实看了源码会发现，**Spring Cloud Zuul**是基于spring的事件驱动模型。（PS. 好多组件都用了这个模式）

源码中实现了ApplicationListener的监听器ZuulRefreshListener：
```
//刷新路由
private static class ZuulRefreshListener
		implements ApplicationListener<ApplicationEvent> {
	@Autowired
	private ZuulHandlerMapping zuulHandlerMapping;
	private HeartbeatMonitor heartbeatMonitor = new HeartbeatMonitor();
	@Override
	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof ContextRefreshedEvent
				|| event instanceof RefreshScopeRefreshedEvent
				|| event instanceof RoutesRefreshedEvent) {
			this.zuulHandlerMapping.setDirty(true);
		}
		else if (event instanceof HeartbeatEvent) {
			if (this.heartbeatMonitor.update(((HeartbeatEvent) event).getValue())) {
				this.zuulHandlerMapping.setDirty(true);
			}
		}
	}
}
```
那我们只需要通过publish RoutesRefreshedEvent就可以触发。

### 集群通知

上面虽然实现了动态路由，但是现在的服务为了保证**高可用**，不可能只有一个节点，网关也不例外。那每次要更新网关的路由，就要一个个节点去触发更新，那就太不优雅了。所以再折腾下，以达到触发单个节点，可以通知整个集群进行更新。

首先我们需要引入
```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

#### Spring Cloud Bus

**Spring Cloud Bus**的主要任务是将Spring的事件处理机制（又是事件驱动模型）和消息中间件消息的发送和接收整合起来，可以实现多个节点之间的通信，正符合我们的需求，下面就混带着源码和扩展来讲下如何实现。

##### 创建 endpoint

首先，我们可以参考源码中的端点，如``RefreshBusEndpoint``来实现我们自己的端点：``AutoRouteEndpoint``

```
@Endpoint(id = "route-refresh")
public class AutoRouteEndpoint extends AbstractBusEndpoint {

    public AutoRouteEndpoint(ApplicationEventPublisher context, String appId) {
        super(context, appId);
    }

    @WriteOperation
    public void refreshRouteWithDestination(@Selector String destination) { //TODO: document destination
        publish(new AutoRouteEvent(this, getInstanceId(), destination));
    }

    @WriteOperation
    public void refreshRoute() {
        publish(new AutoRouteEvent(this, getInstanceId(), null));
    }
}
```

```
@Configuration
public class ActuatorExtConfig {

    @Bean
    @ConditionalOnEnabledEndpoint
    public AutoRouteEndpoint autoRouteEndpoint(ApplicationContext context, BusProperties bus) {
        return new AutoRouteEndpoint(context,bus.getId());
    }

}
```

##### 接收消息

然后主要是注意源码中的``BusAutoConfiguration``类，下面是接受消息的代码：

```
@EventListener(classes = RemoteApplicationEvent.class)
	public void acceptLocal(RemoteApplicationEvent event) {
		if (this.serviceMatcher.isFromSelf(event)
				&& !(event instanceof AckRemoteApplicationEvent)) {
			this.cloudBusOutboundChannel.send(MessageBuilder.withPayload(event).build());
		}
	}
```

通过``@EventListener``来注册监听者，简化了以前需要实现``ApplicationListener``(像上面的``ZuulRefreshListener``)。

这个注解如何实现注册呢，需要通过``EventListenerFactory``的实现类

```
public class DefaultEventListenerFactory implements EventListenerFactory, Ordered {

	private int order = LOWEST_PRECEDENCE;

	@Override
	public int getOrder() {
		return order;
	}

	public void setOrder(int order) {
		this.order = order;
	}

	public boolean supportsMethod(Method method) {
		return true;
	}

	@Override
	public ApplicationListener<?> createApplicationListener(String beanName, Class<?> type, Method method) {
		return new ApplicationListenerMethodAdapter(beanName, type, method);
	}

}
```

具体大家可以跟下``ApplicationListenerMethodAdapter``就清晰了。

那么我们需要创建一个更新路由的event和一个Listener来接受消息

```
public class AutoRouteEvent extends RemoteApplicationEvent {

    protected AutoRouteEvent() {
    }

    protected AutoRouteEvent(Object source, String originService) {
        super(source, originService, null);
    }

    public AutoRouteEvent(Object source, String originService, String destinationService) {
        super(source, originService, destinationService);
    }

}
```

```
@Configuration
@RemoteApplicationEventScan({"com.cloud.gateway.event"})
public class BusConfiguration {

    @Autowired
    RouteRefreshService routeRefreshService;

    @EventListener(classes = AutoRouteEvent.class)
    public void routeRefresh() {
        routeRefreshService.routeRefresh();
    }
}

```

##### 发送消息

代码同样在``BusAutoConfiguration``类

```
@StreamListener(SpringCloudBusClient.INPUT)
	public void acceptRemote(RemoteApplicationEvent event) {
		if (event instanceof AckRemoteApplicationEvent) {
			if (this.bus.getTrace().isEnabled() && !this.serviceMatcher.isFromSelf(event)
					&& this.applicationEventPublisher != null) {
				this.applicationEventPublisher.publishEvent(event);
			}
			// If it's an ACK we are finished processing at this point
			return;
		}
		if (this.serviceMatcher.isForSelf(event)
				&& this.applicationEventPublisher != null) {
			if (!this.serviceMatcher.isFromSelf(event)) {
				this.applicationEventPublisher.publishEvent(event);
			}
			if (this.bus.getAck().isEnabled()) {
				AckRemoteApplicationEvent ack = new AckRemoteApplicationEvent(this,
						this.serviceMatcher.getServiceId(),
						this.bus.getAck().getDestinationService(),
						event.getDestinationService(), event.getId(), event.getClass());
				this.cloudBusOutboundChannel
						.send(MessageBuilder.withPayload(ack).build());
				this.applicationEventPublisher.publishEvent(ack);
			}
		}
		if (this.bus.getTrace().isEnabled() && this.applicationEventPublisher != null) {
			// We are set to register sent events so publish it for local consumption,
			// irrespective of the origin
			this.applicationEventPublisher.publishEvent(new SentApplicationEvent(this,
					event.getOriginService(), event.getDestinationService(),
					event.getId(), event.getClass()));
		}
	}
```

一样通过注解来注册，当接收到消息，根据消息的来源，目的地（destination）配置等信息，将数据转化为``RemoteApplicationEvent``对象，再次publish到spring context中。这一块代码可以复用，不需要重写。

自此我们就已经实现了动态路由和集群通知了~


#### Spring Cloud Actuator

**Spring Cloud Bus**中的endpoint是依赖于Actuator，它的主要作用是用于监控与管理。提供了许多端点，可以查看信息，也可以自定义端点（像上面的**route-refresh**）。

Actuator 部分端点：

| HTTP 方法|     路径|   描述|
| :-------- | --------:| :------: |
|GET|/beans|描述应用程序上下文里全部的Bean，以及它们的关系|
|GET|/health|健康检查     |
|GET|/env|获取全部环境属性     |
|GET|/env/{toMatch}|根据名称获取特定的环境属性值     |
|GET|/configprops|描述配置属性(包含默认值)如何注入Bean     |
|GET|/mappings| 描述全部的URI路径，以及它们和控制器(包含Actuator端点)的映射关系    |
|GET|/metrics|报告各种应用程序度量信息，比如内存用量和HTTP请求计数   |
|GET|/metrics/{requiredMetricName}|报告指定名称的应用程序度量值     |
|POST|/bus-refresh|端点手动刷新配置     |
|GET|/httptrace|提供基本的HTTP请求跟踪信息(时间戳、HTTP头等)     |

需要注意的是，对于spring-boot2.x 需要自己开放端点，配置如下：

```
management:
  endpoints:
    web:
      exposure:
        include: '*'   # 代表全部放开，可以自行选择
      base-path: /application
```

实现的代码在 [springcloud-config](https://github.com/7le/springcloud-analysis/tree/master/config)


---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
