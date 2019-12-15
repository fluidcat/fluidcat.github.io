---
layout:     post
title:      Spring与SpringMVC的容器关系分析
subtitle:   Spring容器允许嵌套，所以应用常常中存在多个Spring容器
date:       2018-04-11
author:     Belin
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Spring
    - 源码分析
	- Java
---

> 正所谓前人栽树，后人乘凉。
>
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板

在一个项目中，容器不一定只有一个，Spring中可以包括多个容器，而且容器可以有上下层关系，这个关系可以是开发者自定义，也可以是框架预定义好的。

**Spring和SpringMVC这两个框架容器**
1. 其实就是2个容器，Spring是根容器，SpringMVC是其子容器，并且在Spring根容器中对于SpringMVC容器中的Bean是不可见的，而在SpringMVC容器中对于Spring根容器中的Bean是可见的，也就是子容器可以看见父容器中的注册的Bean，反之就不行。对于开发者来说必须确定使用的bean在哪个容器中，通过容器的xml配置文件可以判断bean所在的容器：
1、在哪个容器配置文件定义的bean就是在哪个容器中，这个简单易懂显而易见；
2、通过注解注入的bean，需要从xml配置的注解扫描范围来区分，一般来说不同容器的包扫描范围是不同的，在容器扫描内的bean就在该容器中。

**案例：**以常见的找不到controller为例：

Spring配置文件applicationContext.xml，SpringMVC配置文件applicationContext-MVC.xml，这样项目中就有2个容器了，如下：
配置方式A
applicationContext.xml中配置了
```
<!-- 负责所有需要注册的Bean的扫描工作 -->
<context:component-scan base-package=“com.test" />
```
applicationContext-MVC.xml中配置
```
<!-- 负责springMVC相关注解的使用 -->
<mvc:annotation-driven />
```
启动项目发现，springMVC失效，无法进行跳转，开启log的DEBUG级别进行调试，发现springMVC容器中的请求好像没有映射到具体controller中；

配置方式B
为了快速验证效果，将
````<context:component-scan base-package=“com.test" />````
扫描配置到applicationContext-MVC.xml中，重启后，验证成功，springMVC跳转有效。

**解析：**
springMVC初始化时，会寻找所有当前容器中的所有@Controller注解的Bean，来确定其是否是一个handler，而当前容器springMVC中注册的Bean中并没有@Controller注解的。注意上面案例中的配置方式A，所有的@Controller配置的Bean都注册在Spring这个父容器中了。看代码

![initHandlerMethods](/md_image/截图.png "initHandlerMethods")
在方法isHandler中会判断当前bean的注解是否是controller，代码如下：
```
protected boolean isHandler(Class<?> beanType) {
        return AnnotationUtils.findAnnotation(beanType, Controller.class) != null;
}
```
在配置方式B中，springMVC容器中包括了所有的@Controller注解的Bean，所以自然就能找到了。

------------
以上是原因，解决办法是什么？注意看initHandlerMethods()方法中，detectHandlerMethodsInAncestorContexts这个开关，它主要控制从那里获取容器中的bean，是否包括父容器，默认是不包括的。所以解决办法是有的，即在springMVC的配置文件中配置HandlerMapping的detectHandlerMethodsInAncestorContexts属性为true即可（这里需要根据具体项目看使用的是哪种HandlerMapping），让其检测父容器的bean。如下：
```
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping">
	<property name="detectHandlerMethodsInAncestorContexts">
		<value>true</value>
	</property>
</bean>
```
以上已经有了2种解决方案了，但在实际工程中，会包括很多配置，根据不同的业务模块来划分，所以我们一般思路是各负其责，明确边界，Spring根容器负责所有其他非controller的Bean的注册，而SpringMVC只负责controller相关的Bean的注册。第三种方案如下：

Spring容器配置，排除所有@controller的Bean
```
<context:component-scan base-package="com.test">
    <context:exclude-filter type="annotation" 
		expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```
SpringMVC容器配置，让其只包括@controller的Bean
```
<context:component-scan base-package="com.test" use-default-filters="false">
    <context:include-filter type="annotation"
		expression="org.springframework.stereotype.Controller" />
</context:component-scan>
```