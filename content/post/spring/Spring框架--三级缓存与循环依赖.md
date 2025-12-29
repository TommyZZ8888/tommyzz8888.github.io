---
title: 三级缓存和循环依赖
date: '2025-12-28T09:38:23+08:00'

categories:
- spring
---



1.[循环依赖](https://so.csdn.net/so/search?q=%E5%BE%AA%E7%8E%AF%E4%BE%9D%E8%B5%96&spm=1001.2101.3001.7020)
------------------------------------------------------------------------------------------------------

首先我们需要明白什么是循环依赖 ， 打个比方 ， 就是说A对象在创建的过程中 ， 需要[依赖注入](https://so.csdn.net/so/search?q=%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5&spm=1001.2101.3001.7020)B对象 ， 但是B对象没有 ， 就需要去创建 ， 而在创建B对象的过程中又需要注入A对象 ， A对象此时还在创建中，所以就构成了一个死循环 ， A，B相互依赖 这样的关系被成为**循环依赖**（当然 ， 可能还会有其他的情况），下面我们就来看看Spring是如何让解决循环依赖的

2.一二三级缓存
--------

三个缓存对应着三个不同的Map

### 一级：singletonObjects

这个缓存也就是单例池 ， 它存放的是完整的经过Bean生命周期的Bean

### 二级：earlySingletonObject

这个缓存存放的一个残疾Bean ， 怎么理解呢？就是创建到一半就放进去了

### 三级：singletonFactories

这个缓存存放的是ObjectFactory ， 表示用来 创建早期Bean 对象的工厂

3.情况一
-----

A ， B对象相互依赖

比如现在有两个Service ， 分别是AService和BService

AService：

```java
@Componentpublic class AService {     @Resource    private BService bService;     public void test(){        System.out.println("AService test() -> " + bService);    } }
```

BService：

```typescript
@Componentpublic class BService {     @Resource    private AService aService;     public void test(){        System.out.println("BService test() -> " + aService);    } }
```

如果不考虑Spring的话 ， 循环依赖是很正常的事情 ， 比如

```java
AService aService = new AService();BService bService = new BService();aService.bService = bService;bService.aService = aService;
```

但是 ， 如果考虑Spring的话这就是一个问题 ， 因为如果使用Spring ， 那么就会有控制反转， 这个对象就不在是由你自己去创建了 ， 而是在Spring内部经过一系列Bean的生命周期 ， 就是因为有了Bean的生命周期 ， 所以才会出现循环依赖  ，那我们简单回顾一下**Bean的生命周期**

### Bean的生命周期

Bean的生命周期具体指的就是一个Bean对象从创建到销毁的过程 ， 其生成步骤如下

1.Spring会通过扫描Class得到BeanDefintion（Bean的定义）

2.根据BeanDefintion去生成对象

3.首先通过Class来进行构造方法的推断（推断构造方法）

4.根据推断出来的构造方法然后反射得到一个对象（暂时叫做原始对象）

5.填充原始对象的属性（属性填充 ， 依赖注入）

6.如果原始对象中的某个方法被AOP了 ， 那么就需要根据原始对象生成一个代理对象

7.把最终生成的原始对象放入单例池 ， 也就是上面的一级缓存(singletonObjects) , 下次getBean从此Map获取就行

这是一个大致的过程 ， 对于Spring而言 ， Bean的生命周期不止这么7步 ， 还包括Aware回调 ， 初始化等等 ，可以发现在Spring中构造一个Bean包括了new这个步骤（第四步），生成一个原始对象之后再来进行**属性注入** , 那么对于**情况1**而言他的属性注入的过程大概是这样的: 

> 创建AService得到原始对象 ， 然后去给BService属性赋值 ， 此时就会根据BService属性的类型或名字在BeanFactory寻找BService所对应的单例Bean ， 如果存在 ， 直接赋值给BService ， 如果不存在则需要生成一个BService对应的Bean ， 然后赋值给BService 如下图

![](https://i-blog.csdnimg.cn/blog_migrate/8c8051b5e2e62385bb18dc0f6284e328.png)

通过上图 ， 我们可以很清晰的看出来循环依赖出现的原因 ， 如何打破这个循环呢？在中间加一层缓存 ， 如下图

![](https://i-blog.csdnimg.cn/blog_migrate/09fce9c494ca263b4d57e5470b238111.png)

流程我就不必多做解释了 ， 此时从缓存中获取的AService为原始对象 ， 还不是最终的Bean ， BService的原始对象依赖注入完后 ， BService的生命周期结束 ， 那么AService的生命周期也结束

因为在整个过程中AService都是单例的 ， 所以即使从缓存中拿到的AService的原始对象也没有关系 ， 因为在后续的Bean生命周期中 ，AService在堆内存中没有发生变化

所以**情况一的循环依赖**也就完美的解决了 , 那么又会产生新的问题：

1.一个缓存就可以解决循环依赖， 为什么要三级缓存？

用下面情况二的例子来分析分析

**4.情况二**
---------

还是AService ， BService相互依赖的场景 ， 但是多了一层AOP ， 就是AService的原始对象赋值给BService时 ， 进行了**AOP ,** 那么AService进行AOP之后 ， 它的真实对象是代理对象 ， 那么BService中AService的值是原始对象的值 ，那么就会产生**BService的AService值和AService的实际值不符的问题**

AOP就是通过一个**BeanPostProcessor**来实现的 ， 而这个BeanPostProcessor就是

**AnnotationAwareAspectJAutoProxyCreator，** 它的父类是**AbstractAutoProxyCreator（如下图）,**而在SpringAOP中要么利用**JDK动态代理** ， 要么利用**CGLib的动态代理** ， 所以如果给这个类中的某一个方法设置了切面 ， 那么它最后都会生成一个代理对象

![](https://i-blog.csdnimg.cn/blog_migrate/c3e726d3a12166732b7ce8551b0476af.png)

 基本流程看下图

![](https://i-blog.csdnimg.cn/blog_migrate/646a8d9195656d1b9e378219ee2d2162.png)

 而我们都知道 ， AOP是Spring除开IOC的另外一大功能 ， 而循环依赖又属于IOC的范围 ， 所以如果想要这两者功能共存 ， 就必须使用其他的手段：**三级缓存 singletonFactories**

首先 这个缓存存放的是（beanName：ObjectFactory），在Bean的生命周期中 ， 构造完一个原始对象就生成一个ObjectFactory , 然后缓存起来 ，这个ObjectFactory是一个函数式接口 ， 所以支持lambda表达式 ， **() -> getEarlyBeanReference(beanName, mbd, bean);**而这个lambda就表示是一个ObjectFactory ， 他会去执行这个方法 ， 这个方法在**SmartInstantiationAwareBeanPostProcessor中**

![](https://i-blog.csdnimg.cn/blog_migrate/22ba1efbca8fa7e7ad2fabfafc088fa1.png)

代码的大概意思就是说  ，找到继承了**InstantiationAwareBeanPostProcessor**这个类的**BeanPostProcessor**，就表示你需要进行AOP ，然后获取 **SmartInstantiationAwareBeanPostProcessor**这个 **BeanPostProcessor ，** 然后执行**getEarlyBeanReference()**方法 ， 由于这是一个接口 ， 我们直接看实现类是如何实现的

![](https://i-blog.csdnimg.cn/blog_migrate/b8acd3cddf16b2d533c07ce381ad41c4.png)

 1.首先是执行getCacheKey()获取key的名称 ， 如下

![](https://i-blog.csdnimg.cn/blog_migrate/ab05c8f64580e9653a8b23e6ef2fe0ac.png)

他会判断beanName是否为null ， 如果不为空 ， 那么判断是否是FactoryBean ， 如果是就拼接&符号 ， 否则直接返回 ， 如果beanName为null ， 最直接返回

2.然后把cacheKey当作key ， bean当作value ， 放入**earlyProxyReferences（关于**earlyProxyReferences下面有解释**），**然后执行**wrapIfNecessary()**进行AOP

![](https://i-blog.csdnimg.cn/blog_migrate/e07c82c6fca8b412b20f3df961a26514.png)

 3.该方法大概意思就是如果你符合AOP的条件 ， 那么我就创建一个代理对象返回 ， 如果不需要，那么返回原始对象

到这其实可以体会到**二级缓存为什么放的是残级Bean，因为此时创建一个代理对象 ， 还没有完成Bean的生命周期 ， 而三级缓存的ObjectFactory的lambda也可以理解为一段逻辑（大概就是需要AOP的时候， 那么返回一个代理对象 ， 如果不需要AOP我还是返回的原始对象），但是不管返回什么对象都是没有完成Bean生命周期的**

那么什么时候调用**getEarlyBeanReference()**这个方法呢？回到循环依赖场景，用一张图就大概可以理清这个思路了

![](https://i-blog.csdnimg.cn/blog_migrate/a73318999a8dcef57163ecb21818b0c8.png)

 就是创建AService ， 然后会产生一个AService的原始对象 ， 并且key为beanName ， Value为**lambda**表达式放入三级缓存 ， 然后注入BService ， 生成BService原始对象 ， 此时需要注入AService就要从单例池获取 ， 取不到 ， 从二级缓存获取 ， 取不到 ， 然后从三级缓存获取 ， 并执行**lambda**表达式，如果符合AOP的条件 ， 那么返回代理对象 ， 如果不符合 ， 返回原始对象 ， 然后赋值给BService的AService ， 然后BService完成创建

源码如下：

1.放入三级缓存

![](https://i-blog.csdnimg.cn/blog_migrate/f0541ef2d7a1443a052c0660c2092a91.png)

2\. 从一二三级缓存获取，然后执行表达式

![](https://i-blog.csdnimg.cn/blog_migrate/d514a7027f7768faefcb17ccca6ec3d9.png)

 这是getSingletion方法

![](https://i-blog.csdnimg.cn/blog_migrate/64af6d291a819a2cdcf22444ef3966f5.png)

此时AService正常进行AOP，但是前面已经执行过AOP了 ， 所以对于AServicec本身而言就不需要AOP了所以就又产生了一个问题：**怎么判断是否执行过AOP了？**

会利用**earlyProxyReferences，**就是上面方法提到的

![](https://i-blog.csdnimg.cn/blog_migrate/b8acd3cddf16b2d533c07ce381ad41c4.png)

再**AbstractAutoProxyCreator**类中的**postProcessAfterInitialization()**会判断

![](https://i-blog.csdnimg.cn/blog_migrate/fcb792b958e37686f3862da7899c671a.png)

如果成功移除 ， 表示不需要AOP了， 否则就需要AOP

对于AService而言 ， 进行了AOP的判断之后 ， 以及执行了BeanPostProcessor之后 ， 就需要把AService对应的对象放入单例池了 ，但是我们知道 ，如果进行AOP之后那么就要把代理对象放入单例池 ， 那么代理对象就可以在二级缓存拿到了 ， 然后放入单例池

**但也不是所有情况的循环依赖都能解决 ， 比如AService和BService都是原型的等**

**5.总结：**
---------

### 5.1.singletonObjects：单例池 ， 一级缓存

缓存的是经历了**完整的Bean生命周期**的Bean

### 5.2 earlySingletonObjects：二级缓存

缓存的是**未经过完整的Bean生命周期**的Bean，如果出现了循环依赖 ， 那么就会提前把这个暂时未经过Bean生命周期的Bean放入这个缓存 ， 如果这个Bean需要经过AOP ， 那么就会把代理对象放入这个缓存 ， 就算是经过了AOP ， 那么这个代理对象代理的真实对象也是是未完成生命周期的 ， 所以我称它为**残疾Bean ，** 这个缓存可以理解为存放的是**未经过完成的Bean生命周期的Bean**

### **5.3** **singletonFactories：三级缓存**

缓存的是ObjectFactory ， 也就是一个**Lambda表达式** ， 这个表示在存入的时候不会执行 ， 在get的时候会执行，在每个Bean生成一个原始对象的时候 ， 都会基于这个原始对象暴露一个lambda表达式 ， 然后放到这个缓存 ， **这个lambda可能用到 ， 也可能用不到 ,** 如果当前Bean没有出现循环依赖的情况 ， 那么这个表达式就没用 ， 按照自己的生命周期正常执行 ， 执行完之后把这个对象放入单例池 ， 如果出现了循环依赖（**当前创建的Bean被其他的Bean依赖了**） ， 那么就会把这个表达式取出来 ， 然后执行得到一个对象 ， 并把得到的缓存放入二级缓存 ， （如果当前Bean需要AOP ， 那么执行完lambda之后得到的就是一个代理对象 ，如果无需AOP ， 那就是原始对象）

### **5.4 earlyProxyReferences**

其实还有这个缓存 ， 用来记录某个原始对象是否进行了AOP

本文转自 <https://blog.csdn.net/qq_45001002/article/details/124420213?spm=1001.2014.3001.5506>，如有侵权，请联系删除。