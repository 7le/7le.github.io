---
title: Rabbitmq：消息可靠发送
date: 2019-02-21 22:30:08
type: "tags"
tags: [mq]
---

> 接上文[分布式事务：基于可靠消息服务](https://7le.top/2018/12/04/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1%EF%BC%9A%E5%9F%BA%E4%BA%8E%E5%8F%AF%E9%9D%A0%E6%B6%88%E6%81%AF%E6%9C%8D%E5%8A%A1/#more)介绍了整体中间件的设计思路，有些内容没有展开。故此，本文详细讲解下如何将消息可靠发送到Rabbitmq。

<!--more-->

在上文提到了如何将消息进行可靠发布，因为[shine-mq](https://github.com/7le/shine-mq)是无缝集成``spring-boot-starter``的，所以rabbitmq的操作是基于``rabbitTemplate``来完成的。

``rabbitTemplate``提供了``setConfirmCallback``方法，可以在消息发送到RabbitMQ交换器后，进行ack的回调。

```
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    ...
    //消息能投入正确的消息队列，并持久化，返回的ack为true
    if (ack) {
        log.info("The message has been successfully delivered to the queue, correlationData:{}", correlationData);
        ...
    }
    ...
});
```

在此之前还需要设置``CachingConnectionFactory``

```
//设置生成者确认机制
rabbitConnectionFactory.setPublisherConfirms(true);
```

如果还需要设置``setReturnCallback``(消息发送到RabbitMQ交换器，但无相应Exchange时的回调)，那就还需要设置``rabbitTemplate``
```
//使用return-callback时必须设置mandatory为true
rabbitTemplate.setMandatory(true);
```

熟悉Rabbitmq的同学可能知道，Rabbitmq有两种机制来实现消息的可靠发布。

* 通过事务机制，这个上篇文章分析过，在这个模式下，rabbitmq的效率很低，不适合。
* Confirm模式，这个模式下会有三种方式，分别是：
    * 普通confirm模式：每发送一条消息后，调用waitForConfirms()方法，等待服务器端confirm。实际上是一种串行confirm了。
    * 批量confirm模式：每发送一批消息后，调用waitForConfirms()方法，等待服务器端confirm
    * 异步confirm模式：提供一个回调方法，服务端confirm了一条或者多条消息后Client端会回调这个方法。（``rabbitTemplate``设置回调就是这个模式）

> 所以我们知道了``rabbitTemplate``提供的确认机制是一种异步机制，并不能同步的发现问题，也就是说在极端的网络条件下是会出现消息丢失的。

所以[shine-mq](https://github.com/7le/shine-mq)通过增加一个``Coordinator``（协调者）来实现。``Coordinator``会保存2个状态，一个是prepare（**携带回查id**），这个状态在前文说过是用来保证上游服务的任务状态的。

而另一个状态ready，就是来保证消息的可靠投递。

首先[shine-mq](https://github.com/7le/shine-mq)是使用``@DistributedTrans``来开启。在这个注解的切面里，先持久化ready状态。

```
@Around(value = "@annotation(trans)")
public void around(ProceedingJoinPoint pjp, DistributedTrans trans) throws Throwable {
    ...
    try {
        EventMessage message = new EventMessage(exchange, routeKey, SendTypeEnum.DISTRIBUTED.toString(), checkBackId,
                coordinatorName, msgId);
        //将消息持久化
        coordinator.setReady(msgId, checkBackId.toString(), message);
        rabbitmqFactory.setCorrelationData(msgId, coordinatorName, message, null);
        rabbitmqFactory.addDLX(exchange, exchange, routeKey, null, null);
        if (flag) {
            rabbitmqFactory.add(MqConstant.DEAD_LETTER_QUEUE, MqConstant.DEAD_LETTER_EXCHANGE,
                    MqConstant.DEAD_LETTER_ROUTEKEY, null, null);
            flag = false;
        }
        rabbitmqFactory.getTemplate().send(message, 0, 0, SendTypeEnum.DISTRIBUTED);
    } catch (Exception e) {
        log.error("Message failed to be sent : ", e);
        throw e;
    }
}
```

然后在回调中删除该状态：

```
//消息发送到RabbitMQ交换器后接收ack回调
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (correlationData != null) {
        log.info("ConfirmCallback ack: {} correlationData: {} cause: {}", ack, correlationData, cause);
        String msgId = correlationData.getId();
        CorrelationDataExt ext = (CorrelationDataExt) correlationData;
        Coordinator coordinator = (Coordinator) applicationContext.getBean(ext.getCoordinator());
        //可以自定义实现的回调
        coordinator.confirmCallback(correlationData, ack);
        //消息能投入正确的消息队列，并持久化，返回的ack为true
        if (ack) {
            log.info("The message has been successfully delivered to the queue, correlationData:{}", correlationData);
            //删除ready状态
            coordinator.delStatus(msgId);
        } else {
            ...
        }
    }
});
```

因为存储ready是在上游服务任务执行之后的，所以只要有超时的ready记录未被清理掉，``daemon（守护线程）``只管捞起来进行重发就行，因为Mq的可靠性投递就已经要求下游服务是需要保证幂等性了。

最后还有个极端的情况，就是ready消息存储的时候因为网络抖动该消息丢失了，这时候也没有关系，因为有prepare状态会进行回查，该状态只有在ready存储后才会触发删除。

如果对你有帮助，那就帮忙点个星星把 ^.^
> github地址：**https://github.com/7le/shine-mq**

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)