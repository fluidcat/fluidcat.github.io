---
layout:     post
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

### 1、问题来源
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
### 2、面向Google编程
google到排序的方法：
- 1、根据filter名自然排序的顺序执行（测试不行）
- 2、xml配置文件中配置filter的先后顺序（springboot还写xml？）
- 3、使用```FilterRegistrationBean```向Spring容器注册bean，同时指定```FilterRegistrationBean```的order值（亲测可行，但每个filter都需要写一个```FilterRegistrationBean```,不优雅，嫌弃）
- 4、使用注解```@ServletComponentScan```(亲测不行，原因后续会提到)

google没找到满意的解决方法，还是自己看下为什么不能排序吧。
### 3、解决方法
这里写下处理方法：
添加```@Order```、```@Component```即可，其他不需要处理。并且不能使用```@ServletComponentScan```，或者```@ServletComponentScan```的扫描范围不能包括想要排序的Filter
### 4、排序分析
#### 1、filter执行链执行方式  
a. 通过断点进去```filterChain#doFilter```方法
```
public void doFilter(ServletRequest request, ServletResponse response)
    throws IOException, ServletException {

    if( Globals.IS_SECURITY_ENABLED ) {
        // ...
        internalDoFilter(req,res);
        // ...
        
    } else {
        internalDoFilter(request,response);
    }
}
``` 
b. 跟踪进入 internalDoFilter 可以看到如下关键代码 
```
private void internalDoFilter(ServletRequest request, ServletResponse response)
        throws IOException, ServletException {

    // Call the next filter if there is one
    if (pos < n) {
        // 获取并偏移下标
        ApplicationFilterConfig filterConfig = filters[pos++];
        try {
            Filter filter = filterConfig.getFilter();
            // ...
            // 执行过滤器逻辑（递归）
            filter.doFilter(request, response, this);
         }
    // ...
    // 所有过滤器前置执行结束，执行servlet
    servlet.service(request, response);
    // 执行servlet执行结束，递归出口
    // ...
```  
这里看到```filters[pos++]```,这是Filter责任链模式实现的核心，从```filterChain```内维护的filter列表依次获取并执行。  
> Servlet的Filter责任链模式是通过filter列表和递归调用实现的。  

于是我们得到一个结论：*filter的执行顺序取决与filter在```filterChain```的Filters列表的顺序，继续跟进Filters列表创建*

#### 2、Filters列表的创建  
找到```ApplicationFilterChain#addFilter```方法，filters由外部添加，断点继续往找；
找到```ApplicationFilterFactory#createFilterChain```方法，在这里创建处理链，如下：
```
public static ApplicationFilterChain createFilterChain(ServletRequest request,
            Wrapper wrapper, Servlet servlet) {
    // ...
    // 创建处理链
    filterChain = new ApplicationFilterChain();
    
    // ...
    FilterMap filterMaps[] = context.findFilterMaps();
    
    // ...
    // 循环从context找到的FilterMap
    for (int i = 0; i < filterMaps.length; i++) {
        // 判断filter是否符合当前请求 
        if (!matchDispatcher(filterMaps[i] ,dispatcher)) {
            continue;
        }
        if (!matchFiltersURL(filterMaps[i], requestPath))
            continue;
        // 获取真正的Filter，filterConfig保存了Filter
        ApplicationFilterConfig filterConfig = (ApplicationFilterConfig)
            context.findFilterConfig(filterMaps[i].getFilterName());
        if (filterConfig == null) {
            // FIXME - log configuration problem
            continue;
        }
        // 添加到处理链
        filterChain.addFilter(filterConfig);
    }
    // ...
    // Return the completed filter chain
    return filterChain;
}
```
这里也使用for循环遍历一个```filterMaps```,将符合的Filter加入Filters列表，意味着顺序是通过filterMaps传递过来的；  
#### 3、ServletContext中FilterMaps的来源  
断点发现FilterMaps的来源```StandardContext```两个方法，
```
public void addFilterMap(FilterMap filterMap) {
    validateFilterMap(filterMap);
    // 添加到filterMap
    filterMaps.add(filterMap);
    fireContainerEvent("addFilterMap", filterMap);
}

public void addFilterMapBefore(FilterMap filterMap) {
    validateFilterMap(filterMap);
    // 添加到filterMaps
    filterMaps.addBefore(filterMap);
    fireContainerEvent("addFilterMap", filterMap);
}
```  
继续断点往上跟进，根据如下调用栈找到是顺序的来源
![调用栈](/img/post-bg-filterMapsStackTraces.jpg)  
初始化:```ServletWebServerApplicationContext#selfInitialize```  
```
private void selfInitialize(ServletContext servletContext) throws ServletException {
    // 获取ServletContextInitializer，并依次执行
    for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
        beans.onStartup(servletContext);
    }
}
```  
这里每执行一次Filter就往filterMaps添加一个元素，由此可见顺序的信息再一次有上层决定，这里的上层是指ServletContext的初始化。  
在调用栈有几处关键的代码，这先按调用顺序贴一下，嫌多的可以跳过：  
代码a：```DynamicRegistrationBean#register```
```
protected final void register(String description, ServletContext servletContext) {
    // 将真正的filter注册到ServletContext中，并返回一个registration对象，后续使用这个对象对filter的元数据进行配置
    D registration = addRegistration(description, servletContext);
    if (registration == null) {
        logger.info(StringUtils.capitalize(description) + " was not registered (possibly already registered?)");
        return;
    }
    // 使用registration对filter进行配置(虽然就set一下数据，但是是关键数据)
    configure(registration);
}
```
代码b：```AbstractFilterRegistrationBean#configure```
```
protected void configure(FilterRegistration.Dynamic registration) {
    // 父类方法，set元数据initParams
    super.configure(registration);
    // ...
    
    // 添加FilterMaps到ServletContext的实现
    registration.addMappingForUrlPatterns(dispatcherTypes, this.matchAfter,
                    StringUtils.toStringArray(this.urlPatterns));
    // ...
}
```  
代码c：父类调用```super.configure(registration)```
```
// 初始化asyncSupported，initParameters
// 这里是springboot与servlet api的连接点之一（springboot如何传递filter的元数据到servlet api的），后续会用到 
protected void configure(D registration) {
    registration.setAsyncSupported(this.asyncSupported);
    if (!this.initParameters.isEmpty()) {
        registration.setInitParameters(this.initParameters);
    }
}
```  

代码d: ```ApplicationFilterRegistration#addMappingForUrlPatterns```
```
public void addMappingForUrlPatterns(EnumSet<DispatcherType> dispatcherTypes, boolean isMatchAfter,String... urlPatterns) {
    // 创建FilterMap
    // 我个人想法：从上文可知处理链使用FilterMaps生成，FilterMap的作用是方便请求进行匹配，而不用频繁读取Filter的元数据
    // 某种程度上讲也起到解耦的作用，将框架的代码和开发者代码隔离。可参考FilterMap注释
    FilterMap filterMap = new FilterMap();
    filterMap.setFilterName(filterDef.getFilterName());
    // ...

    // 添加到ServletContext
    if (isMatchAfter) {
        context.addFilterMap(filterMap);
    } else {
        context.addFilterMapBefore(filterMap);
    }
}
```

#### 4、SpringBoot启动时如何处理初始化Servlet的组件
上文子目录3知道，Servlet的组件通过ServletContextInitializer进行初始化与配置
继续跟踪方法```getServletContextInitializerBeans```,定位到代码：
```
public ServletContextInitializerBeans(ListableBeanFactory beanFactory,
        Class<? extends ServletContextInitializer>... initializerTypes) {
    // 初始化类对象集合
    this.initializers = new LinkedMultiValueMap<>();
    
    // 指定用于初始化Servlet组件的初始化类，这里指定了ServletContextInitializer
    this.initializerTypes = (initializerTypes.length != 0) ? Arrays.asList(initializerTypes) 
        : Collections.singletonList(ServletContextInitializer.class);
    
    // 根据初始化类从Spring容器获取bean到initializers集合
    addServletContextInitializerBeans(beanFactory);
    
    // 转换Spring容器中以非ServletContextInitializer形式存在的Servlet组件bean
    // 就是把Spring容器中已经有的的Filter对象、Servlet对象转换成ServletContextInitializer
    // 为啥要转换？因为统一了处理方式，SpringBoot与Servlet之间初始化时使用ServletContextInitializer完成bean的传递
    // (将Spring容器的bean注册到Servlet上下文)
    // 这也是为什么直接仅使用@Component注解Filter与Servlet类后组件久能生效的原因
    addAdaptableBeans(beanFactory);
    
    // 关键代码！！！！！！！！！！！！！！！！！！！！
    // 对获取到的Servlet组件进行排序，且使用的是*AnnotationAwareOrderComparator*
    List<ServletContextInitializer> sortedInitializers = this.initializers.values().stream()
            .flatMap((value) -> value.stream().sorted(AnnotationAwareOrderComparator.INSTANCE))
            .collect(Collectors.toList());
    this.sortedList = Collections.unmodifiableList(sortedInitializers);
    logMappings(this.initializers);
}
```
至此，SpringBoot中Servlet组件的顺序确定了。  
撒花~~~

由上文跟踪分析可知道 上文的解决方法是可行的，使用@order+@Component注解可以实现对Filter执行顺序的控制，
但是也有弊端，会导致@WebFilter注解的参数全部失效。
原因：上文可以知道Filter的初始化和注册过程交给了```FilterRegistrationBean```完成，FilterRegistrationBean的创建与初始化过程并没有读取@WebFilter的任何值，所以@WebFilter是失无效的
弥补办法：编写Filter并继承FilterRegistrationBean，重写configure方法对@WebFilter进行解析（这时候不用@Component注解），该解决办法未验证。
#### 5、SpringBoot中Filter不同注册方式的区别
SpringBoot中注册Filter的方法笔者知道有三种方式：
- 1、实现Filter接口 + @Component
- 2、@ServletComponentScan + @WebFilter注解
- 3、包装成FilterRegistrationBean注册到Spring中  
相同：
- 最终都是包装成FilterRegistrationBean对象然后再统一注册到ServletContext中，见上文代码段  
差异：
- 1、3 都会导致@WebFilter注解失效，而 2 不会导致失效，但是 3 可以手动set配置
- 1、3 都可以控制顺序而 2 无法控制（3手动set order值）

#### 6、@ServletComponentScan分析  
emmmmmmm，通过```ServletComponentRegisteringPostProcessor```实现，直接贴代码吧
```
/************************* ServletComponentRegisteringPostProcessor *******************************/
class ServletComponentRegisteringPostProcessor implements BeanFactoryPostProcessor, ApplicationContextAware {
	private static final List<ServletComponentHandler> HANDLERS;
	static {
		List<ServletComponentHandler> servletComponentHandlers = new ArrayList<>();
		servletComponentHandlers.add(new WebServletHandler());
		// 1、定义Filter扫描处理器
		servletComponentHandlers.add(new WebFilterHandler());
		servletComponentHandlers.add(new WebListenerHandler());
		HANDLERS = Collections.unmodifiableList(servletComponentHandlers);
	}
	
	// xxxxxx 
	// balabala
	
	private void scanPackage(ClassPathScanningCandidateComponentProvider componentProvider, String packageToScan) {
		for (BeanDefinition candidate : componentProvider.findCandidateComponents(packageToScan)) {
			if (candidate instanceof ScannedGenericBeanDefinition) {
				for (ServletComponentHandler handler : HANDLERS) {
				// 2、执行扫描处理器
					handler.handle(((ScannedGenericBeanDefinition) candidate),
							(BeanDefinitionRegistry) this.applicationContext);
				}
			}
		}
	}
	// ........
	
	/********************************** WebFilterHandler#doHandle *********************************************/
	// 这很明显了吧，转换成FilterRegistrationBean
	public void doHandle(Map<String, Object> attributes, ScannedGenericBeanDefinition beanDefinition,
       			BeanDefinitionRegistry registry) {
       		BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(FilterRegistrationBean.class);
       		builder.addPropertyValue("asyncSupported", attributes.get("asyncSupported"));
       		builder.addPropertyValue("dispatcherTypes", extractDispatcherTypes(attributes));
       		builder.addPropertyValue("filter", beanDefinition);
       		builder.addPropertyValue("initParameters", extractInitParameters(attributes));
       		String name = determineName(attributes, beanDefinition);
       		builder.addPropertyValue("name", name);
       		builder.addPropertyValue("servletNames", attributes.get("servletNames"));
       		builder.addPropertyValue("urlPatterns", extractUrlPatterns(attributes));
       		registry.registerBeanDefinition(name, builder.getBeanDefinition());
     }
```
上述代码完了之后就是前面的流程了。嗯，端庄又优雅。  
### 5、额外功能
#### 排除模式的Filter
由前面那FilterMaps看到，Filter无法支持排除模式，但是又想像SpringMVC拦截器那样使用排除模式，那就自己加一个URL匹配吧

贴一个demo
```
public abstract class OrderedExcludeModeFilter extends OncePerRequestFilter {
    /**
     * 排除URL模式.
     */
    private String exclude = null;  //不需要过滤的路径集合
    private Pattern pattern = null;  //匹配不需要过滤路径的正则表达式

    public void setExclude(String exclude) {
        if (!StringUtils.isEmpty(exclude)) {
            this.exclude = exclude;
            pattern = Pattern.compile(getRegStr(exclude));
        }
    }

    /* (non-Javadoc)
     * @see org.springframework.web.filter.GenericFilterBean#initFilterBean()
     */
    @Override
    public void initFilterBean() throws BeansException {
        FilterConfig config = super.getFilterConfig();
        // 从filter配置获取排除的url
        if (config != null) {
            // 尝试从注解中获取
            WebInitParam[] webInitParams = this.getClass().getAnnotation(WebFilter.class).initParams();
            if (webInitParams != null && webInitParams.length > 0) {
                Arrays.stream(webInitParams).forEach(param -> {
                    if ("excludeUrl".equals(param.name())) {
                        this.exclude = param.value();
                    }
                });
            }
            // 尝试从FilterConfig中获取
            String config_excludeUrl = config.getInitParameter("excludeUrl");
            if (!StringUtils.isEmpty(config_excludeUrl)) {
                this.exclude = config_excludeUrl;
            }
            if (!StringUtils.isEmpty(exclude)) {
                pattern = Pattern.compile(getRegStr(exclude));
            }
        }
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        if (pattern != null) {
            // 请求路径
            String requestURI = request.getRequestURI();
            if (StringUtils.isEmpty(requestURI)) {
                requestURI = requestURI.replace(request.getContextPath(), "");
            }
            // 排除模式过滤
            if (pattern.matcher(requestURI).matches()) {
                filterChain.doFilter(request, response);
                return;
            }
        }
        doExcludeFilterInternal(request, response, filterChain);
    }

    protected abstract void doExcludeFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException;

    /**
     * 将传递进来的不需要过滤得路径集合的字符串格式化成一系列的正则规则
     *
     * @param str 不需要过滤的路径集合
     * @return 正则表达式规则
     */
    private String getRegStr(String str) {
        if (StringUtils.isEmpty(str)) {
            String[] excludes = str.split(";");  //以分号进行分割
            int length = excludes.length;
            for (int i = 0; i < length; i++) {
                String tmpExclude = excludes[i];
                //对点、反斜杠和星号进行转义
                tmpExclude = tmpExclude.replace("\\", "\\\\").replace(".", "\\.").replace("*", ".*");

                tmpExclude = "^" + tmpExclude + "$";
                excludes[i] = tmpExclude;
            }
            return String.join("|", excludes);
        }
        return str;
    }
}
```

#### 本文完，撒花~~~ 欢迎指正