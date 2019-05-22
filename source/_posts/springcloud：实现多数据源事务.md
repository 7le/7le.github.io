---
title: springcloud：实现多数据源事务
date: 2018-07-28 17:30:08
type: "tags"
tags: [springcloud,分布式事务]
---

> 最近在重构项目中，需要兼容多数据源，故此实现下多数据源事务。 🥕

<!--more-->

这次重构项目中，为了支持后续庞大的数据量接入，更迭了数据库，但是为了要兼容老版本，也不能直接拿掉老的数据库。所以就有了兼容多数据源的需求，尤其是要保证事务。

其实这个需求就是要实现分布式事务，但是我们的这个场景是在一个服务内，所以可以利用**AOP**来轻量的实现这个需求，若是多个服务的话，就需要实现一个管理器。

### 具体实现

用过**spring**的都知道，我们一般都是使用**@Transactional**注解，但是这个注解在多数据源下，只能支持指定的数据源（不指定就是默认的）。

所以我们新建个自定义注解：

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface MultiTransactional {

    String[] values() default "";
}
```

再定义一个切面：

```
/**
 * 多数据源事务
 *
 * @author 7le
 */
@Slf4j
@Aspect
@Order(-7)
@Component
public class MultiTransactionalAspect {

    @Autowired
    private ApplicationContext applicationContext;

    @Around(value = "@annotation(multiTransactional)")
    public Object around(ProceedingJoinPoint pjp, MultiTransactional multiTransactional) throws Throwable {
        Stack<DataSourceTransactionManager> dataSourceTransactionManagerStack = new Stack<>();
        Stack<TransactionStatus> transactionStatusStack = new Stack<>();
        try {
            if (!openTransaction(dataSourceTransactionManagerStack, transactionStatusStack, multiTransactional)) {
                return null;
            }
            Object ret = pjp.proceed();
            commit(dataSourceTransactionManagerStack, transactionStatusStack);
            return ret;
        } catch (Throwable e) {
            rollback(dataSourceTransactionManagerStack, transactionStatusStack);
            log.error(String.format(
                    "MultiTransactionalAspect catch exception class: %s method: %s detail:", pjp.getTarget().getClass().getSimpleName(),
                    pjp.getSignature().getName()), e);
            throw e;
        }
    }

    private boolean openTransaction(Stack<DataSourceTransactionManager> dataSourceTransactionManagerStack,
                                    Stack<TransactionStatus> transactionStatusStack, MultiTransactional multiTransactional) {
        String[] transactionMangerNames = multiTransactional.values();
        if (ArrayUtils.isEmpty(multiTransactional.values())) {
            return false;
        }
        for (String beanName : transactionMangerNames) {
            DataSourceTransactionManager dataSourceTransactionManager = (DataSourceTransactionManager) applicationContext
                    .getBean(beanName);
            TransactionStatus transactionStatus = dataSourceTransactionManager
                    .getTransaction(new DefaultTransactionDefinition());
            transactionStatusStack.push(transactionStatus);
            dataSourceTransactionManagerStack.push(dataSourceTransactionManager);
        }
        return true;
    }


    private void commit(Stack<DataSourceTransactionManager> dataSourceTransactionManagerStack,
                        Stack<TransactionStatus> transactionStatusStack) {
        while (!dataSourceTransactionManagerStack.isEmpty()) {
            dataSourceTransactionManagerStack.pop().commit(transactionStatusStack.pop());
        }
    }

    private void rollback(Stack<DataSourceTransactionManager> dataSourceTransactionManagerStack,
                          Stack<TransactionStatus> transactionStatusStack) {
        while (!dataSourceTransactionManagerStack.isEmpty()) {
            dataSourceTransactionManagerStack.pop().rollback(transactionStatusStack.pop());
        }
    }
}
```

这样就大功告成，只需要在需要的方法上，加上``@MultiTransactional({"xxxx","xxxxx"})``

实现的代码在 [springcloud-gateway](https://github.com/7le/springcloud-analysis/tree/master/gateway/src/main/java/com/cloud/gateway)

### 注意事项

在使用的时候，需要注意一些细节，要加上``@EnableTransactionManagement``。

以及``@MultiTransactional({"xxxx","xxxxx"})``是**AOP**的方式，那就意味着是动态代理的，那下面的方式就会失效：

* private, protected方法无效 (Spring AOP使用JDK动态代理或者CGLIB来为目标对象创建代理。使用原生的Java的代理是无法代理protected和private类型的方法。CGLIB的代理虽然在技术上可以代理protected和private类型的方法,但是用于AOP的时候不推荐代理protected和private类型的方法.)
* 同一个class中public方法无效 (@Transactional的事务开启 ，是基于接口或者是类的代理被创建。所以在同一个类中一个无事务的方法调用另一个有事务的方法，将是对this引用进行调用而非代理，事务是不会起作用的。  )
* 注解写在父类抽象方法上

### 隐患

上面也提到过这个方案比较轻量，也是针对一些对数据一致性要求不高的场景，因为会存在数据不一致的可能。

我们用伪代码来描述下，假设2个数据源

```
begin1
begin2
sql1
sql2
commit1
commit2
```

这种方案是可以实现在sql1 sql2之间的异常回滚。如果出现commit1提交成功，commit2提交失败（或者超时）这种情况，就会造成数据不一致，虽然这种情况概率很低，但也是一个隐患。

这个实现很类似于**2PC**，都会有在一个参与者执行任务提交后，另一个参与者出现异常而导致数据不一致的问题。

其实一般情况下，系统的需求只是要达到**最终一致性**，那就可以考虑使用**TCC**，对每个事物进行Try，如果执行没有问题，再执行Confirm，如果执行过程中出了问题，则执行操作的逆操Cancel（自动化补偿手段）。

但是**TCC**对每个事务都需要Try，再执行Confirm，略微显得臃肿，根据不同的业务场景可以有更好的方案（比如补偿模式，定期校对模式之类），具体的可以看[分布式服务化系统一致性的“最佳实干”](https://www.jianshu.com/p/1156151e20c8)

### 分布式事务实现

抽了点时间，自己实现了一个分布式事务的中间件，在一些对数据一致性要求高的场景可以使用。[shine-mq](https://github.com/7le/shine-mq) 
相应的设计思路在 [分布式事务：基于可靠消息服务](https://7le.top/2018/12/04/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1%EF%BC%9A%E5%9F%BA%E4%BA%8E%E5%8F%AF%E9%9D%A0%E6%B6%88%E6%81%AF%E6%9C%8D%E5%8A%A1/#more)

### 参考文献

李艳鹏. 著. 分布式服务架构：原理、设计与实战[M].

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
