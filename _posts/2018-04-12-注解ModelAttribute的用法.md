---
layout:     post
title:      注解ModelAttribute的用法
subtitle:   ModelAttribute可以标注在方法和参数上
date:       2018-04-12
author:     Belin
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - MVC模型
    - 参数绑定
	- Java
---

> 正所谓前人栽树，后人乘凉。
>
> 感谢[Huxpro](https://github.com/huxpro)提供的博客模板

@ModelAttribute使用大致有有两种，一种是是直接标记在方法上，一种是标记在方法的参数中，两种标记方法产生的效果也各不相同。
### 注解标注在方法上
当同一个controller中有任意一个方法被@ModelAttribute注解标记，页面请求只要进入这个控制器，不管请求那个方法，均会先执行被@ModelAttribute标记的方法，所以我们可以用@ModelAttribute注解的方法做一些初始化操作。当同一个controller中有多个方法被@ModelAttribute注解标记，所有被@ModelAttribute标记的方法均会被执行，按先后顺序执行，然后再进入请求的方法。  
当注解@ModelAttribute的方法上还有@RequestMapping时，其他请求不会先执行该方法，且该方法的返回值将作为此次请求model中的value，key为@ModelAttribute的value属性指定的值；若没指定@ModelAttribute的value属性，则key与方法返回的数据类型有关，例如返回的是String类型，则key为"string";建议指定@ModelAttribute的value属性，在页面取值比较明确。
*注：同时@ModelAttribute和@RequestMapping修饰的方法转向的视图是当前RequestMapping的路径。*  
例如：
```java
@Controller
@RequestMapping("/index")
public class IndexController{

    @RequestMapping(value = "/list")
    @ModelAttribute
    public String list(ModelMap model) {
		return "HelloWorld!";
	}
}
```
*此时转向的视图是"**index/list.html**"*
### 注解标注在参数前
SpringMVC会自动匹匹配页面传递过来的参数的name属性和后台控制器中的方法中的参数名，如果参数名相同，会自动匹配，如果控制器中方法是封装的bean,会自动匹配bean中的属性，其实这种取值方式不需要用@ModelAttribute注解，只要满足匹配要求，也能拿得到值。   
若使用如下代码：
```java
@Controller
@RequestMapping(value="/model")
public class ModelAttributeTest {

    @ModelAttribute("pojo")
    public PojoTest init( PojoTest pojo)
    {
        pojo.setSex("男");
        return pojo;
    }

    @RequestMapping(value="/modelTest")
    public String modelTest(@ModelAttribute("pojo") PojoTest pojo) 
    {
        pojo.setUserName("小明");
        return "modelTest";
    }
}
```
 则modelTest拿到inint方法中的pojo对象，合并两次set的参数后返回页面。

