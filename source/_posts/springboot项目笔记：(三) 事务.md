---
title: springboot项目笔记：(三) 事务
date: 2017-06-11 22:49:09
type: "tags"
tags: [springboot]
---

>在一个项目中，事务，异常处理和拦截器都是必不可少的，今天就先记录一下事务的笔记~ 🥕

<!--more-->

### 事务

#### 具体配置

springboot 使用事务非常简单，我们只需要使用注解 @EnableTransactionManagement开启事务支持后，然后在访问数据库的Service方法上添加注解。

我们的依赖中添加的是spring-boot-starter-jdbc，框架会默认为我们注入DataSourceTransactionManager实例，如果是我们依赖中添加的是spring-boot-starter-data-jpa，那注入的JpaTransactionManager实例。因为他们都是实现自接口PlatformTransactionManager。

具体的代码如下：
```java
@EnableTransactionManagement
@SpringBootApplication
public class VideoApplication implements TransactionManagementConfigurer {


	@Resource(name="txManager1")
	private PlatformTransactionManager txManager1;

	// 创建事务管理器1
	@Bean(name = "txManager1")
	public PlatformTransactionManager txManager(DataSource dataSource) {
		return new DataSourceTransactionManager(dataSource);
	}
	// 实现接口 TransactionManagementConfigurer方法，其返回值代表在拥有多个事务管理器的情况下默认使用的事务管理器
	@Override
	public PlatformTransactionManager annotationDrivenTransactionManager() {
		return txManager1;
	}

	public static void main(String[] args) {
		SpringApplication.run(VideoApplication.class, args);
	}

}
```

配置好之后，我们只需要在需要的地方加上注释：
```java
@Service
public class CollectServiceImpl extends BaseServiceImpl implements CollectService {


    @Override
    @Transactional(rollbackFor = Exception.class)
    public void collect(Integer userId, Integer videoId) {

        if(collectMapper.selectByUidAndVid(userId,videoId)!=null){
            throw new HttpMessageNotReadableException("该视频已经收藏");
        }

        Collect collect=new Collect();
        collect.setUid(userId);
        collect.setVid(videoId);
        collect.setDeleteFlag(Constant.NO_DELETE);
        collect.setCreatedAt(new Date());
        collectMapper.insert(collect);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public void delete(Integer userId, Integer videoId) {
        Collect collect=collectMapper.selectByUidAndVid(userId,videoId);
        collect.setDeleteFlag(Constant.DELETE);
        collectMapper.updateByPrimaryKeySelective(collect);
    }

    @Override
    @Transactional(rollbackFor = Exception.class)
    public List<Collect> page(Integer userId) {
        return collectMapper.page(userId);
    }

}
```

> 注：如果Spring容器中存在多个PlatformTransactionManager实例，并且没有实现接口TransactionManagementConfigurer 指定默认值，在我们在方法上使用注解@Transactional的时候，就必须要用value指定，如果不指定，则会抛出异常。就需要这样指定@Transactional(value="txManager1")

#### 注释配置事务的传播行为和隔离级别
> 贴上注释的重要参数，以备方便查阅

> 事物传播行为介绍: 
@Transactional(propagation=Propagation.REQUIRED) 
如果有事务, 那么加入事务, 没有的话新建一个(默认情况下)
@Transactional(propagation=Propagation.NOT_SUPPORTED) 
容器不为这个方法开启事务
@Transactional(propagation=Propagation.REQUIRES_NEW) 
不管是否存在事务,都创建一个新的事务,原来的挂起,新的执行完毕,继续执行老的事务
@Transactional(propagation=Propagation.MANDATORY) 
必须在一个已有的事务中执行,否则抛出异常
@Transactional(propagation=Propagation.NEVER) 
必须在一个没有的事务中执行,否则抛出异常(与Propagation.MANDATORY相反)
@Transactional(propagation=Propagation.SUPPORTS) 
如果其他bean调用这个方法,在其他bean中声明事务,那就用事务.如果其他bean没有声明事务,那就不用事务.

> 事物超时设置:
@Transactional(timeout=30) //默认是30秒

> 事务隔离级别:
@Transactional(isolation = Isolation.READ_UNCOMMITTED)
读取未提交数据(会出现脏读, 不可重复读) 基本不使用
@Transactional(isolation = Isolation.READ_COMMITTED)
读取已提交数据(会出现不可重复读和幻读)
@Transactional(isolation = Isolation.REPEATABLE_READ)
可重复读(会出现幻读)
@Transactional(isolation = Isolation.SERIALIZABLE)
串行化

> MYSQL: 默认为REPEATABLE_READ级别
SQLSERVER: 默认为READ_COMMITTED

> 脏读 : 一个事务读取到另一事务未提交的更新数据
不可重复读 : 在同一事务中, 多次读取同一数据返回的结果有所不同, 换句话说, 后续读取可以读到另一事务已提交的更新数据. 相反, "可重复读"在同一事务中多次读取数据时, 能够保证所读数据一样, 也就是后续读取不能读到另一事务已提交的更新数据
幻读 : 一个事务读到另一个事务已提交的insert数据

> rollbackFor：
指定单一异常类：@Transactional(rollbackFor=RuntimeException.class)
指定多个异常类：@Transactional(rollbackFor={RuntimeException.class, Exception.class})

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)