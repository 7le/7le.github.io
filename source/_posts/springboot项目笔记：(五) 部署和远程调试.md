---
title: springboot项目笔记：(五) 部署和远程调试
date: 2017-06-19 23:12:35
type: "tags"
tags: [springboot]
---

> springboot部署的方式非常简单好用，接下来介绍下如何在Linux下部署项目，以及如何进行远程调式。

<!--more-->
### 部署

在idea中，在命令行中输入mvn package
![打包](http://oqipguzbl.bkt.clouddn.com/springboot-5-1.png)

成功后会出现
![成功](http://oqipguzbl.bkt.clouddn.com/springboot-5-2.png)

然后我们只需要把jar包放到linux中，然后cd到目录下输入
```
nohup java -jar xxx.jar > /tmp/web.log.out 2>&1 &
```
就大功告成拉~

如果还需要增加jvm的参数和gc日志，也可以直接跟在后面，优化的参数根据实际情况来选择，

```
nohup java -jar -Xmx3072m -Xms3072m -Xmn1152m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:ParallelGCThreads=4 -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:/logs/gc/gc.log -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/logs/gc xxx.jar &
```

### 远程调试

远程调试可以帮我们检查一些在日志中判断不出来的bug，对于idea我们需要如图：
![idea远程调试](http://oqipguzbl.bkt.clouddn.com/springboot-5-3.png)

需要输入服务器的地址，和你开启用来远程调试的端口号。这个端口号如何开启呢？对于springboot的项目来说

``` 
// address 端口号可以自己设置 要跟remote中的port一样，还要注意不要跟linux已经使用的端口号冲突
java -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005 -jar XXX.jar &
```

然后如图所示就是成功远程调试了
![idea连接成功](http://oqipguzbl.bkt.clouddn.com/springboot-5-4.png)

补充一下，如果是用tomcat来部署的话，修改如下:
```
// bin\startup.bat（.sh）文件，在里面添加
 
// windows
set CATALINA_OPTS="-agentlib:jdwp=transport=dt_socket,address=8888（自定义调试端口）,server=y,suspend=n %CATALINA_OPTS%"
 
// linux
export CATALINA_OPTS="-agentlib:jdwp=transport=dt_socket,address=8888（自定义调试端口）,server=y,suspend=n $CATALINA_OPTS"

```
这里还有个注意的点，就是别加在最末尾，嵌套在指令里面，不然会失效。

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)
