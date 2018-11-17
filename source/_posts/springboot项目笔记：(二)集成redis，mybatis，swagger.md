---
title: springboot项目笔记：(二)集成redis，mybatis，swagger
date: 2017-06-04 22:42:55
type: "tags"
tags: [springboot]
---

>在搭建完springboot项目后，接下来就是要集成一些好东西。接下来本文中会集成redis，mybatis，swagger，pagehelper。

<!--more-->

先看下项目结构
![path](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-2-1.png)

### 集成redis

Redis作为一个高性能的key-value数据库，相较Memcache有了更加丰富的数据类型，更加迷人的就是redis的持久化，一般有RDB和AOF两种。RDB是记录一段时间内的操作，而AOF可以实现每次操作都持久化。这里就不详细介绍redis，直奔正题。 

#### 引入依赖
在pom.xml引入redis相应的依赖，（不清楚的可以看上一篇文章的pom.xml）

#### 添加Redis相关配置

在application.properties中添加：
```
#redis
spring.redis.host=XXX.XXX.XXX.XXX
spring.redis.password=XXXXXXXX
spring.redis.port=XXX
spring.redis.pool.max-idle=100
spring.redis.pool.min-idle=1
spring.redis.pool.max-active=1000
spring.redis.pool.max-wait=-1
```

#### Redis配置类
创建RedisConfig，相应的代码如下

```
/**
 * Redis配置类
 * Created by 7le on 2017/5/20 0020.
 */
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurationSelector{
    /**
     * 生成key的策略
     *
     * @return
     */
    @Bean
    public KeyGenerator keyGenerator() {
        return new KeyGenerator() {
            @Override
            public Object generate(Object target, Method method, Object... params) {
                StringBuilder sb = new StringBuilder();
                sb.append(target.getClass().getName());
                sb.append(method.getName());
                for (Object obj : params) {
                    sb.append(obj.toString());
                }
                return sb.toString();
            }
        };
    }

    /**
     * 管理缓存
     *
     * @param redisTemplate
     * @return
     */
    @SuppressWarnings("rawtypes")
    @Bean
    public CacheManager cacheManager(RedisTemplate redisTemplate) {
        RedisCacheManager rcm = new RedisCacheManager(redisTemplate);
        //设置缓存过期时间
        rcm.setDefaultExpiration(60*24);//秒
        //设置value的过期时间
        //Map<String,Long> map=new HashMap();
        //map.put("test",60L);
        //rcm.setExpires(map);
        return rcm;
    }

    /**
     * RedisTemplate配置
     * @param factory
     * @return
     */
    @Bean
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
        StringRedisTemplate template = new StringRedisTemplate(factory);
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);
        template.setValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }
}

```

#### 简单的RedisUtil
配置完后为了方便使用，抽象出一些简单的公用方法
```
/**
 * redis cache 工具类
 * Created by 7le on 2017/5/20 0020.
 */

@Component
public class RedisUtil {
    @SuppressWarnings("rawtypes")
    @Autowired
    private RedisTemplate redisTemplate;
    /**
     * 批量删除对应的value
     *
     * @param keys
     */
    public void remove(final String... keys) {
        for (String key : keys) {
            remove(key);
        }
    }
    /**
     * 批量删除key
     *
     * @param pattern
     */
    public void removePattern(final String pattern) {
        Set<Serializable> keys = redisTemplate.keys(pattern);
        if (keys.size() > 0)
            redisTemplate.delete(keys);
    }
    /**
     * 删除对应的value
     *
     * @param key
     */
    public void remove(final String key) {
        if (exists(key)) {
            redisTemplate.delete(key);
        }
    }
    /**
     * 判断缓存中是否有对应的value
     *
     * @param key
     * @return
     */
    public boolean exists(final String key) {
        return redisTemplate.hasKey(key);
    }
    /**
     * 读取缓存
     *
     * @param key
     * @return
     */
    public Object get(final String key) {
        Object result = null;
        ValueOperations<Serializable, Object> operations = redisTemplate.opsForValue();
        result = operations.get(key);
        return result;
    }
    /**
     * 写入缓存
     *
     * @param key
     * @param value
     * @return
     */
    public boolean set(final String key, Object value) {
        boolean result = false;
        try {
            ValueOperations<Serializable, Object> operations = redisTemplate.opsForValue();
            operations.set(key, value);
            result = true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return result;
    }
    /**
     * 写入缓存
     *
     * @param key
     * @param value
     * @return
     */
    public boolean set(final String key, Object value, Long expireTime) {
        boolean result = false;
        try {
            ValueOperations<Serializable, Object> operations = redisTemplate.opsForValue();
            operations.set(key, value);
            redisTemplate.expire(key, expireTime, TimeUnit.MILLISECONDS);
            result = true;
        } catch (Exception e) {

            e.printStackTrace();
        }
        return result;
    }
}
```

到这一步就大功告成了，大家就可以使用redis了，测试类这里就不写了，大家需要的话，可以直接去download代码 
[springboot](https://github.com/7le/video) ^.^

### 集成mybatis

#### 引入依赖
同样的引入依赖，详见上一篇文章

#### 添加mybatis相关配置
根据项目路径配置：
![path](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-2-2.png)
在application.properties中添加：
```
#mybatis
mybatis.type-aliases-package=com.shine.video.dao
mybatis.mapper-locations=classpath:mapping/*.xml
```

#### 添加MybatisConf类
有了以上的配置，其实就可以使用mybatis了（要注意的是需要在接口加上@Mapper注释），这里一并将PageHelper集成了。

```
/*
 * 注册MyBatis分页插件PageHelper
 */

@Configuration
public class MybatisConf {
    @Bean
    public PageHelper pageHelper() {
        PageHelper pageHelper = new PageHelper();
        Properties p = new Properties();
        p.setProperty("offsetAsPageNum", "true");
        p.setProperty("rowBoundsWithCount", "true");
        p.setProperty("reasonable", "true");
        pageHelper.setProperties(p);
        return pageHelper;
    }
}
```

配置完之后，只要如下使用就可以了。相当简单粗暴，爱不释手。
```
	/**
     * 收藏列表展示接口
     * @return
     */
    @RequestMapping(value = "/collect",method = RequestMethod.GET)
    @ApiOperation(value="收藏列表", httpMethod = "GET" , notes="收藏列表")
    public ResultBean list(HttpServletRequest request,
                           @ApiParam(required = true, name = "pageNo", value = "第几页")
                           @RequestParam(defaultValue = "1", value = "pageNo") Integer pageNo,
                           @ApiParam(required = true, name = "pageSize", value = "每页显示条数")
                           @RequestParam(defaultValue = "10", value = "pageSize") Integer pageSize) throws Exception {

        if(Constant.USER_TYPE_SPECIAL!=(int)request.getAttribute("type")&&
                Constant.USER_TYPE_ORDINARY!=(int)request.getAttribute("type")){
            throw new HttpMessageNotReadableException("该用户没有查看收藏列表权限");
        }
        /*
         * 第一个参数是第几页；第二个参数是每页显示条数。
         */
        PageHelper.startPage(pageNo,pageSize);
        return ResultBean.success(collectService.page((Integer) request.getAttribute("userId")));
    }
```

### 集成swagger
看到上面的接口，大家可能会奇怪一些注释，比如@ApiOperation，@ApiParam，是不是好像不是属于spring家族的呀。这些就是swagger的注释。虽然这样的写法对代码有一定的侵入性，但是想想大家可以摆脱写接口文档了，这对于后台人员来说简直就是Gospel呀~ 另外在项目跟前端调式完后，我们也可以把所谓侵入的代码删除呀，总之就是不想写接口文档。哈哈哈

swagger的接口文档是restful的，大家要注意接口设计。以后有机会要补一篇restful的文章，总结下小弟自己的理解。

#### 引入依赖
老规矩，还是需要引入依赖，详见上篇文章。

#### 添加SwaggerConfig类

```
/**
 * @author 7le
 * @since 2017-05-17
 */
@Configuration
@EnableSwagger2
public class SwaggerConfig {


    @Bean
    public Docket createRestApi() {
        Predicate<RequestHandler> predicate = new Predicate<RequestHandler>() {
            @Override
            public boolean apply(RequestHandler input) {
                Class<?> declaringClass = input.declaringClass();
                if (declaringClass == BasicErrorController.class)// 排除
                    return false;
                if(declaringClass.isAnnotationPresent(RestController.class)) // 被注解的类
                    return true;
                if(input.isAnnotatedWith(ResponseBody.class)) // 被注解的方法
                    return true;
                return false;
            }
        };
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .useDefaultResponseMessages(false)
                .select()
                .apis(predicate)
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("Shine-video")//大标题
                .contact(new Contact("7le", "https://github.com/7le", "silk.heqian@gmail.com"))//作者
                .version("1.0")//版本
                .build();
    }


    /**
     * SpringBoot默认已经将classpath:/META-INF/resources/和classpath:/META-INF/resources/webjars/映射
     * 所以该方法不需要重写，如果在SpringMVC中，可能需要重写定义（我没有尝试）
     * 重写该方法需要 extends WebMvcConfigurerAdapter
     *
     */
//    @Override
//    public void addResourceHandlers(ResourceHandlerRegistry registry) {
//        registry.addResourceHandler("swagger-ui.html")
//                .addResourceLocations("classpath:/META-INF/resources/");
//
//        registry.addResourceHandler("/webjars/**")
//                .addResourceLocations("classpath:/META-INF/resources/webjars/");
//    }

    /**
     * 可以定义多个组，比如本类中定义把test和demo区分开了
     * （访问页面就可以看到效果了）
     *
     */
    @Bean
    public Docket testApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("test")
                .genericModelSubstitutes(DeferredResult.class)
//                .genericModelSubstitutes(ResponseEntity.class)
                .useDefaultResponseMessages(false)
                .forCodeGeneration(true)
                .pathMapping("/")// base，最终调用接口后会和paths拼接在一起
                .select()
                .paths(or(regex("/api/.*")))//过滤的接口
                .build()
                .apiInfo(testApiInfo());
    }

    @Bean
    public Docket demoApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .groupName("demo")
                .genericModelSubstitutes(DeferredResult.class)
//              .genericModelSubstitutes(ResponseEntity.class)
                .useDefaultResponseMessages(false)
                .forCodeGeneration(false)
                .pathMapping("/")
                .select()
                .paths(or(regex("/demo/.*")))//过滤的接口
                .build()
                .apiInfo(demoApiInfo());
    }

    private ApiInfo testApiInfo() {
        return new ApiInfoBuilder()
                .title("Electronic Health Record(EHR) Platform API")//大标题
                .description("EHR Platform's REST API, all the applications could access the Object model data via JSON.")//详细描述
                .version("1.0")//版本
                .termsOfServiceUrl("NO terms of service")
                .contact(new Contact("7le", "https://github.com/7le", "silk.heqian@gmail.com"))//作者
                .license("The Apache License, Version 2.0")
                .licenseUrl("http://www.apache.org/licenses/LICENSE-2.0.html")
                .build();
    }

    private ApiInfo demoApiInfo() {
        return new ApiInfoBuilder()
                .title("Electronic Health Record(EHR) Platform API")//大标题
                .description("EHR Platform's REST API, all the applications could access the Object model data via JSON.")//详细描述
                .version("1.0")//版本
                .termsOfServiceUrl("NO terms of service")
                .contact(new Contact("silk", "https://github.com/7le", "silk.heqian@gmail.com"))//作者
                .license("The Apache License, Version 2.0")
                .licenseUrl("http://www.apache.org/licenses/LICENSE-2.0.html")
                .build();

    }
}
```

配置后，然后像收藏列表展示接口,那样加上注释就行了，重复的代码就不写了。
最后呈现的效果不要太美~！

![swagger1](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-2-3.png)
![swagger2](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-2-4.png)

最后在加上代码的地址，大家可以互相交流学习[springboot](https://github.com/7le/video) 

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)