---
title: springcloud：使用总结
date: 2019-05-18 18:32:08
type: "tags"
tags: [springcloud,总结]
---

> springcloud使用了也有快1年了，整理总结下springcloud的一些使用方法，持续更新。 🥕

<!--more-->

该文章记录总结使用的**springboot**版本为**2.0.8.RELEASE**，对应的**springcloud**版本为**Finchley.RELEASE**。

## 分布式链路跟踪(zipkin+sleuth)

> 微服务架构是一个分布式架构，它按业务划分服务单元，一个分布式系统往往有很多个服务单元。由于服务单元数量众多，业务的复杂性，如果出现了错误和异常，很难去定位。主要体现在，一个请求可能需要调用很多个服务，而内部服务的调用复杂性，决定了问题难以定位。所以微服务架构中，必须实现分布式链路追踪，去跟进一个请求到底有哪些服务参与，参与的顺序又是怎样的，从而达到每个请求的步骤清晰可见，出了问题，很快定位。

### 安装zipkin

> 直接采用**消息总线rabbitmq**方式，使服务和zipkin解耦。需要注意的是这种方式默认的队列是叫zipkin，而且是由zipkin服务端创建的，**所以需要先启动zipkin**。

下载最新版的zipkin.jar，然后就可以根据以下命令启动：

```
wget -O zipkin.jar 'https://search.maven.org/remote_content?g=io.zipkin.java&a=zipkin-server&v=LATEST&c=exec'
RABBIT_ADDRESSES=127.0.0.1:5672 RABBIT_USER=admin RABBIT_PASSWORD=admin nohup java -jar zipkin.jar &
```

如果用Docker的话，直接
```
docker run -d -p 9411:9411 openzipkin/zipkin
```

可以配置的参数如下：

| 属性 | 环境变量 | 描述 |
| ------ | ------ | ------ |
| zipkin.collector.rabbitmq.concurrency | RABBIT_CONCURRENCY | 并发消费者数量，默认为1 |
| zipkin.collector.rabbitmq.connection-timeout | RABBIT_CONNECTION_TIMEOUT | 建立连接时的超时时间，默认为 60000毫秒，即 1 分钟 |
| zipkin.collector.rabbitmq.queue | RABBIT_QUEUE | 从中获取 span 信息的队列，默认为 zipkin |
| zipkin.collector.rabbitmq.uri | RABBIT_URI | 符合 RabbitMQ URI 规范 的 URI，例如amqp://user:pass@host:10000/vhost |

如果设置了 URI，则以下属性将被忽略。

| 属性 | 环境变量 | 描述 |
| ------ | ------ | ------ |
| zipkin.collector.rabbitmq.addresses | RABBIT_ADDRESSES | 用逗号分隔的 RabbitMQ 地址列表，例如localhost:5672,localhost:5673 |
| zipkin.collector.rabbitmq.password | RABBIT_PASSWORD | 连接到 RabbitMQ 时使用的密码，默认为 guest |
| zipkin.collector.rabbitmq.username | RABBIT_USER | 连接到 RabbitMQ 时使用的用户名，默认为guest |
| zipkin.collector.rabbitmq.virtual-host | RABBIT_VIRTUAL_HOST | 使用的 RabbitMQ virtual host，默认为 / |
| zipkin.collector.rabbitmq.use-ssl | RABBIT_USE_SSL | 设置为true则用 SSL 的方式与 RabbitMQ 建立链接 |

### 服务接入

> 因为采用的**消息总线rabbitmq**方式，zipkin宕机的话，相应的链路的信息会在队列中存放，重启后又会消费掉。

在对应微服务的pom增加依赖：
```java
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

然后增加对应的配置：
```java
spring:
  application:
    name: login
  cloud:
    bus:
      trace:
        enabled: true
    stream:
      default-binder: rabbit
  zipkin:
    #base-url: http://localhost:9411
    sender:
      type: rabbit
  sleuth:
    web:
      client:
        enabled: true
    sampler:
      probability: 1.0 # 采样比例设置为1.0也就是全部都需要。
  rabbitmq:
    host: 127.0.0.1
    username: admin
    password: admin
```

这样就配置完成了，触发请求，可以得到服务间调用的链路，耗时等信息。

![zipkin](https://github.com/7le/7le.github.io/raw/master/image/springcloud/zipkin.PNG)

## 服务重试机制

> 微服务架构下会衍生出非常多的服务调用，而在一些极端网络情况下，会出现服务调用超时或者失败等，这时候我们就需要加入服务重试机制，来保证好的用户体验。

需要引入的依赖:
```java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

然后对应的配置文件：
```java
feign:
  hystrix:
    enabled: true
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: true
        isolation:
          thread:
            timeoutInMilliseconds: 7000
ribbon:
  ConnectTimeout: 5000                # 请求连接的超时时间
  ReadTimeout:  5000                  # 请求处理的超时时间
  MaxAutoRetries: 1                   # 对当前实例的重试次数
  MaxAutoRetriesNextServer: 2         # 切换实例的重试次数
  OkToRetryOnAllOperations: true      # 对所有操作请求都进行重试
```

> 需要注意点是：Hystrix的超时时间大于Ribbon的超时时间，否则Hystrix命令超时后，该命令直接熔断，重试机制就没有任何意义了。

另外MaxAutoRetries和MaxAutoRetriesNextServer这样的设置，最多请求次数会是 2*3=6次，根据业务特性，自行优化设置。

对于网关（zuul）来说，需要额外增加依赖：

```java
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

配置稍微有些不同：
```java
zuul:
  retryable: true
  sensitive-headers:          # 传递原始的header信息到最终的微服务
ribbon:
  ConnectTimeout: 500                 # 请求连接的超时时间
  ReadTimeout:  1000                  # 请求处理的超时时间
  MaxAutoRetries: 2                   # 对当前实例的重试次数
  MaxAutoRetriesNextServer: 1         # 切换实例的重试次数
  OkToRetryOnAllOperations: true      # 对所有操作请求都进行重试
```

需要加上``retryable: true``

## JPA打印sql优化

> 因为平台使用的是jpa，而jpa不支持输出完整的sql，下面的方式通过依赖log4jdbc的方式可以实现。

需要引入的依赖:
```java
<dependency>
    <groupId>com.googlecode.log4jdbc</groupId>
    <artifactId>log4jdbc</artifactId>
    <version>1.2</version>
</dependency>
```

修改配置文件：
```java
spring:
  datasource:
    url: jdbc:log4jdbc:postgresql://127.0.0.1:5432/shine
    username: ${ymsg.datasource.username}
    password: ${ymsg.datasource.password}
    driver-class-name: net.sf.log4jdbc.DriverSpy
```

这样就可以实现输出完整的sql（包括占位符也能替换），因为输出的日志较多，通过logback进行过滤，具体的配置如下：
```java
<logger name="jdbc.sqlonly" level="OFF" />
<logger name="jdbc.audit" level="OFF" />
<logger name="jdbc.resultset" level="OFF" />
<logger name="jdbc.connection" level="INFO" />
<logger name="jdbc.sqltiming" level="INFO" />
```

最后得到的效果就是：
```java
2019-05-22 20:22:22.129 INFO shine [elastic-2] [jdbc.sqltiming] - select * from shine m where m.type = 1 order by m.create_time limit 5 
 {executed in 11 msec} 
 ``` 
其中executed就是执行的时间，可以用来排查慢查询。

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
