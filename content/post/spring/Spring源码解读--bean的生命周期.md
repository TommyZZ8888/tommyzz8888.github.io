---
title: bean的生命周期
date: '2025-12-28T09:38:23+08:00'

categories:
- spring
---


spring在起初是是通过xml文件进行bean的配置的，本文就从xml文件配置作为bean生命周期的一个入口讲起。

一、猜想bean的生命周期
=============

先简单举例个xml的配置，通过如下方式即可配置我们的bean。

```xml
<bean id=? class=?>
<property name=? value=?>
<property name=? ref=?>
</bean>

<bean id=? class=?>
<constructor-arg name=? value=?>
<constructor-arg name=? ref=?>
</bean>
```

那么xml中的bean是如何加载到spring容器的呢？我们不妨做出如下的猜想：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4006e424860042ff98b827c8505809f1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

后面章节我们也是大体按照猜想的过程，在源码中逐步的去阅读源码，找到bean真正的生命周期是什么样的。

二、几个问题
======

2.1 spring是以什么方式实例化对象的？
-----------------------

我们知道的实例化对象的常见方式应该有以下几种： 1）new() 2）反射 3）工厂模式

spring是以何种方式呢？ 众所周知，spring中的bean默认scope是单例的，所以new()的方式肯定不好，我们不妨假装默认spring使用反射的方式，去看看源码的实现是什么样的。

常见的反射方式：

```ini
1）Class clazz = Class.forName()
2）Class clazz = 类名.class
3）Class clazz = 对象名.getClass()
```

拿到clazz文件后，我们就可以通过以下方式去实例化对象，后面我们看看在源码中是否能找到以下代码？

```ini
Constructor ctor = clazz.getDeclaredConstructor();
Object object  = ctor.newInstance();
```

2.2 实例化、初始化都做了哪些事情？这时候bean的状态是什么样的？
-----------------------------------

在我们学习jvm类加载机制时候就知道，类被加载的时候会有准备阶段和初始化的阶段，并且在准备阶段的时候，只是给静态文件在方法区分配内存，并赋默认值。

上面类加载阶段实际与spring的bean实例化，初始化阶段一样，在实例化阶段，会生成一个bean的对象，此时赋默认值，在初始化时，才会对bean进行一个赋值。

具体做了哪些操作我们通过下面的一幅图来简单概括：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ceef19564a654c91b4b06952862fb330~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

三、跟着代码找流程
=========

下面我们开始真正的跟随代码去找spring中bean的生命周期。Spring中，bean的整个生命周期其实是从AbstractApplicationContext中的refresh方法完成的。

```scss
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备，记录容器的启动时间startupDate, 标记容器为激活，初始化上下文环境如文件路径信息，验证必填属性是否填写
			prepareRefresh();

			// 获取新的beanFactory，销毁原有beanFactory、为每个bean生成BeanDefinition等
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 初始化beanFactory的各种属性
			prepareBeanFactory(beanFactory);

			try {
				// 模板方法，此时，所有的beanDefinition已经加载，但是还没有实例化。
				//允许在子类中对beanFactory进行扩展处理。比如添加ware相关接口自动装配设置，添加后置处理器等，是子类扩展prepareBeanFactory(beanFactory)的方法
				postProcessBeanFactory(beanFactory);

				// 实例化并调用所有注册的beanFactory后置处理器（实现接口BeanFactoryPostProcessor的bean，在beanFactory标准初始化之后执行）
				invokeBeanFactoryPostProcessors(beanFactory);

				//注册bean后置处理器
				registerBeanPostProcessors(beanFactory);

				// 初始化上下文的消息
				initMessageSource();

				// 初始化事件
				initApplicationEventMulticaster();

				// 模板方法，在容器刷新的时候可以自定义逻辑，不同的Spring容器做不同的事情。
				onRefresh();

				// 注册监听器，广播early application events
				registerListeners();

				// 实例化所有剩余的（非懒加载）单例
				// 比如invokeBeanFactoryPostProcessors方法中根据各种注解解析出来的类，在这个时候都会被初始化。
				// 实例化的过程各种BeanPostProcessor开始起作用。
				finishBeanFactoryInitialization(beanFactory);

				// refresh做完之后需要做的其他事情。
				// 清除上下文资源缓存（如扫描中的ASM元数据）
				// 初始化上下文的生命周期处理器，并刷新（找出Spring容器中实现了Lifecycle接口的bean并执行start()方法）。
				// 发布ContextRefreshedEvent事件告知对应的ApplicationListener进行响应的操作
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// 销毁已经生成的bean
				destroyBeans();

				// 重置激活状态.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```

### 1、加载配置文件

我们知道，spring定义bean可以通过xml，或者注解的形式，那么是如何将定义好的配置文件加载到IOC容器中呢，spring提供了一个接口规范**BeanDefinitionReader**：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fff112b926134eb380f9f9736aa97b8b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

除了这个接口之外，还定义了针对注解的**AnnotatedBeanDefinitionReader**。

那么有了这两个组件后我们的整体流程会如下图所示：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad8e6ae222d94a1b9ef8d640dc46ef55~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

将配置文件通过以上的接口和类加载到spring的ioc容器当中，处理成满足ioc容器的bean结构。

**下面具体看下源码是如何实现的**：

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备，记录容器的启动时间startupDate, 标记容器为激活，初始化上下文环境如文件路径信息，验证必填属性是否填写
			prepareRefresh();

			// 获取新的beanFactory，销毁原有beanFactory、为每个bean生成BeanDefinition等
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

在refresh方法中，目前主要关注obtainFreshBeanFactory()这个方法。

```scss
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
                //刷新
		refreshBeanFactory();
		return getBeanFactory();
	}
```

继续跟踪refreshBeanFactory()。

```scss
protected final void refreshBeanFactory() throws BeansException {
		//销毁原有的beanFactory
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			//创建一个BeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			//给当前的bean设置id
			beanFactory.setSerializationId(getId());
			//设置是否循环引用和是否重名注册
			customizeBeanFactory(beanFactory);
			//从配置文件加载bean信息为beanDefinition
			loadBeanDefinitions(beanFactory);
			this.beanFactory = beanFactory;
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

主要关注loadBeanDefinitions(beanFactory)。

```scss
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 为beanFactory创建一个XML格式的bean定义读取器
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		//给beanDefinitionReader设置环境变量
		beanDefinitionReader.setEnvironment(getEnvironment());
		//给beanDefinitionReader设置资源加载器
		beanDefinitionReader.setResourceLoader(this);
		//给beanDefinitionReader设置实体解析器
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		//目前没有实内容
		initBeanDefinitionReader(beanDefinitionReader);
		//从配置文件获取bean的定义
		loadBeanDefinitions(beanDefinitionReader);
	}
```

进入loadBeanDefinitions(beanDefinitionReader)。

```scss
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
		//获取配置文件的地址，/WEB-INF/下的xml文件
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			//遍历
			for (String configLocation : configLocations) {
				//加载bean定义
				reader.loadBeanDefinitions(configLocation);
			}
		}
	}
```

里面的具体这里不再深究了，后面会通过递归的方式逐步解析xml文件的各个节点信息。

通过上面你的分析，我们发现通过**BeanDefinitionReader**的解析处理，**xml中的bean信息最终变成了能被IOC容器接收的BeanDefinition，并且创建了一个BeanFactory**，自重的bean生命周期变成如下图所示：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/121053dd38bc4180a37729adf0985907~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 2、做准备工作

上一小节中，通过加载配置文件，生成了BeanDefinition和Beanfactory，后面继续跟踪refresh代码。在代码后面的执行过程中，实际要做很多的准备工作，例如提供BeanFactoryPostProcessor，注册BeanFactoryPostProcessor，国际化，初始化事件多播器，注册监听器等

```scss
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 准备，记录容器的启动时间startupDate, 标记容器为激活，初始化上下文环境如文件路径信息，验证必填属性是否填写
			prepareRefresh();

			// 获取新的beanFactory，销毁原有beanFactory、为每个bean生成BeanDefinition等
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 初始化beanFactory的各种属性
			prepareBeanFactory(beanFactory);

			try {
				// 空模板方法，此时，所有的beanDefinition已经加载，但是还没有实例化。
				//允许在子类中对beanFactory进行扩展处理。比如添加ware相关接口自动装配设置，添加后置处理器等，是子类扩展prepareBeanFactory(beanFactory)的方法
				postProcessBeanFactory(beanFactory);
				// 实例化并调用所有注册的beanFactory后置处理器（实现接口BeanFactoryPostProcessor的bean，在beanFactory标准初始化之后执行）
				invokeBeanFactoryPostProcessors(beanFactory);
				//注册bean后置处理器
				registerBeanPostProcessors(beanFactory);

				// 初始化上下文的消息
				initMessageSource();

				// 初始化事件
				initApplicationEventMulticaster();

				// 模板方法，在容器刷新的时候可以自定义逻辑，不同的Spring容器做不同的事情。
				onRefresh();

				// 注册监听器，广播early application events
				registerListeners();
```

这个过程中基本在做准备工作，除了BeanFactoryPostProcesser可以对已经定义的bean进行增强。经历过这些阶段后的流程如下。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d494e408f72478a80bbac86a10e4bda~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 3、实例化

当前面的准备工作做完后，真正开始bean的实例化：

```scss
 // 实例化所有剩余的（非懒加载）单例				
finishBeanFactoryInitialization(beanFactory);
```

跟踪到内部，发现，前面都在设置一些属性，直接看最后一句：

```scss
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// 实例化所有剩下单例bean
		beanFactory.preInstantiateSingletons();
	}
```

跟踪**beanFactory.preInstantiateSingletons()**方法，忽略其他方法，直接到**getBean()**。

```scss
public void preInstantiateSingletons() throws BeansException {
		if (logger.isTraceEnabled()) {
			logger.trace("Pre-instantiating singletons in " + this);
		}

		// 获取bean名称
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// 遍历实例化所有非延时bean
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged(
									(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					//获取bean
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

下面是真正实例化bean的方法：

```java
    @Override
	public Object getBean(String name) throws BeansException {
		//真正实例化bean的方法
		return doGetBean(name, null, null, false);
	}
```

首次启动时候，bean没有被创建过，我们直接跟踪到下面方法的**createBean(beanName, mbd, args)**：

```scss
	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {

		String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 创建bean实例
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					if (!StringUtils.hasLength(scopeName)) {
						throw new IllegalStateException("No scope name defined for bean ´" + beanName + "'");
					}
					Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new ScopeNotActiveException(beanName, scopeName, ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

createBean(beanName, mbd, args)方法：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			//真正创建bean的方法
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

继续跟踪上面方法的**doCreateBean(beanName, mbdToUse, args)**，这里面逐渐接近**反射创建实例化对象**了：

```scss
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			//创建bean实例
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isTraceEnabled()) {
				logger.trace("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//填充属性
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

跟踪**createBeanInstance(beanName, mbd, args)**：

```ini
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// 使用无参构造实例化bean
		return instantiateBean(beanName, mbd);
	}
```

跟踪**instantiateBean(beanName, mbd)**：

```scss
	protected BeanWrapper instantiateBean(String beanName, RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged(
						(PrivilegedAction<Object>) () -> getInstantiationStrategy().instantiate(mbd, beanName, this),
						getAccessControlContext());
			}
			else {
				//获取实例化策略进行实例化
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, this);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}
```

跟踪**getInstantiationStrategy().instantiate(mbd, beanName, this)**，我们发现了很熟悉的一条代码**clazz.getDeclaredConstructor()**，刚好证明了前面的问题，**spring是通过反射实现bean的实例化**的，并且通过最后一行代码**BeanUtils.instantiateClass(constructorToUse)**，在此方法内部执行了bean的实例化：

```scss
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		// Don't override the class with CGLIB if no overrides.
		if (!bd.hasMethodOverrides()) {
			Constructor<?> constructorToUse;
			synchronized (bd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse == null) {
					final Class<?> clazz = bd.getBeanClass();
					if (clazz.isInterface()) {
						throw new BeanInstantiationException(clazz, "Specified class is an interface");
					}
					try {
						if (System.getSecurityManager() != null) {
							constructorToUse = AccessController.doPrivileged(
									(PrivilegedExceptionAction<Constructor<?>>) clazz::getDeclaredConstructor);
						}
						else {
							//反射获取bean的构造
							constructorToUse = clazz.getDeclaredConstructor();
						}
						bd.resolvedConstructorOrFactoryMethod = constructorToUse;
					}
					catch (Throwable ex) {
						throw new BeanInstantiationException(clazz, "No default constructor found", ex);
					}
				}
			}
			return BeanUtils.instantiateClass(constructorToUse);
		}
		else {
			// Must generate CGLIB subclass.
			return instantiateWithMethodInjection(bd, beanName, owner);
		}
	}
```

**通过上面很长一段代码跟踪，我们发现了bean的实例化是通过BeanFactory经过一系列的处理，最终调用初始化策略，使用反射的方式实例化的**，那么最终的生命周期将变成下面样：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42927a459acd4700b5ffee87c6551d33~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

### 4、初始化

bean经过实例化后，其内部都是默认值，需要经过初始化进行赋值等操作，我们根据代码继续看它的实现过程。 在实例化过程中，有一个方法doCreateBean，在其执行过程中，创建完实例化对象后，对该对象进行了初始化，代码如下：

```scss
//填充属性
populateBean(beanName, mbd, instanceWrapper);
//初始化bean
exposedObject = initializeBean(beanName, exposedObject, mbd);
```

populateBean具体代码不做过多研究了，直接看initializeBean(beanName, exposedObject, mbd)：

```scss
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			//执行实现Aware接口的方法
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			//在初始化bean前执行bean的后置处理器
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			//执行初始化方法（xml配置的init-method）
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			//在初始化bean后执行bean的后置处理器
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

在上面的代码中有几个重要的过程： 1）**执行实现Aware接口的方法**。 2）**在初始化bean前执行bean的后置处理器**。 3）**执行初始化方法（xml配置的init-method）**。 4）**在初始化bean后执行bean的后置处理器**。 在经过上面的阶段后，则整个对象完成了其初始化的过程，成为了一个完整对象。 那么其生命周期就如下图所示：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77170ed490254d1187b15b4e89e31f62~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

本文转自 <https://juejin.cn/post/7016571732597145608>，如有侵权，请联系删除。