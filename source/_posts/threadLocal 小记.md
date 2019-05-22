---
title: 'threadLocal 小记'
date: 2017-08-29 22:16:42
type: "tags"
tags: [java]
---

> 最近在工作的时候遇到一个诡异的情况，花了点时间仔细考究了下，特此记录下~ 🥕

<!--more-->

事情是这样滴，因为最近维护的一个的项目比较有"年代感"，里面的东西也比较庞杂，原先是使用struts的，后来我另外实现了一套springmvc，两者是共存的。（有点奇葩，因为以前struts的内容太多，全部改掉成本太大，也就这样一直放着）

### 时隐时现的bug

俗话说前人栽树，后人乘凉。原先代码里有这样一段代码：

![mybatis拦截器](https://github.com/7le/7le.github.io/raw/master/image/threadLocal/threadLocal-1.png)

显而易见，这是实现了Mybatis的接口，然后拦截它的update方法（其实也就是SqlSession的新增，删除，修改操作）。对于更新操作的时候，只要实体类继承了NulEntity，就会更新的时候自动加上更新时间和更新人，这样就不需要自己再另外操作了，确实挺方便的。但是问题来了，在做更新操作的时候，更新人偶尔会诡异的变成不是当前用户的名字。因为是新做的springmvc这一套，当时就干脆直接去掉了实体类的继承关系，问题就没了，也就没有在意。最近同事又遇到了这样的问题，来问我。于是乎，就花了点时间深究了下~

### 真相大白

其实更新人是从ThreadLocalHolder中拿的，看代码：

![ThreadLocalHolder](https://github.com/7le/7le.github.io/raw/master/image/threadLocal/threadLocal-2.png)

可以发现是利用ThreadLocal来实现，了解ThreadLocal的都知道，它是为每一个线程维护变量的副本，在 ThreadLocal 中定义了一个静态内部类 ThreadLocalMap，可以将其理解为一个特有的 Map 类型，而在 Thread 类中声明了一个 ThreadLocalMap 类型的属性 threadLocals，所以针对每个 Thread 对象，也就是每个线程来说都包含了一个 ThreadLocalMap 对象，即每个线程都有一个属于自己的内存数据库，而数据库中存储的就是我们用 ThreadLocal 修饰的属性，这里的 key 就是对应的 ThreadLocal 对象，而 value 就是我们记录在 ThreadLocal 中的值。所以会在每次请求的时候进行拦截，给ThreadLocal赋上登陆人的信息（这是struts的登录拦截，因为springmvc也存在，就利用session共享了登录信息）。

![struts拦截器](https://github.com/7le/7le.github.io/raw/master/image/threadLocal/threadLocal-3.png)

其实扒到这里，八成也能定位出问题了。前面提到过ThreadLocal是用线程为键，那当线程被复用的时候，就会有问题。现在一般使用的web容器，大多都是使用线程池的。我使用的是Tomcat，当一个请求进入时，比如A登录使用了thread-1，struts拦截器为ThreadLocal添加了thread-1了变量为A。这时候B登录了使用了thread-2，同样ThreadLocal添加了thread-2了变量为B。因为线程池复用线程，当B去修改springmvc的接口，线程池为B的请求分配了thread-1来处理，那就出现了之前诡异的现象。所以需要另外在springmvc的拦截器中为ThreadLocal重新赋值。

![springmvc拦截器](https://github.com/7le/7le.github.io/raw/master/image/threadLocal/threadLocal-4.png)

在Web容器环境中使用ThreadLocal和本地环境中的ThreadLocal还是有很多差异的，使用起来要特别小心！既然使用ThreadLocal会有风险，那这里使用它的意义是为何。我个人的看法是为了松耦合，当然我们也可以引用redis之类来处理，也是不错的方案。

### Tomcat线程池配置

既然上面也提到了tomcant的线程池，那再写一些关于tomcant线程池的配置。
首先打开/conf/server.xml文件,然后配置参数：

```
<Executor name="tomcatThreadPool"   
        namePrefix="tomcatThreadPool-"   
        maxThreads="500"    
        minSpareThreads="400"/>  
```
>name：共享线程池的名字。这是Connector为了共享线程池要引用的名字，该名字必须唯一。默认值：None；
namePrefix:在JVM上，每个运行线程都可以有一个name 字符串。这一属性为线程池中每个线程的name字符串设置了一个前缀，Tomcat将把线程号追加到这一前缀的后面。默认值：tomcat-exec-；
maxThreads：该线程池可以容纳的最大线程数。默认值：200；
maxIdleTime：在Tomcat关闭一个空闲线程之前，允许空闲线程持续的时间(以毫秒为单位)。只有当前活跃的线程数大于minSpareThread的值，才会关闭空闲线程。默认值：60000(一分钟)。
minSpareThreads：Tomcat应该始终打开的最小不活跃线程数。默认值：25。

然后配置

```
<Connector executor="tomcatThreadPool"  
           port="8080" protocol="HTTP/1.1"  
               connectionTimeout="20000"
               enableLookups="false"  
               redirectPort="8443"   
           minProcessors="5"  
           maxProcessors="75"  
           acceptCount="1000"/>  
```
>executor：表示使用该参数值对应的线程池；
minProcessors：服务器启动时创建的处理请求的线程数；
maxProcessors：最大可以创建的处理请求的线程数；
acceptCount：指定当所有可以使用的处理请求的线程数都被使用时，可以放到处理队列中的请求数，超过这个数的请求将不予处理。

再贴个jvm配置：
```
set JAVA_OPTS=-Xms1024m -Xmx1024m -XX:PermSize=128M -XX:MaxPermSize=256m
```

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)