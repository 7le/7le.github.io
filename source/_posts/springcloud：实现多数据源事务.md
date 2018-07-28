---
title: springcloud：实现多数据源事务
date: 2018-07-28 17:30:08
type: "tags"
tags: [springcloud,db]
---

> 最近在重构项目中，需要兼容多数据源，故此实现下多数据源事务。

<!--more-->

这次重构项目中，为了支持后续庞大的数据量接入，更迭了数据库，但是为了要兼容老版本，也不能直接拿掉老的数据库。所以就有了兼容多数据源的需求，尤其是要保证事务。

往往这时候一般会引入一些分布式事务的方案，但是都有点太重了，所以利用**AOP**来轻量的实现这个需求。

### 具体实现

用过**spring**的都知道，我们一般都是使用**@Transactional**注解，但是这个注解在多数据源下，只能支持制定的数据源（不指定就是默认的）。

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

### 注意事项

在使用的时候，需要注意一些细节，要加上``@EnableTransactionManagement``。

以及``@MultiTransactional({"xxxx","xxxxx"})``是**AOP**的方式，那就意味着是动态代理的，那下面的方式就会失效：

* private, protected方法无效
* 同一个class中public方法无效
* 注解写在父类抽象方法上


实现的代码在 [springcloud-gateway](https://github.com/7le/springcloud-analysis/tree/master/gateway/src/main/java/com/cloud/gateway)

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
