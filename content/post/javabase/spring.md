---
title: spring
date: '2025-12-28T09:38:23+08:00'

categories:
- java 基础
---


# spring

### 1、Bean的生命周期

在spring项目中，bean是最重要的存在。

bean的生命周期分为三部分：==**生产**==  >>    ==**使用**==  >>    ==**销毁**==

其中生产部分比较复杂。

生产部分分为四部分： **实例化**  >>  **填充属性**  >>  **初始化**  >>  **销毁**  



普通java创建对象和spring创建bean实例不同。

java创建对象分为四部分：将java文件编译为class文件，等到类需要被初始化时（new或者反射），将class文件通过类加载器加载到jvm，，最后初始化对象供我们使用。



spring所管理的bean不同，除了class对象，还会使用beandefinition的实例描述对象信息。

我们可以在spring管理的bean对象上添加一系列描述：比如@scope，@lazy，@DependOn。

java是描述类的信息，spring是描述对象的信息。

在spring启动时，会通过xml/注解/javaconfig来查询需要被spring管理的bean信息。然后将这些查询到的bean封装成beanDefinition，，最后把这些信息放到BeanDefinitionmap中，到这里只是把定义的元数据加载起来，还没有进行实例化。

这个map中的key是beanname，value是bean。

然后通过**createBean**去遍历**beanDefinitionmap**,执行**BeanFactoryPostProcessor**的bean工厂后置处理器的逻辑。我们可以自定义BeanFactoryPostProcessor来对我们定义好的**bean元数据进行获取或者修改**。



接下来就是实例化对象了，通过**doCreateBean**方法对bean进行实例化，spring一般是通过**反射**来选择合适的构造器对bean进行实例化。

实例化之后将对象放到**三级缓存**中。

这里的实例化只是把bean对象创建出来，还没有进行属性填充的。比如我的对象是A，A又依赖B对象，此时B对象还是null。

接下来通过**populateBean**方法来对bean进行属性填充。

填充之后开始进行初始化，通过**InvokeAwareMethod**方法来对实现了aware相关接口的bean填充相关的资源。比如我们在项目中会抽取出来一个实现了ApplicationContextAware接口的工具类，来通过获取ApplicationContext对象来获取相关的bean。

之后会通过**BeanPostProcesser**后置处理器，该处理器有两个方法，**before和after**。先执行**BeanPostBeforeProcessor**方法，之后执行**InvokeInitMethod**方法，比如**@PostConstruct**，实现了tializatingBean接口的方法。最后执行**beanPostProcessor的after**方法。这里就初始化完了。然后会把二级缓存给remove掉，塞到一级缓存中。我们自己去getBean的时候，实际上拿到的是一级缓存的。

销毁的时候就看有没有配置相关的destroy方法，执行就完事了



最后总结：

在Spring Bean的生命周期，Spring预留了很多的hook给我们去扩展

Bean实例化之前有BeanFactoryPostProcessor

Bean实例化之后，初始化时，有相关的Aware接口供我们去拿到Context相关信息

环绕着初始化阶段，有BeanPostProcessor（AOP的关键）

在初始化阶段，有各种的init方法供我们去自定义

而循环依赖的解决主要通过三级的缓存

在实例化后，会把自己扔到三级缓存（此时的key是BeanName，Value是ObjectFactory）

在注入属性时，发现需要依赖B，也会走B的实例化过程，B属性注入依赖A，从三级缓存找到A

删掉三级缓存，放到二级缓存



关键源码方法（强烈建议自己去撸一遍）

- `org.springframework.context.support.AbstractApplicationContext#refresh`(入口)
- `org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization`(初始化单例对象入口)
- `org.springframework.beans.factory.config.ConfigurableListableBeanFactory#preInstantiateSingletons`(初始化单例对象入口)
- `org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String)`（万恶之源，获取并创建Bean的入口）
- `org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean`（实际的获取并创建Bean的实现）
- `org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String)`（从缓存中尝试获取）
- `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])`（实例化Bean）
- `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean`（实例化Bean具体实现）
- `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance`（具体实例化过程）
- `org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingletonFactory`（将实例化后的Bean添加到三级缓存）
- `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean`（实例化后属性注入）
- `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)`（初始化入口）



### 2、Spring @[Configuration](https://so.csdn.net/so/search?q=Configuration&spm=1001.2101.3001.7020) 和 @Component 区别

> 一句话概括就是 `@Configuration` 中所有带 `@Bean` 注解的方法都会被动态代理，因此调用该方法返回的都是同一个实例。
>
> Spring 注解中 @Configuration 和 @Component 的区别总结为一句话就是：
>
>         @Configuration 中所有带 @Bean 注解的方法都会被动态代理（cglib），因此调用该方法返回的都是同一个实例。而 @Conponent 修饰的类不会被代理，每实例化一次就会创建一个新的对象。
>
> 在 @Configuration 注解的源代码中，使用了 @Component 注解：
>
> 从定义来看， @Configuration 注解本质上还是 @Component，因此 <context:component-scan/> 或者 @ComponentScan 都能处理 @Configuration 注解的类。
>
> 下面我们通过一个例子来说明上述情况：
>
> // 使用@Configuration和@Bean注解创建Room实例和People实例，并注入进spring容器
> @Configuration
> public class RoomPeopleConfig {
>
> ```java
> @Bean
> public Room room() {
>     Room room = new Room();
>     room.setId(1);
>     room.setName("房间");
>     room.setPeople(people());// 在创建Room实例时，再调用一次People()创建一个People实例
>     return room;
> }
>  
> @Bean
> public People people() {
>     People people = new People();
>     people.setId(1);
>     people.setName("小明");
>     return people;
> }
> }
> ```
>
> 
> 
>
> ```java
> @SpringBootTest
> @ContextConfiguration(classes = Application.class)
> public class ConfigurationTests {
> @Autowired
> private Room room;
>  
> @Autowired
> private People people;
> ```
>
>  
>
> ```java
> @Test
> public void test() {
>     System.out.println(people == room.getPeople() ? "是同一个实例" : "不是同一个实例");
> }
> }
> ```
>
>
> 输出结果：同一个
>
> 如果将 @Configuration 换成 @Component ，则输出：不同
>
> 从上面的结果可以发现使用 @Configuration 时在 people 和 spring 容器之中的是同一个对象，而使用 @Component 时是不同的对象。这就是因为 @Configuration 使用了 cglib 动态代理，返回的是同一个实例对象。
>
> 虽然 @Component 注解也会当做配置类，但是并不会为其生成 CGLIB 代理 Class，所以在生成 room 对象时和生成 people 对象时调用 people( ) 方法执行了两次 new 操作，所以是不同的对象。当使用 @Configuration 注解时，生成当前对象的子类 Class，并对方法拦截，第二次调用 people（）方法时直接从 BeanFactory 之中获取对象，所以得到的是同一个对象

### 3、springboot配置文件的加载位置和优先顺序

springboot会按照下面优先级顺序寻找application.properties文件和application.yml文件。高优先级覆盖低优先级的文件内容。

file:./config 文件路径下和src同级别新建一个文件夹
file:./
classpath:/config/ 类路径下
classpath:
spring.config.location可以用来改变加载文件位置。在项目打包后，可以用命令行参数的形式指定配置文件的新位置。和默认加载的文件形成互补，共同起作用。
java -jar XXX.jar --server.port=8007



![image-20230704145759440](..\img\javabase\spring\配置文件加载顺序.png)

外部配置文件加载顺序，由jar包外向jar包内寻找配置文件，高优先级覆盖低优先级。
红色是file:路径，也就是项目根路径。
蓝色是classpath路径，也就是和类根目录平级的位置。

### 4、springboot自动装配

```java
主配置类启动，通过@SringBootApplication 中的@EnableAutoConfguration 加载所需的所有自动配置类，然后自动配置类生效并给容器添加各种组件。那么@EnableAutoConfguration 其实是通过它里面的@AutoConfigurationPackage 注解，将主配置类的所在包下面所有子包里面的所有组件扫描加载到 Spring 容器中; 还通过 @EnableAutoConfguration 里面的 AutoConfigurationImportSelector 选择器中的 SringFactoriesLoader.loadFactoryNames()方法，获取类路径下的 META-INF/spring.factories 中的资源并经过一些列判断之后作为自动配置类生效到容器中，自动配置类生效后帮我们进行自动配置工作，就会给容器中添加各种组件:这些组件的属性是从对应的 Properties 类中获取的，这些 Properties 类里面的属性又是通过@ConfigurationProperties 和配置文件绑定的，所以 我们能配置的属性也都是来源于这个功能的 Properties 类。SpringBoot 在自动配置很多组件的时候，先判断容器中有没有用户自己配置的(@Bean、@Component)如果有就用用户配置的，如果没有，才自动配置;如果有些组件可以有多个就将用户配置和默认配置的组合起来
```

