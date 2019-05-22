---
title: springcloud：webflux源码浅析
date: 2018-06-11 22:55:17
type: "tags"
tags: [springcloud]
---

> spring-webflux采用了事件驱动异步非阻塞模式，借助EventLoop以少量线程应对高并发的访问，这样的改变使spring也变得像vert.x一样有趣~ 🥕

<!--more-->

之前就一直很看好**spring-webflux**，最近这星期就花时间用了用，也看了自己感兴趣的源码（虽然还没看完），就先记录分享一下，省的到时候就忘了。

### webflux and vert.x

用过**vert.x**的小伙伴都知道，**vert.x**是函数式开发模式，而**spring-webflux**不但支持函数式，还很贴心的提供了webmvc注解的方式来编写web服务，这让学习和迁移代码上都有很大的便利。

```
@RestController
public class AdController {

    @RequestMapping(value = "/")
    public Mono<Object> test() {
        return Mono.create(s -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            s.success(ResultBean.SUCCESS);
        }).subscribeOn(Schedulers.elastic());
    }
}
```

之前我封装过**vert.x**，通过注解的方式来声明路由并统一管理，跟webflux非常相似。有兴趣的小伙伴可以去玩耍一下 [vertx-shine](https://github.com/7le/vertx-shine)

```
@RouteHandler
@RouteMapping(value = "/video")
public class VideoVerticle {

    private Vertx vertx = VerticleLauncher.getStandardVertx();

    @RouteMapping(method = RequestMethod.GET, value = "/test")
    public Handler<RoutingContext> test() {
        return routingContext -> vertx.executeBlocking(future -> {
            System.out.println("executeBlocking: "+Thread.currentThread().getName());
            System.out.println("type : " + routingContext.request().getParam("type"));
            //需要调用complete  FutureImpl -> setHandler 需要
            future.complete(1);
        }, false, h -> routingContext.response().setStatusCode(200).end("It is amazing !"));
    }
}
```

两者大概的思路都是一致的，通过一个eventloop线程来不断循环接收event，并将event（一些耗时的业务逻辑和网络IO等）分配给相应的handler（worker线程）来处理。

![event-loop](https://github.com/7le/7le.github.io/raw/master/image/springcloud/event-loop.png)

在**vert.x**中两者称为**EventLoop线程**和**Worker线程**，而**webflux**对应的是**NIO线程**和**elastic（single、parallel 看业务需要选择）线程**。

这是一个简化的模型，很便于理解，而真正有趣的地方是如何来实现这个模型。

### 源码分析

整个实现的源码比较多，先整理出一部分我比较感兴趣的，是关于线程处理完event后，到底是哪个线程返回结果。其实就是上述代码中``s.success(ResultBean.SUCCESS);``的玄机。

> 注： 如果**spring-boot-starter-web**和**spring-boot-starter-webflux**都依赖的话，会默认设置为SpringMVC。
不过在SpringMVC模式下使用异步类型（DeferredResult，Flux或SseEmitter），这将会是异步的，但读写仍然会被阻塞。

使用的依赖版本如下，Servlet容器使用的是**undertow**:
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
    <version>2.0.0.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
    <version>2.0.0.RELEASE</version>
</dependency>
```

可以在启动的时候指定使用**REACTIVE**或**SERVLET**
```
    SpringApplication app = new SpringApplication（MyApplication.class）;
    app.setWebApplicationType（WebApplicationType.REACTIVE）;
    app.run();
```

> 因为我的项目中有依赖会引入**spring-boot-starter-web**，所以接下来的分析是基于**SERVLET**，等之后整理了再分析基于**REACTIVE**

从``s.success()``方法进入，会经过很多类``MonoCreate``->``MonoSubscribeOn``->``ScopePassingSpanSubscriber``->``StrictSubscriber``->``ReactiveTypeHandler``，最后到``DeferredResult``的**setResultInternal**，这里很关键。

```
public class DeferredResult<T> {
    
    /**
     * 重要
     */ 
    private boolean setResultInternal(Object result) {
	// 查验是否过期
	if (isSetOrExpired()) {
		return false;
	}
	DeferredResultHandler resultHandlerToUse;
	//这里很关键
	synchronized (this) {
		// 抢占到了锁，再次查验是否过期
		if (isSetOrExpired()) {
			return false;
		}
		// 将传进来的结果赋值给result，注意result是volatile修饰的
		this.result = result;
		resultHandlerToUse = this.resultHandler;
		if (resultHandlerToUse == null) {
			// 是否设置了handler 没有设置返回true
			return true;
		}
		// 清除handler
		this.resultHandler = null;
	}
	// 如果设置了handler，进行回调
	resultHandlerToUse.handleResult(result);
	return true;
    }
}
```

我们继续找``DeferredResultHandler``函数式接口的实现，发现是在``WebAsyncManager``。向上去找源码会发现源头是我们熟悉的``RequestMappingHandlerAdapter``，那该方法就是在请求进来是**NIO线程**来执行的。

```
public final class WebAsyncManager {
    
    public void startDeferredResultProcessing(
    			final DeferredResult<?> deferredResult, Object... processingContext) throws Exception {
        ......
       
        try {
        	interceptorChain.applyPreProcess(this.asyncWebRequest, deferredResult);
        	// 这个setResultHandler方法，非常重要
        	deferredResult.setResultHandler(result -> {
        		result = interceptorChain.applyPostProcess(this.asyncWebRequest, deferredResult, result);
        		setConcurrentResultAndDispatch(result);
        	});
        }
        catch (Throwable ex) {
        	setConcurrentResultAndDispatch(ex);
        }
    }
}

public class DeferredResult<T> {

    /**
     * 重要
     */ 
    public final void setResultHandler(DeferredResultHandler resultHandler) {
    	Assert.notNull(resultHandler, "DeferredResultHandler is required");
    	// 查验是否过期
    	if (this.expired) {
    		return;
    	}
    	Object resultToHandle;
    	synchronized (this) {
    		// 抢占到了锁，再次查验是否过期
    		if (this.expired) {
    			return;
    		}
    		resultToHandle = this.result;
    		if (resultToHandle == RESULT_NONE) {
    		    //如果结果还没有被设置，将传进来的DeferredResultHandler赋值给resultHandler
    			this.resultHandler = resultHandler;
    			return;
    		}
    	}
    	// 走到这说明result已经被设置了，那就直接执行回调
    	try {
    		resultHandler.handleResult(resultToHandle);
    	}
    	catch (Throwable ex) {
    		logger.debug("Failed to handle existing result", ex);
    	}
    }
}

```

那我们继续顺着代码往下，假设``resultHandlerToUse.handleResult(result);``成功回调了，就会到**setConcurrentResultAndDispatch**方法中，也是一些类的跳转，``WebAsyncManager``->``StandardServletAsyncWebRequest``->``AsyncContextImpl``的**dispatchAsyncRequest**方法。

```
public class AsyncContextImpl implements AsyncContext {

    private void dispatchAsyncRequest(final ServletDispatcher servletDispatcher, final ServletPathMatch pathInfo, final HttpServerExchange exchange) {
        doDispatch(new Runnable() {
            @Override
            public void run() {
                Connectors.executeRootHandler(new HttpHandler() {
                    @Override
                    public void handleRequest(final HttpServerExchange exchange) throws Exception {
                        servletDispatcher.dispatchToPath(exchange, pathInfo, DispatcherType.ASYNC);
                    }
                }, exchange);
            }
        });
    }

    private synchronized void doDispatch(final Runnable runnable) {
        if (dispatched) {
            throw UndertowServletMessages.MESSAGES.asyncRequestAlreadyDispatched();
        }
        dispatched = true;
        final HttpServletRequestImpl request = servletRequestContext.getOriginalRequest();
        addAsyncTask(new Runnable() {
            @Override
            public void run() {
                request.asyncRequestDispatched();
                runnable.run();
            }
        });
        if (timeoutKey != null) {
            timeoutKey.remove();
        }
    }
    
    public synchronized void addAsyncTask(final Runnable runnable) {
        asyncTaskQueue.add(runnable);
        if (!processingAsyncTask) {
            processAsyncTask();
        }
    }
    
    private synchronized void processAsyncTask() {
        if (!initialRequestDone) {
            return;
        }
        updateTimeout();
        final Runnable task = asyncTaskQueue.poll();
        if (task != null) {
            processingAsyncTask = true;
            asyncExecutor().execute(new TaskDispatchRunnable(task));
        } else {
            processingAsyncTask = false;
        }
    }
}
```

``AsyncContextImpl``中的方法涉及比较多，**dispatchAsyncRequest**和**doDispatch**方法类似都是包装了一个``Runnable``，等待被启动，其中**dispatchToPath**方法最后会返回结果到请求方。而**addAsyncTask**和**processAsyncTask**维护了一个双端队列，这里不去深究，关注到``asyncExecutor().execute(new TaskDispatchRunnable(task));``，其中**asyncExecutor()**方法决定了最后是什么线程来启动之前的``Runnable``。跟下去会发现到了``XnioIoThread``，那之前疑惑的答案就已经出来了。

### 取其精华

前面唠叨了一堆，无非就是源码的运行过程，整理了一下线程的流程，如下图：

![thread-process](https://github.com/7le/7le.github.io/raw/master/image/springcloud/thread-process.png)

不过这图只是画出了一种理想的状态，因为**NIO**线程和**elastic**线程两者的时序是无法保证的，所以并不能保证**setResultHandler**是在**setResultInternal**之前执行，就我们上面源码中标识了重要的方法，没有注意到的可以翻回去看看。

这里就很有意思了，spring的设计是不管这2个线程的时序，而是通过**synchronized**的互斥，来保证同一时刻内只有一个线程能获得``DeferredResult``这个锁的对象。

若是先执行**setResultHandler**，会判断被**volatile**修饰的**result**，若没有被重新赋值，那就说明还未执行**setResultInternal**，那就设置**handler**等待**setResultInternal**运行来处理。相对的，如果先执行了**setResultInternal**，会将**result**赋值，有**handler**就让**elastic**执行回调，如果没有则让**NIO**线程自己执行回调。


---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
