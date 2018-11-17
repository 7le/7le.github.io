---
layout: posts
title: '玩耍Devops Git+Gogs+Jenkins+Docker'
date: 2017-10-09 22:37:30
tags: Devops
---

> 最近受人熏陶，准备尝试一下自己搭建一套自己的Devops，故此记录一番。

<!--more-->

首先，我们要了解Devops（Development和Operations），可以理解为是一个完整的面向IT运维的工作流，以 IT 自动化以及CI,CD为基础，来优化程式开发、测试、系统运维等所有环节。其中持续集成（Continuous integration，简称CI），指的是频繁地（一天多次）将代码集成到主干。而CD，这个概念就容易引起混淆了，因为CD这个缩写代表了两个短语，一个是Continuous Delivery，一个是Continuous Deployment，两者的区别就是，部署到生产环境这一步骤，是手工的，还是自动的。

详细的可以看[持续集成是什么？](http://www.ruanyifeng.com/blog/2015/09/continuous-integration.html)。

当我们大概了解Devops之后，会发现互联网软件的开发和发布是一套相对繁琐的流程，而我们可以通过诸多工具，来进行简化，方便操作，达到让产品可以快速迭代，同时还能保持高质量。

PS：文章有点长，记录了整个环境的搭建的细节，以及踩的坑，用的服务器是Centos6.5，如果有什么问题，欢迎拍砖讨论。

### Git+Gogs

> git 大家早已耳熟能详，而gogs是基于Go语言的自助Git服务,图形化界面跟github非常相似。

一般要搭建自己的git仓库，现在的选择大多都是Gitlab或者Gogs。两者分别有各自的优势，考虑自己可怜的服务器配置，毫无疑问我的选择会是更加轻量的Gogs。

#### Git

Git一般大家的服务器都会已经安装，可以通过 ``git --version`` 查看。如果版本过低，最好升级下，我原先是1.7.*的版本的，后来出现了点问题。

##### 下载

``cd``到自己服务器存放包的路径下，执行以下命令

```
wget https://github.com/git/git/archive/v2.3.0.zip
```

##### 解压

因为下载的是个zip包，``unzip v2.3.0 -d git `` 进行解压

##### 编译安装

进入git目录，将其安装在“/usr/local/git”目录下,命令如下：
```
make prefix=/usr/local/git all
sudo make prefix=/usr/local/git install
```

之后编辑``/etc/profile``此文件,在最下边添加git的路径：``export PATH=/usr/local/git/bin:$PATH``

运行``source /etc/profile``

最后检验``git --version``

#### Gogs

##### 下载

Gogs 在Github是开源的，可以在[Gogs版本下载地址](https://github.com/gogits/gogs/releases) 下载自己需要的版本。我下载的是最新版本v0.11.29的linux_amd64.tar.gz。

##### 解压
```
$ tar -xzvf linux_amd64.tar.gz
```

##### 安装

进入到刚刚解压后的目录执行命令``./gogs web``
然后在浏览器输入``http://ip:3000``
会出现如下图片：![Gogs配置图片](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-1.jpg)
根据实际情况配置即可。配置后就可以使用了，可以注册自己的帐号，操作跟Github如出一辙。

经过一番折腾后，![silk's Gogs](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-2.png)

这样就完成了第一步，搭建Git+Gogs。
PS：后台运行命令 
```
nohup ./gogs web > nohup.out 2>&1 &
```
以及官方文档的[版本升级](https://gogs.io/docs/upgrade/upgrade_from_binary)

### Jenkins

#### 安装Jenkins

安装完Git和Gogs之后,接下来就是我们的主角jenkins了。jenkins安装比较简单，
执行命令方可完成。
```
yum install jenkins
```

安装之后启动：
```
service jenkins start
```

如果需要修改端口，路径在``/etc/sysconfig/jenkins``，修改``JENKINS_PORT="your_port"``。重启jenkins服务。

进入的界面如下，表示启动成功

![jenkis成功登录](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-3.png)

PS:首次登录，是没有右边的任务，那是我已经添加了的。

#### Jenkins 插件

登录都OK之后，就需要为jenkins下载插件了。

因为我们是通过git去拉取代码，需要安装，一般是自带了，如果没有就安装一下，非常简便。

![jenkis插件](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-4.png)

其中``Gogs plugin``也需要安装，后续的hook会用到。还要安装``Pipeline``，这个是持续交付的重头戏，以及``Publish Over SSH`` 这个是用来远程部署的。

#### Jenkins 配置

配置只需要按照自己的服务器的实际情况填写，配置的地方为``系统管理``->``Global Tool Configuration``

##### Jdk

![jdk](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-5.png)

##### Git

![git](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-6.png)

##### Maven

![maven](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-7.png)

如果还没安装maven，就下载放到服务器。我用的是``apache-maven-3.3.9-bin.tar.gz``

解压后 在``/etc/profile``中添加
```
export M2_HOME=/software/apache-maven-3.3.9
export PATH=$PATH:$JAVA_HOME/bin:$M2_HOME/bin
```

基本配置就差不多了，建议注册帐号，提高安全性也方便后续操作。

我的Configure Global Security是这样配置的，这个按照自己的需要，如果像我这样配置，就需要登录帐号。
![Configure Global Security](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-8.png)

#### 创建Job

前面的操作都完成之后，点击``My views`` --> ``新建``

![新建job](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-9.png)

##### Maven project

先记录使用maven实现，输入项目名称，选择maven项目按钮。

源码管理选择git，填写自己gogs的URL。例如我的video项目。

![gogs-video](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-10.png)

对应填写相关参数：
![maven-project1](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-11.png)

构建触发器，就是什么时候执行jenkins的自动化部署，选择第一个，其他的基本是定时执行什么的，有兴趣都可以去尝试一下

![maven-project2](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-12.png)

Build和Post Steps设置:

![maven-project3](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-13.png)

Execute shell 脚本(带了注解，放入配置的时候可以把注释去掉)
```
#!/bin/bash

export JAVA_HOME=/usr/local/jdk1.8.0_73
#export BUILD_ID=dontKillMe这一句很重要，因为该job启动完后执行下一job,jenkins直接把tomcat进程杀了
export BUILD_ID=dontKillMe

project=video-0.0.1.jar
#jenkins的路径
file_path=/var/lib/jenkins/workspace/video/target

sleep 3s 
# 查看服务是否已经启动，启动了就先kill
pid=$(lsof -i:9000 |awk '{print $2}' | tail -n 1)

if [ "$pid" != "" ] 
echo "close old video"
then
echo $pid
kill -9 $pid
fi

echo "begin new video"

cd $file_path
# springboot 后台部署方式
java -jar ${project} &
```

邮件设置：

![maven-project4](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-14.png)

设置完成后，点击保存。跳转到新的页面，点击页面上的``立即构造``就会开始运行了。

![maven-project5](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-15.png)

运行的时候可以点击``console Output``查看日志。

等到小太阳出现，查看下自己的项目，已经启动了，那就大功搞成拉~ 如果很悲剧的出现问题了，那可以看看下面的踩坑记录中的东西能不能帮到你了~

![maven-project6](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-16.png)

##### pipeline

pipeline是jenkins2.0的一大亮点，在新建任务的时候选择Pipeline。

pipeline设置：
![pipeline](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-17.png)

```
node {  

    stage('checkout'){
        echo 'checkout from git'
        git url: 'http://114.215.122.158:3000/silk_gogs/video.git', branch: 'master'
    }
    stage('Test') {            
        echo "Test start..."
        sh 'mvn clean '
    }        
    stage('Build') {            
        echo "Deploy start..."
        sh ' mvn package -Dmaven.test.skip=true'
        sh ' sh /root/video-pipeline.sh'
        
    }
}
```
pipeline 采用的是Groovy语法，
1. Stage：表示当前进行的阶段，名字可以随意去，如：git、build、test、build等
2. Node:后面接一个小括号和一个大括号，小括号指定节点的名字或标签（缺省时随机选择节点），大括号表示在该节点运行的内容
3. Step:像git、echo、sh、archiveArtifacts等这些指令，一个指令就是一个step

Groovy脚本中调用的shell脚本,跟maven项目的差不多。
```
#!/bin/bash

export JAVA_HOME=/usr/local/jdk1.8.0_73
export BUILD_ID=dontKillMe

project=video-0.0.1.jar
file_path=/var/lib/jenkins/workspace/video-pipeline/target

sleep 3s 

pid=$(lsof -i:9000 |awk '{print $2}' | tail -n 1)

if [ "$pid" != "" ] 
then
echo "close old video"
echo $pid
kill -9 $pid
fi

echo "begin new video:$file_path/${project}"

nohup java -jar $file_path/${project} > nohup.out  2>&1 &

echo "end ^.^"
```

点击``立即构造``：
![pipeline](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-18.png)

跟maven project不同的是多了许多的Stage块，非常直观，而且支持复杂的CD(Continuous Delivery)需求,还有许多有趣的点等有时间再去玩耍。

#### Gogs plugin

之前在安装插件的时候，特意安装了Gogs plugin，这个插件是干嘛的呢？我们之前的操作都是通过手动触发的，这个插件可以帮我们实现代码的push操作触发（还有其他的hook事件），点击这个插件进去可以看到

![Gogs plugin](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-19.png)

很明显的告诉我们，只需要在gogs的项目中配置webhook

![Gogs hook](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-20.png)

这样就可以实现代码``push``就触发整个流程了。

### Docker

> 待整理出来补充~

### 踩坑记录

#### Maven JVM terminated unexpectedly with exit code 137

遇到这个问题，原因是我的服务器配置比较差，jenkins还是吃配置的。选择一个比较好的服务器，就可以省去这个问题，如果跟我一样，服务器不太好的话。

解决的方法:

1. 在``/etc/profile`` 中添加``export _JAVA_OPTIONS="-Xmx500m"`` -Xmx是jvm的最大堆大小，默认值为物理内存的1/4,这边可以根据情况手动调大。

2. 执行Linux释放内存命令 
```
sync   #把内存数据同步到硬盘（有需要可以执行）
echo 3 > /proc/sys/vm/drop_caches  #删除内存缓存。
```

3. 分出swap。 在linux中输入命令
```
swapon -s #確認目前的swap大小
dd if=/dev/zero of=/swapfile bs=1024 count=1048576 #增加1G的空间
mkswap /swapfile  #格式化swapfile
swapon /swapfile  #启动swapfile
```

最后输入``free -m ``，查看内存及swap情况
![free -m](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-21.png)


这样一顿操作，八九不离十就能解决137的问题了，因为也很难找到比我服务器更差的机器了...

#### 权限问题

日志中出现 XXXXX permission denied。

如果是运行调用到其他文件，权限不够，可以``chown -R jenkins /xxx/xxx ``,最好的是设置用户组，这样的话比较简单粗暴。

但是我是maven install出来的jar包 没有权限运行，这不是坑我么。本来想在sh脚本中切到root用户的，尝试了几种方法都未果，于是就用了粗暴的方式...

直接修改``/etc/sysconfig/jenkins`` 中的``JENKINS_USER="root"`` ，再修改下权限
```
chown -R root:root /var/lib/jenkins
chown -R root:root /var/cache/jenkins
chown -R root:root /var/log/jenkins
```
最后记得修改后重启下jenkins

#### command not found
我们在项目中的日志报错，出现XXX command not found，但是服务器中明明已经是有相应的软件。这时候我们需要，在服务器中输入``echo $PATH``

将一长段环境变量复制到配置中，就OK了
![环境变量配置](https://github.com/7le/7le.github.io/raw/master/image/devops/Devops-22.png)

#### 项目无法后台运行

出现这个问题的时候，jenkins 的job显示是success的，但是查看自己的服务器，对应的项目死活起不来。经过一番google，在jenkins的官网中找到了答案。
[jenkins-ProcessTreeKiller](https://wiki.jenkins.io/display/JENKINS/ProcessTreeKiller),大概的意思就是说jenkins的特性会杀掉在任务中运行的生成的进程。``To reliably kill processes spawned by a job during a build`` 就是这句话。

那我们只需要关掉这个特性就可以了，同样修改``/etc/sysconfig/jenkins`` 中的``JENKINS_JAVA_OPTIONS="-Djava.awt.headless=true -Dhudson.util.ProcessTree.disable=true"`` 记得修改后重启下jenkins

罗里吧嗦了一堆，希望大家也都能顺利搭建起来~ 

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)