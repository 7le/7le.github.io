---
title: springboot项目笔记：(一)搭建项目
date: 2017-05-26 00:13:08
type: "tags"
tags: [springboot]
---

> springboot的评价一直很高，那就用它来做一次完整的项目，现在就一点点记录下来，可以以后自己翻看，也可以帮助各位第一次使用springboot的小伙伴们。 🥕

<!--more-->

既然说要完整的一个项目，那肯定不是说笑的拉。准备做一个视频点播的项目，比较简单没有特别大的难度，感觉唯一的难度就是截视频的图和转码了把，那些到时候一点点记录下来。不废话了，现在开始第一步~

### idea 创建项目
![idea-1](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-idea-1-1.png)

点击箭头的按钮，然后就是next。

![idea-2](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-idea-1-2.png)

这里面可以自己选择下name，和一些配置。

![idea-3](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-idea-1-3.png)

这个直接next，点点过去，后面可以自己配置的。next后有个选择路径自己选择下。

![idea-4](https://github.com/7le/7le.github.io/raw/master/image/springboot/springboot-idea-1-4.png)

最后就是等待之后的springboot项目拉~~~

### 添加依赖

然后就是在大家熟知的pom.xml里面进行添加依赖

这是我的依赖，之后会用到的基本都在这里
```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.shine</groupId>
	<artifactId>video</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>video</name>
	<description>shine-video</description>

	<!--
       spring boot 父节点依赖,
       引入这个之后相关的引入就不需要添加version配置，
       spring boot会自动选择最合适的版本进行添加。
     -->
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.2.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<springfox.version>2.6.1</springfox.version>
		<spring.redis>1.3.2.RELEASE</spring.redis>
	</properties>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-freemarker</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-jdbc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-aop</artifactId>
		</dependency>

		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>

		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-core</artifactId>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.datatype</groupId>
			<artifactId>jackson-datatype-joda</artifactId>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.module</groupId>
			<artifactId>jackson-module-parameter-names</artifactId>
		</dependency>

		<!--druid-->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid</artifactId>
			<version>1.0.11</version>
		</dependency>
		<!--fastJson-->
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>fastjson</artifactId>
			<version>1.2.30</version>
		</dependency>
		<!--mybatis-->
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>1.2.0</version>
		</dependency>

		<!--redis-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-redis</artifactId>
			<version>${spring.redis}</version>
		</dependency>

		<!--pagehelper-->
		<dependency>
			<groupId>com.github.pagehelper</groupId>
			<artifactId>pagehelper-spring-boot-starter</artifactId>
			<version>1.1.0</version>
		</dependency>

		<!-- 热部署 -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>springloaded</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<optional>true</optional>
		</dependency>

		<!--guava-->
		<dependency>
			<groupId>com.google.guava</groupId>
			<artifactId>guava</artifactId>
			<version>21.0</version>
		</dependency>

		<!--swagger-->
		<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger2</artifactId>
			<version>${springfox.version}</version>
			<exclusions>
				<exclusion>
					<groupId>com.google.guava</groupId>
					<artifactId>guava</artifactId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>io.springfox</groupId>
			<artifactId>springfox-swagger-ui</artifactId>
			<version>${springfox.version}</version>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-configuration-processor</artifactId>
			<optional>true</optional>
		</dependency>

	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>


</project>

```

### 热部署
上面的包，基本都加了注释了。很容易知晓哪个是对应什么，这里特别说下，关于springboot的热部署，

上面的pom中 有两种springboot热部署的方式。

```java
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-devtools</artifactId>
	<optional>true</optional>
</dependency>
```
这种方式的话，就需要修改代码后，ctrl+F9，这样就可以了。

想必大家平时开发的时候，都用过一些热部署插件，比如JRebel，配合上idea的Update classes and resources，敲代码简直要起飞。所以很难忍受时不时ctrl+F9一下，而另外一种就是可以这样。
```java
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>springloaded</artifactId>
</dependency>
```
这种方式的话：
添加以后，通过mvn spring-boot:run启动就支持热部署了。
　注意：使用热部署的时候，需要IDE编译类后才能生效，可以打开自动编译功能，这样在保存修改的时候，类就自动重新加载了。通过在IDEA下面的终端中运行mvn spring-boot:run命令

这样就可以搭建一个springboot的项目了。

---
[Github](https://github.com/7le) 不要吝啬你的star ^.^
[更多精彩 戳我](https://7le.top)