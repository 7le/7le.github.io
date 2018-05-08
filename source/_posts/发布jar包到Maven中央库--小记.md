---
title: '发布jar包到Maven中央库--小记'
date: 2017-12-17 21:29:18
type: "tags"
tags: [maven]
---

> 最近项目选用了vert.x，故此简单的封装了一下，好达到从spring体系到vert.x平滑过度，减少其他人的学习成本，而后发布到Maven中央库，记录一下~

<!--more-->

### create issues

首先需要在[sonatype](https://issues.sonatype.org/secure/Dashboard.jspa)中创建issues，没有帐号的需要注册一下（帐号密码后续会有用到）。

类似如图：![issues](http://oqipguzbl.bkt.clouddn.com/maven-issus.png)

这里有个注意的点，这个group Id需要自己有个域名或者使用github的，类似io.github.7le or com.github.7le。这里我自己买了个域名，价格动人。当issue的Status变为RESOLVED，就代表已经通过了，如果有什么问题，会有留言告诉你如何修改。

### gpg & maven

Windows的话可以到[gpg](https://www.gpg4win.org/download.html)下载gpg4win。Linux可以通过yum install gpg命令安装gpg。

我们可以通过``gpg --version``,来查看安装的gpg
![gpg-version](http://oqipguzbl.bkt.clouddn.com/maven-gpg-version.png)

执行``gpg --gen-key`` 按照如下的命令设置好你的名字，邮箱。不过输入Passphrase的值需要记住，这个相当于密钥的密码

然后设置maven的配置，在``setting.xml``
需要设置
```
<servers>
  <server>
    <id>sonatype-nexus-snapshots</id>
    <username>Sonatype网站的账号</username>
    <password>Sonatype网站的密码</password>
  </server>
  <server>
    <id>sonatype-nexus-staging</id>
    <username>Sonatype网站的账号</username>
    <password>Sonatype网站的密码</password>
  </server>
</servers>
```
以及
```
<profile>
	<id>shine</id>
	<distributionManagement>
        <snapshotRepository>
            <id>oss</id>
            <url>https://oss.sonatype.org/content/repositories/snapshots/</url>
        </snapshotRepository>
        <repository>
            <id>oss</id>
            <url>https://oss.sonatype.org/service/local/staging/deploy/maven2/</url>
        </repository>
    </distributionManagement>
</profile>
```

在自己的``pom.xml``中添加
```
<parent>
    <groupId>org.sonatype.oss</groupId>
    <artifactId>oss-parent</artifactId>
    <version>7</version>
</parent>

....

<licenses>
    <license>
        <name>The Apache Software License, Version 2.0</name>
        <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
        <distribution>repo</distribution>
    </license>
</licenses>
<scm>
    <tag>master</tag>
    <url>https://github.com/7le/vertx-shine</url>
    <connection>scm:git:https://github.com/7le/vertx-shine.git</connection>
    <developerConnection>scm:git:https://github.com/7le/vertx-shine.git</developerConnection>
</scm>
<developers>
    <developer>
        <name>7le</name>
        <email>silk.heqian@gmail.com</email>
        <organization>ArkStack</organization>
        <organizationUrl>7le.top</organizationUrl>
    </developer>
</developers>
```

### deploy & release 

一切就绪后，在命令行输入
``mvn clean deploy -P sonatype-oss-release -Darguments="gpg.passphrase=设置的Passphrase"``
![operation](http://oqipguzbl.bkt.clouddn.com/maven-operation.jpg)

成功之后可以到[Neuxs](https://oss.sonatype.org/index.html#stagingRepositories)查看发布好的jar包：
![manager](http://oqipguzbl.bkt.clouddn.com/maven-manager.jpg)

按着步骤，先close 再release。![manager2](http://oqipguzbl.bkt.clouddn.com/maven-manager2.png)

这里在close的会报一个错误，错误如下
```
typeId	signature-staging
failureMessage	No public key: Key with id: (your_id) was not able to be located on http://pgp.mit.edu:11371/. Upload your public key and try the operation again.
failureMessage	No public key: Key with id: (your_id) was not able to be located on http://keyserver.ubuntu.com:11371/. Upload your public key and try the operation again.
failureMessage	No public key: Key with id: (your_id) was not able to be located on http://pool.sks-keyservers.net:11371/. Upload your public key and try the operation again.
failureMessage	No public key: Key with id: (your_id) was not able to be located on http://pgp.mit.edu:11371/. Upload your public key and try the operation again.
.....
```
这里将your_id，上传到第三方的key验证库，如下
![gpg-addKey](http://oqipguzbl.bkt.clouddn.com/maven-gpg-addKey.png)

然后再等一段时间，就可以在[Maven中央仓库](http://search.maven.org)搜到了。正式版本和SNAPSHOT大家可以自己选择~

![search](http://oqipguzbl.bkt.clouddn.com/maven-search.png)

如果你也需要，可以戳[vertx-shine-web](https://github.com/7le/vertx-shine)

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](http://7le.top)