---
layout:     post
title:      SpringAop 源码分析与总结
subtitle:   SpringAop 源码分析与总结
date:       2020-03-26
author:     Belin
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Java
    - AOP
    - 源码
---

> 正所谓前人栽树，后人乘凉。
> 
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板

## 简述
面向切面编程，也称AOP编程，已经是耳熟能详了吧。相比与OPP、OOP的编程思想，AOP是更高级的编程思想，是在OOP的基础上延生出来的思想，是对OOP的完善和补充。
AOP能完成OOP难以完成的横向的、跨业务的设计，在一定程度上降低不同功能模块之间的耦合度。比如日志功能与其他功能的关系。

## AOP需要什么
AOP并没有标准规范，实现方式不尽相同，便捷实用一般都具有两部分组成：
- 切面语法：用于定义切面，以静态方式定义好切面，也可以说是代码埋点（方式：约定命名规则、路径寻找、代码标记、注解、切面声明文件等）
- 代码织入：如何往切面切入点注入代码（Java为例：静态织入（编译时织入，编译后织入）、动态织入（类加载时织入LWT、运行时织入））

## AOP实现
Java现在比较流行的是AspectJ与SpringAop。  

AspectJ具有一套完整的AOP的框架。
切面语法上有两种：
- 使用注解的方式定义切面，如@PointPut；
- 使用aspect文件声明，将切面定义在文件中；

织入方式有三种：
- 编译时织入，使用ajc编译器对进行编译，在编译时把修改切面处代码并生成class，将代码织入切面;
- 编译后织入，在javac对源码编译后，在使用编译器ajc编译器进行处理，织入代码，与第一种效果一样；
- 加载时织入，利用aspectjweaver.jar工具，使用Java探针（-javaagent:）在类加载时修改字节码进行代码织入；

SpringAop并不重复造轮子，结合了AspectJ和代理实现是AOP 
切面语法：直接使用AspectJ的注解。这么做好处在于兼容了AspectJ,如果开发者想使用AspectJ的织入方式，直接使用ajc编译器进行编译或者运行时添加-javaagent即可
织入方式：使用运行时织入的方式，有两种实现：
- JDK动态代理的方式：使用JdkDynamicAopProxy类代理所有连接点对象，并在invoke方法中进行代码织入
- CGLIB生成子类方式：使用ObjenesisCglibAopProxy类生成代理类的子类，并在MethodInterceptor#intercept中进行代码织入


## springAOP源码流程  
spring全家桶在发展流程中，springboot绝对是一个里程碑式的项目，对项目管理、架构有着深远的影响。然而之前广泛流行的xml配置在当时也是非常“正确”的选择（`配置即代码`、`功能与实现彻底解耦`），
随便探讨一下xml的实现流程也有助于加深对aop的理解。
> 曾经有段时间Java的生态圈走了极端，为了彻底的解耦，把“什么时候需要什么样的实现”这个问题的解决方案，写在配置文件（也就是xml里），因为xml不需要重新编译，因此达成了“彻底解耦”这个目标（我需要什么实现我就改配置文件就行了，不需要改代码，不需要重新编译），然而随着时间的发展，人们发现了以下问题，1.大部分时候我们并不需要解耦的如此彻底，应用级程序甚至很少遇到要到要换接口实现。2，xml的配置学习起来门槛太高了，就算老手看一个配置文件经常也能看的云里雾里，再加上xml文件不能做类型检查，有时候你实际上是把一个错误的类装配进接口，导致系统报错，你却要找半天错误。因此这几年提出了新的口号叫“约定大于配置”，javaconfig实际上就是约定大于配置的产物，它做的事情和xml配置几乎相同，但是因为有类型检查且本身就是java代码的原因，它的可读性和可维护性远高于xml配置，新手接手项目的门槛大大降低。而代价不过是放弃那天知道什么时候才可能用的上“彻底解耦”的能力。

### springBoot流程
首先从配置入手，找到aop得自动配置类,这个简单，直接翻看IDE中的maven依赖`spring-boot-autoconfigure`，找到`AopAutoConfiguration`类
```java
@Configuration(proxyBeanMethods = false)
// aop开关配置,默认true（开启） spring.aop.auto=true
@ConditionalOnProperty(prefix = "spring.aop", name = "auto", havingValue = "true", matchIfMissing = true)
public class AopAutoConfiguration {

    // aop配置，Advice.class为AspectJ包的类，依赖AspectJ的包
    // 通过配置spring.aop.proxy-target-class=false/true 决定使用jdk代理还是cglib代理，默认使用cglib
    // @EnableAspectJAutoProxy注解开启aop解析，主要是AspectJ切面语法解析并通过代理模式进行织入
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(Advice.class)
	static class AspectJAutoProxyingConfiguration {

		@Configuration(proxyBeanMethods = false)
		@EnableAspectJAutoProxy(proxyTargetClass = false)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false", matchIfMissing = false)
		static class JdkDynamicAutoProxyConfiguration {}

		@Configuration(proxyBeanMethods = false)
		@EnableAspectJAutoProxy(proxyTargetClass = true)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
		static class CglibAutoProxyConfiguration {}
	}
    // 默认开启基础aop功能，如事务中用到的aop，可在不引入AspectJ依赖情况下使用
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingClass("org.aspectj.weaver.Advice")
	@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
	static class ClassProxyingConfiguration {
		ClassProxyingConfiguration(BeanFactory beanFactory) {
			if (beanFactory instanceof BeanDefinitionRegistry) {
				BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
				AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
		}
	}
}
```

以上代码片段，主要关注`@EnableAspectJAutoProxy`,AOP支持的关键。在其源码中看到
```
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {...}
```
导入了一个`ImportBeanDefinitionRegistrar`,一个在解析BeanDefinition时注册依赖额外bean的拓展类，在其源码看到主要通过`AopConfigUtils`注册了一个bean后置处理器`BeanPostProcessor`
并传递`EnableAspectJAutoProxy`上的配置信息，代码比较简单不贴了.
`BeanPostProcessor`的类关系如下：  
```
AnnotationAwareAspectJAutoProxyCreator
               ↓
AspectJAwareAdvisorAutoProxyCreator
               ↓
AbstractAdvisorAutoProxyCreator
               ↓
AbstractAutoProxyCreator
               ↓
SmartInstantiationAwareBeanPostProcessor
```  
BeanPostProcessor主要作用是在bean创建后可以对bean进行修改，springAop就是在这里使用代理对象代理刚刚创建的bean，对源bean进行访问控制实现代码织入，
这就是springAOP能够将代码织入延迟到bean创建时间点的关键。继续看代码上是怎么实现的。  
既然是`BeanPostProcessor`直奔其方法`postProcessBeforeInitialization`、`postProcessAfterInitialization`，
在`postProcessAfterInitialization`中看到：
```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    /* skip some code */
    
    // 根据bean的class获取到相应的通知，就是织入的代码
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 使用原bean创建代理
        Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```
在createProxy方法中，使用`ProxyFactory`创建代理对象，这其中封装了JDK代理和CGLiB代理的切换，调用关系如下：
```
AbstractAutoProxyCreator#createProxy
            ↓
Object ProxyFactory#getProxy
                ↓
    AopProxy ProxyCreatorSupport#createAopProxy
                    ↓
        AopProxyFactory ProxyCreatorSupport#getAopProxyFactory
                ↓
    AopProxy DefaultAopProxyFactory#createAopProxy
            ↓
Object JdkDynamicAopProxy#getProxy || Object CglibAopProxy#getProxy
```
这只需知道最终通过Jdk代理或CGLIB代理创建代理对象就可以了，到这里基本上流程差不多了，最终返回了代理对象给容器。

### xml配置流程  
以以往的经验，对xml解析是通过命名空间处理器处理`NamespaceHandler`的。果不其然在springAop的依赖spring-aop-5.2.4.RELEASE.jar中找到一个文件`spring.handlers`：
```properties
http\://www.springframework.org/schema/aop=org.springframework.aop.config.AopNamespaceHandler

```
可以看到xml的解析工作交给了`AopNamespaceHandler`。在处理其中注册了许多解析器
```java
public class AopNamespaceHandler extends NamespaceHandlerSupport {
	@Override
	public void init() {
		// In 2.0 XSD as well as in 2.1 XSD.
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
		// Only in 2.0 XSD: moved to context namespace as of 2.1
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}
}
```
在parser中可以看到同样向容器注册了`AnnotationAwareAspectJAutoProxyCreator`,这意味着接下来的工作和springboot差不多了。
主要区别还是在生成切面时是解析xml标签还是Java注解。