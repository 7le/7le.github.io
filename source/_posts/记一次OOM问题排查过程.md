---
title: 记一次OOM问题排查过程
date: 2019-11-13 22:52:12
type: "tags"
tags: [OOM]
---

> 这周生产环境出了久违的``java.lang.OutOfMemoryError: Java heap space``，花了些时间找了下原因，记录分享一下 🥕

<!--more-->

### OOM分析

之前的jvm优化的文章中，有提到要增加``-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/logs/login/gc``，这对于排查问题非常有帮助，当发生OOM时就会输出堆的内存快照（hprof文件）。

从生产环境拷贝出hprof文件后，就可以用MAT或者Jvisualvm来分析了，相对来说MAT更智能和易用一些。

用MAT打开hprof文件后，很明显可以看到（这里补充一个小插曲：在jdk11的环境下，MAT会报错无法打开，建议切回jdk8的环境打开MAT）

通过**Leak Suspects**其实已经可以看出问题所在了，``SessionFactoryImpl``这个实例占用了87.48%的内存。很明显是内存泄漏了。

![MAT-1](https://github.com/7le/7le.github.io/raw/master/image/oom/MAT-1.PNG)

我们打开**Histogram**，可以看到类的实例数和所占的内存

![MAT-1](https://github.com/7le/7le.github.io/raw/master/image/oom/MAT-2.PNG)

这个看起来还并不是很清晰，可以排除软，弱，虚引用（剩下强引用）看的更加直观

![MAT-1](https://github.com/7le/7le.github.io/raw/master/image/oom/MAT-3.PNG)

现在就很明显了，``SessionFactoryImpl``中的``QueryPlanCache``造成了内存泄漏

![MAT-1](https://github.com/7le/7le.github.io/raw/master/image/oom/MAT-4.PNG)

### QueryPlanCache内存泄漏解决

那什么是``QueryPlanCache``呢，**hibernate**中的``QueryPlanCache``会缓存sql，以便于后边的相同的sql重复编译。

如果in后的参数不同，**hibernate**会把其当成不同的sql进行缓存，从而缓存大量的sql导致heap内存溢出。

所以可以通过设置缓存最大值来进行限制，不设置默认是2G。

```
spring:
  jpa:
    properties:
      hibernate:
        query:
          plan_cache_max_size: 64
          plan_parameter_metadata_max_size: 32
```


---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
