---
title: springboot项目笔记：(四) 拦截器和异常处理
date: 2017-06-13 01:17:04
type: "tags"
tags: [springboot]
---

> 之前简单的说了springboot怎么处理事务，接下来就是记录下关于拦截器和异常处理如何在springboot使用。 🥕

<!--more-->

### 拦截器

#### 拦截器实现
spring中拦截器主要分两种，一个是HandlerInterceptor，一个是MethodInterceptor。其中HandlerInterceptor是属于springmvc中的拦截器，它拦截的目标是请求的地址，比MethodInterceptor先执行。

一般我们可以继承HandlerInterceptorAdapter，或者实现HandlerInterceptor接口，来完成。代码如下：
```
/**
 * 登录拦截器
 */
public class LoginInterceptor extends HandlerInterceptorAdapter {

    @Autowired
    private UserMapper userMapper;
    @Autowired
    private RedisUtil redisUtil;

    //预处理回调方法，实现处理器的预处理（如登录检查）
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        String token =request.getHeader("Authorization");
        if(token!=null){
            String name= EncryptUtil.aesDecrypt(token, EncryptUtil.KEY);
            User user=userMapper.selectByUsername(name);
            String toke1=redisUtil.get(name).toString();
            if(token.equals(toke1)){
                request.setAttribute("token",token);
                request.setAttribute("type",user.getType());
                request.setAttribute("userId",user.getUid());
                request.setAttribute("name",user.getName());
                return true;
            }
        }

        ResultBean resultBean=ResultBean.error("token无效");
        response.setCharacterEncoding("UTF-8");
        response.setContentType("application/json; charset=utf-8");
        PrintWriter out = null;
        try {
            out = response.getWriter();
            out.append(JSON.toJSONString(resultBean));
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (out != null) {
                out.close();
            }
        }

        return false;
    }
}

```
这里我们是集成了HandlerInterceptorAdapter，我们只需要将需要的方法进行重写。一般拦截器中会有如下方法：

* 1、preHandle：预处理回调方法，实现处理器的预处理（如登录检查），第三个参数为响应的处理器返回值：true表示继续流程（如调用下一个拦截器或处理器）；false表示流程中断（如登录检查失败），不会继续调用其他的拦截器或处理器，此时我们需要通过response来产生响应；
* 2、postHandle：后处理回调方法，实现处理器的后处理（但在渲染视图之前），此时我们可以通过modelAndView（模型和视图对象）对模型数据进行处理或对视图进行处理，modelAndView也可能为null。
* 3、afterCompletion：整个请求处理完毕回调方法，即在视图渲染完毕时回调，如性能监控中我们可以在此记录结束时间并输出消耗时间，还可以进行一些资源清理，类似于try-catch-finally中的finally，但仅调用处理器执行链中preHandle返回true的拦截器的afterCompletion。
* 4、集成HandlerInterceptorAdapter会多一个afterConcurrentHandlingStarted方法，该方法是用来处理异步请求。当Controller中有异步请求方法的时候会触发该方法。 异步请求先支持preHandle、然后执行afterConcurrentHandlingStarted。异步线程完成之后执行preHandle、postHandle、afterCompletion。如果需要用实现接口的方式，可以实现AsyncHandlerInterceptor接口。

#### 添加拦截器

拦截器写好后，我们只需要做以下的操作。我们可以自己定义需要拦截的路由，代码如下：

```
@SpringBootApplication
public class VideoApplication implements TransactionManagementConfigurer {

	@Override
    public void addInterceptors(InterceptorRegistry registry) {
        super.addInterceptors(registry);
        registry.addInterceptor(loginInterceptor()).addPathPatterns("/user/**","/collect/**","/register","/video/**").excludePathPatterns("/video/show/**","/video","/video/transCode");
    }

}
```
这里有个坑，需要大家注意，如果我们在拦截器中使用了@Autowired之类的注释，我们需要通过@Bean的声明，表明拦截器是Spring管理的bean，依赖注入工作自然Spring会做处理。不然我们的service将会为null。

具体代码如下：
```
@SpringBootApplication
public class VideoApplication implements TransactionManagementConfigurer {

	@Configuration
	static class WebMvcConfigurer extends WebMvcConfigurerAdapter {

		@Bean
		LoginInterceptor loginInterceptor() {
			return new LoginInterceptor();
		}

		public void addInterceptors(InterceptorRegistry registry) {
			registry.addInterceptor(loginInterceptor()).addPathPatterns("/user/**","/collect/**","/register","/video/**").excludePathPatterns("/video/show/**","/video","/video/transCode");
		}
	}
}
```

### 异常处理

异常处理我们通常都是基于spring的AOP来显示。这个没什么废话，直接上代码：

```
/**
 * 全局的异常处理切面类
 * Created by 7le on 2017/5/18 0018.
 */
@ControllerAdvice
@ResponseBody
public class ExceptionAdvice extends BaseController{

    /**
     * 400 - Bad Request
     */
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResultBean handleHttpMessageNotReadableException(HttpMessageNotReadableException e) {
        videoLogger.error("参数解析失败", e);
        return new ResultBean(HttpStatus.BAD_REQUEST.value(),e.getMessage());
    }

    /**
     * 405 - Method Not Allowed
     */
    @ResponseStatus(HttpStatus.METHOD_NOT_ALLOWED)
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResultBean handleHttpRequestMethodNotSupportedException(HttpRequestMethodNotSupportedException e) {
        videoLogger.error("不支持当前请求方法", e);
        return new ResultBean(HttpStatus.METHOD_NOT_ALLOWED.value(),HttpStatus.METHOD_NOT_ALLOWED.getReasonPhrase());
    }

    /**
     * 415 - Unsupported Media Type
     */
    @ResponseStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
    @ExceptionHandler(HttpMediaTypeNotSupportedException.class)
    public ResultBean handleHttpMediaTypeNotSupportedException(Exception e) {
        videoLogger.error("不支持当前媒体类型", e);
        return new ResultBean(HttpStatus.UNSUPPORTED_MEDIA_TYPE.value(),HttpStatus.UNSUPPORTED_MEDIA_TYPE.getReasonPhrase());
    }

    /**
     * 500 - Internal Server Error
     */
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler(Exception.class)
    public ResultBean handleException(Exception e) {
        videoLogger.error("服务运行异常", e);
        return new ResultBean(HttpStatus.INTERNAL_SERVER_ERROR.value(),HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase());
    }
}

```
补充一下@ControllerAdvice是spring3.2的新注释，从名字上可以看出大体意思是控制器增强。

这里就是一些基本的http状态码对应的异常处理，有需要可以自己添加，springboot中拦截器和异常处理的使用就记录到这了。

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)