---
layout:
title:      Filter执行顺序不可控制问题
subtitle:   SpringBoot中存在多个Filter时如何控制它们的执行顺序
date:       2019-12-14
author:     Belin、L
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - SpringBoot
    - Filter
    - Servlet 3.0
---

> 正所谓前人栽树，后人乘凉。
>
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板

# 1、问题来源
使用SpringBoot 搭建api项目时，需要对api接口进行统一的解密和加密。
加密解密时机：
- 1、SpringMVC 套餐 ```@ControllerAdvice``` 、 ```HttpInputMessage```、```RequestBodyAdvice```
- 2、使用```HttpMessageConverter```
- 3、Servlet Api 过滤器 ```Filter```

经过测试发现：
- 方案1解密时仅对@RequestBody有效，并且对请求Content-Type 进行一些配置，可以实现但是并不友好；
- 方案2解密时也需要使用@RequestBody，Content-Type则在转换器中支持所有类型即可；另外从下面的转换方法的方法签名可以看出，需要返回具体的对象，若controller中入参的对象结构比较复杂，返回这个对象会有一定的成本。*本着开发时能交给框架就交给框架的懒人原则 :）*
``
protected Object readInternal(Class<?> clazz, HttpInputMessage inputMessage)
``
- 方案3 众所周知是一定可以实现的，并且对请求的后续mvc的处理流程完全透明，是比较理想的方案。

最后选择方案3

*好了，已经偏题了*
项目中同时也是使用了Filter做请求log，于是出现了LogFilter中log不到请求参数的问题，理想状态是解密的Filter第一个被执行，后续再执行其他的Filter。
#2、面向Google编程
google到排序的方法：
- 1、根据filter名自然排序的顺序执行（测试不行）
- 2、xml配置文件中配置filter的先后顺序（springboot还写xml？）
- 3、使用```FilterRegistrationBean```向Spring容器注册bean，同时指定```FilterRegistrationBean```的order值（亲测可行，但每个filter都需要写一个```FilterRegistrationBean```,不优雅，嫌弃）
- 4、使用注解```@ServletComponentScan```(亲测不行，原因后续会提到)

google没找到满意的解决方法，还是自己看下为什么不能排序吧。
#3、解决方法
这里写下处理方法：
添加```@Order```、```@Component```和```@WebFilter```即可，其他不需要处理。并且不能使用```@ServletComponentScan```，或者```@ServletComponentScan```的扫描范围不能包括想要排序的Filter
#4、排序分析
1、filter执行链执行方式
2、filter执行链的创建
3、ServletContext中Filter的来源
4、SpringBoo启动时如何处理Servlet的组件
5、SpringBoot中Filter不同注册方式的区别
6、@ServletComponentScan分析
#5、额外功能
## 排除模式的Filter
