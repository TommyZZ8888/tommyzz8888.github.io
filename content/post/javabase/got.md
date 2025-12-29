---
title: java 知识点
date: '2025-12-28T09:38:23+08:00'

categories:
- java 基础
---


# java知识点

https://github.com/Tyson0314/java-books



### 1、spring同类型多个bean的注入

Spring容器中的Bean之间会有很多依赖关系，在注入以来的时候，Spring需要知道明确知道注入的是哪一个Bean。

一、**类型冲突注入**

Spring容器中的Bean可以通过名称注入，或者通过类型注入。

**通过名称注入**

名称注入会指定一个明确的Bean的名称，容器不允许注册相同名称的Bean，所以不会有任何问题。

**通过类型注入**

通过类型注入的时候，有时会因为多个 Bean 的类型相同而产生冲突。例如：

- 同一类型注册多个不同名称的 Bean
- 抽象类型注册多个不同实现类的 Bean

这种情况下，容器不知道该注入哪个会抛出 `NoUniqueBeanDefinitionException` 异常。

#### 二、解决冲突

假设在项目中定义一个 `BeanService` 接口，基于该接口有三个实现类 `OneServiceImpl`、`TwoServiceImpl` 和 `ThreeServiceImpl`，三个实现类都由 Spring 容器管理。

![image-20230608161030464](..\img\javabase\got\beantest-结构.png)



##### 1. 注入主要的

注册 Bean 的时候，使用 `@Primary` 指定一个 Bean 为主要的，存在冲突时默认选择主要的 Bean。



```java
@Component
@Primary
public class OneServiceImpl implements BaseService{
}
```



##### 2. 注入指定的

注入 Bean 的时候，使用 `@Qualifier` 指定具体 Bean 的名称，通过名称注入解决冲突。

```java
public class BeanTestDemo {

    @Autowired
    @Qualifier("oneServiceImpl")
    private BaseService beanService;

    // ......
}
```

也可以直接通过**字段名称**来指定具体 Bean 的名称，来解决冲突。

```java
public class BeanTestDemo {

    @Autowired
    private BaseService oneServiceImpl;

    // ......
}
```

上面两种方法同样适用于构造器注入和 Setter 方法注入。



#### 三、注入多个 Bean

在实际应用中，如果需要注入符合类型的所有 Bean，可以使用集合类型来注入。

集合类型的注入同样适用于字段注入、构造器注入和 Setter 方法注入。

##### 1. 注入集合

通过数组来注入一种类型的所有 Bean。

```java
public class BeanTestDemo {

    @Autowired
    private BaseService[] baseServiceArr;

    // ......
}
```

通过 `List` 来注入一种类型的所有 Bean。

```java
public class BeanTestDemo {

    @Autowired
    private List<BaseService> baseServiceList;

    // ......
}
```

通过 `Set` 来注入一种类型的所有 Bean。

```java
public class BeanTestDemo {

    @Autowired
    private Set<BaseService> baseServiceSet;

    // ......
}
```

##### 2. 注入 Map

通过 `Map` 来注入一种类型的所有 Bean，Key 的类型固定为 `String`。

Key 存储 Bean 的名称，Value 存储 Bean 的实例。

```java
public class BeanTestDemo {

    @Autowired
    private Map<String, BaseService> baseServiceMap;

    // ......
}
```

##### 3. Bean 的顺序

注册 Bean 的时候可以使用 `@Order` 注解来指定 Bean 的权重（或顺序）。

在使用有序集合（数组或 `List`）注入的时候，会根据权重来排序。

```java
@Component("OneServiceImpl")
@Order(1)
public class OneServiceImpl implements BaseService{
    //......
}

@Component("TwoServiceImpl")
@Order(2)
public class TwoServiceImpl implements BaseService{
    //......
}

@Component("ThreeServiceImpl")
@Order(3)
public class ThreeServiceImpl implements BaseService{
    //......
}

```

上面配置类注册的 Bean 使用数组或 `List` 注入时，注入集合类型的元素顺序为：

```java
0 = {OneServiceImpl@1522} 
1 = {ThreeServiceImpl@1527} 
2 = {TwoServiceImpl@1528}
```

`@Order` 注解也可和 `@Component` 等注解一起使用。

#### 四、附录

##### 1. 常用注解

| 注解       | 描述                                                       |
| ---------- | ---------------------------------------------------------- |
| @Primary   | 指定主要的 Bean，存在注入冲突时默认注入的 Bean             |
| @Qualifier | 指定注入 Bean 的名称                                       |
| @Order     | 指定注册同类型的 Bean 的权重（或顺序），值越小，权重越大。 |



示例代码：https://github.com/TommyZZ8888/Java_Practice.git

sometest/bean
