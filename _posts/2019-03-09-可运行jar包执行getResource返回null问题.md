---
layout:     post
title:      可运行jar包执行getResource返回null问题
subtitle:   SpringBoot中存在多个Filter时如何控制它们的执行顺序
date:       2019-03-09
author:     Belin
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - SpringBoot
    - 获取资源
    - ClassLoader
---

> 正所谓前人栽树，后人乘凉。
>
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板

###1. 场景
起因：springboot项目打出可运行jar包，为减小jar体积方便更新，需要分离lib。
过程：使用其他打包插件替换springboot的打包插件，顺便贴代码
```xml
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-jar-plugin</artifactId>
	<configuration>
		<archive>
			<manifest>
				<addClasspath>true</addClasspath>
				<classpathPrefix>lib/</classpathPrefix>
				<mainClass>xxx.main</mainClass>
			</manifest>
		</archive>
	</configuration>
</plugin>
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-dependency-plugin</artifactId>
	<executions>
		<execution>
			<id>copy-dependencies</id>
			<phase>prepare-package</phase>
			<goals>
				<goal>copy-dependencies</goal>
			</goals>
			<configuration>
				<outputDirectory>
					${project.build.directory}/lib
				</outputDirectory>
				<overWriteReleases>false</overWriteReleases>
				<overWriteSnapshots>false</overWriteSnapshots>
				<overWriteIfNewer>true</overWriteIfNewer>
			</configuration>
		</execution>
	</executions>
</plugin>
```
由此得到：
![](/files/md_image/1.png)

命令行运行
```shell
java -jar TestDemo.jar
```
出现异常:
```java
org.springframework.beans.factory.BeanInitializationException: Could not load properties; nested exception is java.io.FileNotFoundException: class path resource [] cannot be resolved to URL because it does not exist
```
找到异常代码：
```java
ResourceUtils.getURL("classpath:").getPath()
```
郁闷至极，不就是把lib包分离出来吗？代码正常启动说明lib包和classes的类都在classpath中，然而缺获取不到类根路径...
###2. 问题原因
经过多次调试、google终于发现问题所在，下面是问题原因：
首先上面代码`ResourceUtils.getURL("classpath:")`等价于`classLoader.getResource("")`,其中classLoader要根据环境而定不同环境可能不同；
其次要理解`getResource`的含义：
> getResource：主要是获取资源,这个资源并不仅限于文件，还有文件夹。

最后，通过以下代码
```java
String jarName = "xxxx.jar";
        JarFile jarFile = null;
        try {
            jarFile = new JarFile(jarName);
        } catch (IOException e) {
            e.printStackTrace();
        }
        Enumeration<JarEntry> entrys = jarFile.entries();
        while (entrys.hasMoreElements()) {
            JarEntry jarEntry = entrys.nextElement();
            System.out.println(jarEntry.getName());
        }
```
会发现，jarentry中并没`""`这个目录资源（这是肯定的，都是空字符串了），所以前面的异常代码就抛异常了。

那么问题来了，为什么springboot的打包插件又能正常运行？
比较springboot打包的jar和上面打包的jar，发现jar包的目录结构有所差别：
- 1、上面打包的jar包根路径直接是classes，即类路径的根路径
- 2、springboot打包插件打包的jar是BOOT-INF、META-INF、org，此时类路径的根路径是BOOT-INF/classes

这个根路径的差别与ClassLoader有关，第一的类型是`APPClassLoader`, 第二的类型是`LaunchedURLClassLoader`具体还是google下吧。第二情况的时候`getResource("")`等价于 `getResource("BOOT-INF/classes")`,所以springboot打包插件打包的jar能获得根路径。

### 3. 解决办法
重新配置打包插件，仍然使用springboot打包插件打包，如下
```xml
<plugin>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-maven-plugin</artifactId>
	<configuration>
	<!--<executable>true</executable>-->
		<layout>ZIP</layout>
		<includes>
			<include>
				<groupId>nothing</groupId>
				<artifactId>nothing</artifactId>
			</include>
		</includes>
	</configuration>
</plugin>
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-jar-plugin</artifactId>
	<configuration>
		<archive>
			<manifest>
				<addClasspath>true</addClasspath>
				<classpathPrefix>lib/</classpathPrefix>
				<mainClass>xxx.main</mainClass>
			</manifest>
		</archive>
	</configuration>
</plugin>
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-dependency-plugin</artifactId>
	<executions>
		<execution>
			<id>copy-dependencies</id>
			<phase>prepare-package</phase>
			<goals>
				<goal>copy-dependencies</goal>
			</goals>
			<configuration>
				<outputDirectory>
					${project.build.directory}/lib
				</outputDirectory>
				<overWriteReleases>false</overWriteReleases>
				<overWriteSnapshots>false</overWriteSnapshots>
				<overWriteIfNewer>true</overWriteIfNewer>
			</configuration>
		</execution>
	</executions>
</plugin>
```
### 4. 问题复现最小demo
![](/files/md_image/demo.png)

**注意maven打包仍是最前面的配置**
### 5. 问题相关知识
1、类加载机制
2、jar包目录结构
3、add directory entries （这里指eclipse导出jar包常见的问题）