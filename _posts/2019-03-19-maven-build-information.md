---
layout: post
title: maven打包时将打包信息写入build.properties
date: 2019-03-19
categories:
    - maven
comments: true
permalink: maven-build-information.html
---

1. 设置日期格式化

```
<properties>
	<timestamp>${maven.build.timestamp}</timestamp>
	<maven.build.timestamp.format>yyyy-MM-dd HH:mm</maven.build.timestamp.format>
</properties>
```

2. 增加build配置

```
<build>
	<resources>
		<resource>
			<directory>src/main/resources</directory>
			<filtering>true</filtering>
		</resource>
	</resources>
</build>
```

3. 增加build.properties

```
build.version=${pom.version}
build.date=${timestamp}
build.url=${pom.url}
build.name=${pom.name}
```
5. 运行 `mvn clean package`，查看打包文件中的build.properties

```
build.version=1.0-SNAPSHOT
build.date=2019-03-19 13:09
build.url=${pom.url}#我没有设置这个属性
build.name=maven-build-information
```
Maven自带时间戳使用`${maven.build.timestamp}`，但是时区是UTC。我们可以使用`buildnumber-maven-plugin`，会增加一个`timestamp`属性

```
<plugin>
	<groupId>org.codehaus.mojo</groupId>
	<artifactId>buildnumber-maven-plugin</artifactId>
	<version>1.4</version>
	<configuration>
	  <timestampFormat>yyyy-MM-dd HH:mm</timestampFormat>
	</configuration>
	<executions>
	  <execution>
		<goals>
		  <goal>create-timestamp</goal>
		</goals>
	  </execution>
	</executions>
	<inherited>false</inherited>
</plugin>
```