---
layout:     post
title:      Spring中Java配置类初始化源码分析
subtitle:   日常看看源码系列
date:       2018-08-03
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

### Spring容器初始化概述
Spring容器初始化的核心是`BeanFactory`的初始化，本文主要内容是注解的配置方式的BeanFactory的初始化。在容器整个初始化的过程中，无论是以前的xml文件配置还是这里的`Java类配置`，无非就是从配置源读取数据，然后从数据解析出`BeanDefinition`，接着注册到`BeanFactory`中。这个过程必须在容器实例化对象前完成，当然，如果开发者对某个bean的整个生命周期有足够的掌握，任何时候都可以进行这个类的初始化。但是这种情况势必会造成容器bean管理的混乱，所以还是建议按照Spring的Bean管理流程来管理Bean。还有如果是动态类，比如运用某些技术手段在运行时生成类的Bean，这种Bean肯定就是要动态注册到Spring容器，可以是定义成某个动态模块或使用子容器进行管理。
### 两种BeanDefintion解析方式
有阅读过一些Spring源码的童鞋应该知道,`ApplicationContext`和`BeanFactory`并没有管理BeanDefintion的能力，BeanDefintion注册能力来自`BeanDefinitionRegistry`接口，如下的类签名 :
```java
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable 
{
	//类实现
}
``` 
然而`BeanDefinitionRegistry`注册Bean定义的接口只接受BeanDefintion对象，并不接受何种来源的bean定义：
```
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition);
```
端庄而又优雅，分离了bean的管理和bean的来源，容器就只管理bean生命周期就可以，不用问哪来的。容器的子类中通过委托给xxxBeanDefintionReader对象来完成对bean定义的解析，不同来源的bean定义就使用不同的Reader。  
这里说下比较常见的两种bean定义方式： 
- 1、XML文件
- 2、Java注解

####XML文件配置
对于xml配置的容器子类*如：AbstractXmlApplicationContext*，向容器注册bean定义是在解析xml的同时注册到容器中，就是说xml中定义的bean这时候就已经解析注册完成，*Component-Scan除外，这属于注解的bean*。
  流程如下：
```
1.ClassPathXmlApplicationContext:构造器调用refresh();
2.AbstractApplicationContext的refresh()方法调用链如下：
refush()->obtainFreshBeanFactory()->refreshBeanFactory()->loadBeanDefinitions(...)
->reader.loadBeanDefinitions(configResources);
```
最终是委托给了`XmlBeanDefinitionReader`*持有容器实例* 进行加载。
reader对xml文件的解析加载分为两种，一种是默认命名空间，另外一种是特殊命名空间的，如：`<tx:xx>`、`<context:xx>`，
对于特殊标签则是通过处理器的方式进行委托处理：
```java
//默认标签：DefaultBeanDefinitionDocumentReader类
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	if (delegate.nodeNameEquals(ele, "import")) {
		importBeanDefinitionResource(ele);
	} else if (delegate.nodeNameEquals(ele, "alias")) {
		processAliasRegistration(ele);
	} else if (delegate.nodeNameEquals(ele, "bean")) {
		processBeanDefinition(ele, delegate);
	} else {
		if (!(delegate.nodeNameEquals(ele, "beans")))
			return;
		doRegisterBeanDefinitions(ele);
	}
}
//特殊标签解析：BeanDefinitionParserDelegate类
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
	String namespaceUri = getNamespaceURI(ele);
	NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver()
		.resolve(namespaceUri);
	if (handler == null) {
		...
		return null;
	}
	return handler.parse(ele, 
		new ParserContext(this.readerContext, this, containingBd));
}
```
对于不同的标签的处理，采用了策略模式，不同标签自动选择不同的处理器，非常灵活。

#### Java注解
其实上面xml配置的`Component-Scan`如果扫描到`@Configuation`的类也会触发相同的初始化流程，Java类配置的方式xml配置的精简版。  
对于Java类配置 *如：AnnotationConfigWebApplicationContext* 向容器注册bean定义通常都是通过两步完成的：
- 第一步：向容器注册Java类配置，就是标志了`@Configuation`的类；
- 第二步：通过`BeanDefinitionRegistryPostProcessor`在初始化容器的时候解析注册Java类配置类中定义的Bean；
整个初始化流程的关键就变成**容器后处理器**的注册和调用了，这部分的容器后处理器注册是在`AnnotatedBeanDefinitionReader`中完成的，别忘了reader是持有容器对象的，向容器注册啥东西都可以。

```java
//reader 向容器注册容器后处理器，注册时机是reader的构造方法
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry ..., Environment ...) {
	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	...
	//注册处理Java类配置的容器后处理器 
	AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```
### 配置类解析过程分析
#### Java类配置
被`@Configuration`注解标志的Java类都视为Java类配置。该类上还可以添加如下注解：
- 1、`@PropertySource`
- 2、`@CompomentScan`
- 3、`@Import`
- 4、`@ImportResource`
 
以上注解将会按照顺序解析，然后才到当前类的解析。

#### 注册解析Java类配置的容器后处理器
见上文

#### 配置解析入口
解析的入口是容器后处理器`ConfigurationClassPostProcessor`，如下是注册改容器后处理器的代码：
```java
//org.springframework.context.annotation.AnnotationConfigUtils类
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
		BeanDefinitionRegistry registry, @Nullable Object source) {
	...
	if (!registry.containsBeanDefinition(...)) {
		//创建容器后处理器bean定义
		RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
		def.setSource(source);
		//将容器后处理器bean定义注册到容器中
		beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
	}
	...
}
```
跟进`ConfigurationClassPostProcessor`  
既然是容器后处理器，则先见方法`postProcessBeanDefinitionRegistry`:
> 判断该处理器是否被重复执行，然后交给其他方法
> **这里记录疑问，为什么会重复执行？一个容器中可以有多个registry？**

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
	//1、这里时候防止该容器处理器被重复执行。
	int registryId = System.identityHashCode(registry);
	if (this.registriesPostProcessed.contains(registryId)) {
		throw new IllegalStateException(...);
	}
	if (this.factoriesPostProcessed.contains(registryId)) {
		throw new IllegalStateException(...);
	}
	this.registriesPostProcessed.add(registryId);
	//、执行处理
	processConfigBeanDefinitions(registry);
}
```
继续跟进`processConfigBeanDefinitions`，大致流程：
> 一、遍历目前Spring容器中所有Bean定义，找出Java类配置类，用集合configCandidates临时保存
> 二、一些解析的初始化工作，包括：bean名称生成策略、environment属性
> 三、使用ConfigurationClassParser 逐一解析configCandidate，解析结果保存在ConfigurationClassParser对象中
> 四、从ConfigurationClassParser对象获取解析结果ConfigurationClass集合，并注册到Spring容器中
> 以上四个步骤只处理了Java类配置类外部的bean定义的解析注册，如果Java类配置类中定义的某些Bean也是Java类配置类，则需要继续处理 
> 五、对当前Java类配置类中定义的Bean进行遍历，如果bean定义中存在Java类配置类，则将类取出并按照上面的加载流程进行加载，直到没找到Java类配置类的bean为止

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
	List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
	String[] candidateNames = registry.getBeanDefinitionNames();

	// 遍历目前Spring容器中所有Bean定义，找出Java类配置类
	// 这里有一个知识点，Spring识别哪些类为Java类配置类，见附录。
	for (String beanName : candidateNames) {
		BeanDefinition beanDef = registry.getBeanDefinition(beanName);
		if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
				ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
			}
		}
		else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
			configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
		}
	}

	// Return immediately if no @Configuration classes were found
	if (configCandidates.isEmpty()) {
		return;
	}

	// Sort by previously determined @Order value, if applicable
	configCandidates.sort((bd1, bd2) -> {
		int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
		int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
		return Integer.compare(i1, i2);
	});

	// bean名称生成策略
	// 这里可以探究一下，这个策略是不是可以自定义
	// Detect any custom bean name generation strategy supplied through the enclosing application context
	SingletonBeanRegistry sbr = null;
	if (registry instanceof SingletonBeanRegistry) {
		sbr = (SingletonBeanRegistry) registry;
		if (!this.localBeanNameGeneratorSet) {
			BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
			if (generator != null) {
				this.componentScanBeanNameGenerator = generator;
				this.importBeanNameGenerator = generator;
			}
		}
	}

	if (this.environment == null) {
		this.environment = new StandardEnvironment();
	}

	// 遍历上面找出来的Java类配置类，并进行逐一的解析处理
	// Parse each @Configuration class
	ConfigurationClassParser parser = new ConfigurationClassParser(
			this.metadataReaderFactory, this.problemReporter, this.environment,
			this.resourceLoader, this.componentScanBeanNameGenerator, registry);

	Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
	Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
	do {
		// 1.解析Java类配置类，解析结果保存在ConfigurationClass类的对象中
		parser.parse(candidates);
		parser.validate();

		// 2.获取ConfigurationClass对象
		Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
		configClasses.removeAll(alreadyParsed);

		// Read the model and create bean definitions based on its content
		if (this.reader == null) {
			this.reader = new ConfigurationClassBeanDefinitionReader(
					registry, this.sourceExtractor, this.resourceLoader, this.environment,
					this.importBeanNameGenerator, parser.getImportRegistry());
		}
		// 3.将ConfigurationClass对象中的解析结果注册到容器中
		this.reader.loadBeanDefinitions(configClasses);
		alreadyParsed.addAll(configClasses);

		// 4.对当前Java类配置类中定义的Bean进行遍历，如果bean定义中存在Java类配置类，
		// 则将类取出并按照上面的加载流程进行加载。这是Java类配置类内部bean定义的循环判断，直到没找到Java类配置类的bean为止
		candidates.clear();
		if (registry.getBeanDefinitionCount() > candidateNames.length) {
			String[] newCandidateNames = registry.getBeanDefinitionNames();
			Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
			Set<String> alreadyParsedClasses = new HashSet<>();
			for (ConfigurationClass configurationClass : alreadyParsed) {
				alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
			}
			for (String candidateName : newCandidateNames) {
				if (!oldCandidateNames.contains(candidateName)) {
					BeanDefinition bd = registry.getBeanDefinition(candidateName);
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
							!alreadyParsedClasses.contains(bd.getBeanClassName())) {
						candidates.add(new BeanDefinitionHolder(bd, candidateName));
					}
				}
			}
			candidateNames = newCandidateNames;
		}
	}
	while (!candidates.isEmpty());

	// Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
	if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
		sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
	}

	if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
		// Clear cache in externally provided MetadataReaderFactory; this is a no-op
		// for a shared cache since it'll be cleared by the ApplicationContext.
		((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
	}
}
```
以上是一个大致解析流程，下面具体看解析过程。主要的方法是`parser.parse(candidates);`  
根据代码调用，跳转到核心处理方法：
```java
//org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
		throws IOException {

	// 1.首先处理嵌套的内部类（Java类配置类），处理过程和普通Java类配置类相同，
	// 所以这里是一个递归的过程，进行逐一处理，递归出口就是所有内部类处理完成
	processMemberClasses(configClass, sourceClass);

	// 2.处理@PropertySource，将该注解上的配置文件加载到Environment中，代码中的@value可获取到属性值
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), PropertySources.class,
			org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) {
			processPropertySource(propertySource);
		}
		else {
			logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}

	// 3.处理扫描注解@ComponentScan，这个处理时候直接将bean定义注册到容器中，所以在注册完bean定义后直接递归处理了扫描到的Java类配置类
	// 设计者这么做的目的应该是复用ComponentScan的处理流程，ComponentScan是一种比较特殊的注册bean定义的方法，已经一个比较完整的模块。
	// 当然这也提高了处理的复杂度，这里处理的直接注册的bean定义和处理的Java类配置类，需要和其他注册方式妥善协调，否则会出现重复注册的问题
	Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
	if (!componentScans.isEmpty() &&
			!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
		for (AnnotationAttributes componentScan : componentScans) {
			// The config class is annotated with @ComponentScan -> perform the scan immediately
			//扫描注册
			Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
			// Check the set of scanned definitions for any further config classes and parse recursively if needed
			// 检查扫描结果
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(
						holder.getBeanDefinition(), this.metadataReaderFactory)) {
					// 递归调用解析
					parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
				}
			}
		}
	}

	// 4.处理@Import导入的配置，一般是Java类配置类。SpringBoot自动化配置的关键之一
	processImports(configClass, sourceClass, getImports(sourceClass), true);

	// 5.处理@ImportResource导入的配置，一般是xml或者groovy
	AnnotationAttributes importResource =
			AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
	if (importResource != null) {
		String[] resources = importResource.getStringArray("locations");
		Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
		for (String resource : resources) {
			String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
			configClass.addImportedResource(resolvedResource, readerClass);
		}
	}

	// 6.处理Java配置类的@Bean注解方法，这里就是bean的定义了
	Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
	for (MethodMetadata methodMetadata : beanMethods) {
		configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
	}

	// 7.处理接口默认方法，这是JDK1.8以后才有的接口默认方法实现
	processInterfaces(configClass, sourceClass);

	// 8.查找父类，如果父类也有注解，则会循环处理
	if (sourceClass.getMetadata().hasSuperClass()) {
		String superclass = sourceClass.getMetadata().getSuperClassName();
		if (superclass != null && !superclass.startsWith("java") &&
				!this.knownSuperclasses.containsKey(superclass)) {
			this.knownSuperclasses.put(superclass, configClass);
			// Superclass found, return its annotation metadata and recurse
			return sourceClass.getSuperClass();
		}
	}

	// No superclass -> processing is complete
	return null;
}
```
这里重点说下第4步，这里是SpringBoot灵活自动化配置的关键：
```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
		Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

	if (importCandidates.isEmpty()) {
		return;
	}

	if (checkForCircularImports && isChainedImportOnStack(configClass)) {
		this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
	}
	else {
		this.importStack.push(configClass);
		try {
			for (SourceClass candidate : importCandidates) {
				// 11111111111111111
				if (candidate.isAssignable(ImportSelector.class)) {
					// Candidate class is an ImportSelector -> delegate to it to determine imports
					Class<?> candidateClass = candidate.loadClass();
					ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
					ParserStrategyUtils.invokeAwareMethods(
							selector, this.environment, this.resourceLoader, this.registry);
					if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
						this.deferredImportSelectors.add(
								new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
					}
					else {
						String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
						Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
						processImports(configClass, currentSourceClass, importSourceClasses, false);
					}
				}
				// 22222222222
				else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
					// Candidate class is an ImportBeanDefinitionRegistrar ->
					// delegate to it to register additional bean definitions
					Class<?> candidateClass = candidate.loadClass();
					ImportBeanDefinitionRegistrar registrar =
							BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
					ParserStrategyUtils.invokeAwareMethods(
							registrar, this.environment, this.resourceLoader, this.registry);
					configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
				}
				// 3333333333333333
				else {
					// Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
					// process it as an @Configuration class
					this.importStack.registerImport(
							currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
					processConfigurationClass(candidate.asConfigClass(configClass));
				}
			}
		}
		// ...
	}
}
```
其实可以清晰的看到有三种情况:
第一种导入的类是`ImportSelector`类型的。利用反射实例化这个类，随后将一些应该注入的注入进去，如果是`DeferredImportSelector`类型的，那么就放到`deferredImportSelectors`这个集合里。如果不是这种类型的，就调用`selectImports`方法得到**需要导入的类**，随后继续递归此方法，进行导入; deferredImportSelectors调用时机是在`parser.parse(candidates);`方法中最后，可见延迟加载是延迟到其他所有Java类配置类加载完成后在进行加载；
第二种是`ImportBeanDefinitionRegistrar`类型，一样利用反射进行初始化，随后注入属性，随后调用了`configClass.addImportBeanDefinitionRegistrar`添加进来了; 
最后一种情况导入的就是一个配置文件`Configration`类，直接调用`processConfigurationClass`方法进行处理。
**至此，解析的流程就告一段落**，解析的数据都放到了`ConfigurationClass`中。
接着就是`ConfigurationClass`中bean定义注册，在`org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions`方法中可以看到
```java
// Read the model and create bean definitions based on its content
if (this.reader == null) {
	this.reader = new ConfigurationClassBeanDefinitionReader(
			registry, this.sourceExtractor, this.resourceLoader, this.environment,
			this.importBeanNameGenerator, parser.getImportRegistry());
}
this.reader.loadBeanDefinitions(configClasses);
```
这里通过`ConfigurationClassBeanDefinitionReader`进行委托注册bean：
```java
/**
 * Read {@code configurationModel}, registering bean definitions
 * with the registry based on its contents.
 */
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
	TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
	for (ConfigurationClass configClass : configurationModel) {
		loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
	}
}

/**
 * Read a particular {@link ConfigurationClass}, registering bean definitions
 * for the class itself and all of its {@link Bean} methods.
 */
private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
		TrackedConditionEvaluator trackedConditionEvaluator) {

	if (trackedConditionEvaluator.shouldSkip(configClass)) {
		String beanName = configClass.getBeanName();
		if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
			this.registry.removeBeanDefinition(beanName);
		}
		this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
		return;
	}

	if (configClass.isImported()) {
		registerBeanDefinitionForImportedConfigurationClass(configClass);
	}
	for (BeanMethod beanMethod : configClass.getBeanMethods()) {
		loadBeanDefinitionsForBeanMethod(beanMethod);
	}
	loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
	loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```
里面的`registerBeanDefinitionForImportedConfigurationClass`、`loadBeanDefinitionsForBeanMethod`、`loadBeanDefinitionsFromImportedResources`、`loadBeanDefinitionsFromRegistrars`分别做的事情时将此Configuration注册成了BeanDefinition,将配置文件里面的使用@Bean注释的方法变成了BeanDefinition，将ImportResource进来的配置文件加载解析，从资源文件加载合适的Bean,最后一个就是调用Config类中ImportBeanDefinitionRegistrar类型的对象的registerBeanDefinitions方法，实现他们内部的注册逻辑。

文章正文到这里吧，大概记录了下流程，方便日后查阅。

### 附录
1.Spring识别哪些类为Java类配置类
在`ConfigurationClassUtils`类中有相应的判断方法，如下：
a.完整的ConfigurationClass：
```java
// 有@Configuration标记的类
public static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
	return metadata.isAnnotated(Configuration.class.getName());
}
```
b.精简的ConfigurationClass：
首先接口肯定不是；
其次判断当前类是否有被（@Component，@ComponentScan，@Import，@ImportResource）至少一个标志，有的则是ConfigurationClass。；
最后，如果上面都没注解，若当前类的成员方法存在被@Bean标记的，则当前类是ConfigurationClass。
```java
public static boolean isLiteConfigurationCandidate(AnnotationMetadata metadata) {
	// Do not consider an interface or an annotation...
	if (metadata.isInterface()) {
		return false;
	}
	// Any of the typical annotations found?
	for (String indicator : candidateIndicators) {
		if (metadata.isAnnotated(indicator)) {
			return true;
		}
	}
	// Finally, let's look for @Bean methods...
	try {
		return metadata.hasAnnotatedMethods(Bean.class.getName());
	}
	catch (Throwable ex) {
		if (logger.isDebugEnabled()) {
			logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + ex);
		}
		return false;
	}
}
```
综上，是否是一个Java类配置类，@Configuration并不是必需的。有@Component，@ComponentScan，@Import，@ImportResource，@Bean 或者他们的子类出现即可。