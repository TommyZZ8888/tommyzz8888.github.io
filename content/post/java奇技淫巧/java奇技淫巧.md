---
title: 奇技淫巧
date: '2025-12-28T09:38:23+08:00'

categories:
- java 奇技淫巧
---


# java

#### 1、excel 公式生成sql实现快速导入

通过excel生成sql语句
有的时候业务部门直接甩过来一个excel表格让我们插入或者更新到数据库中。插入还好说，只要字段对应，就可以插入，但是更新呢？所以我们需要一个其他的操作方式，将excel生成想要的sql语句。

具体操作步骤

1.写好插入、更新语句，将指定位置替换成列序号。
① insert table into (name,age,sex) values(‘张三’,‘11’,‘1’);
② insert table into (name,age,sex) values(’"&A2&"’,’"&A2&"’,’"&A2&"’);
2.在excel中选择公式，在选择CONCATENATE将之前写好的语句放入CONCATENATE括号中。记得一定要有双引号，否则不是公式。
=CONCATENATE(“insert table into (name,age,sex) values(’”&A2&"’,’"&A2&"’,’"&A2&"’);")
3.鼠标点击生成的语句那个单元框的右下角向下拖动，即可生成



![image-20230413095207559.png](img%2Fimage-20230413095207559.png)


#### 2、相同类型的Bean可以注入同一个Map/List/Set

在SpringBoot开发中，当一个接口A有多个实现类时，spring会很智能的将bean注入到List<A>或Map<String,A>变量中。

一、将bean注入List或Map
举例说明如下：

步骤1：定义一个接口

```java
public interface IPerson {
	void doWork();
}
```

步骤2：对该接口做第一个实现类

```java
import org.springframework.stereotype.Component;

@Component("student")
public class StudentImpl implements IPerson {
@Override
public void doWork() {
	System.out.println("I am studying");
}
```

}
步骤3：对该接口做第二个实现类

```java
import org.springframework.stereotype.Component;

@Component("teacher")
public class TeacherImpl implements IPerson {
@Override
public void doWork() {
	System.out.println("I am teaching");
}
```

}
步骤4：使用@Autowired对List和Map进行注入使用

```java
import java.util.List;
import java.util.Map;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class PersonService {
@Autowired
List<IPerson> persons;
 
@Autowired
Map<String, IPerson> personMaps;
 
public void echo() {
	System.out.println("print list:");
	for (IPerson p : persons) {
		p.doWork();
	}
 
	System.out.println("\nprint map:");
	for (Map.Entry<String, IPerson> entry : personMaps.entrySet()) {
		System.out.println("Person:" + entry.getKey() + ", " + entry.getValue());
	}
}
```
}

步骤5：编写启动类调用PersonService的echo()函数进行测试

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;

@SpringBootApplication
public class Application {
public static void main(String[] args) {
	SpringApplication springApplication = new SpringApplication(Application.class);
	ApplicationContext context=springApplication.run(args);
	PersonService service=context.getBean(PersonService.class);
	service.echo();
}
```

程序运行结果为：

```
print list:
I am studying
I am teaching

print map:
Person:student, com.tang.aaa.StudentImpl@723e88f9
Person:teacher, com.tang.aaa.TeacherImpl@5f0fd5a0
```

二、策略模式：根据配置使用对应的实现类
对应Map的注入，key必须为String类型，即bean的名称，而value为IPerson类型的对象实例。

通过对上述Map类型的注入，可以改写为根据bean名称，来获取并使用对应的实现类。

举例如下：

步骤1：修改上述步骤4中的PersonService类如下：

```java
import java.util.Map;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class PersonService {
@Autowired
Map<String, IPerson> personMaps;
 
public void work(String name) {
	IPerson person=personMaps.get(name);
	if(null!=person) {
		person.doWork();
	}
}
```
步骤2：通过对PersonServer的work()传递不同的参数，实现对不同实现类的调用

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;

@SpringBootApplication
public class Application {
public static void main(String[] args) {
	SpringApplication springApplication = new SpringApplication(Application.class);
	ApplicationContext context=springApplication.run(args);
	PersonService service=context.getBean(PersonService.class);
	service.work("teacher");
}
```

我们可以使用service.work("teacher")或者service.work("student")来调用不同的实现类，即达到设计模式中策略模式的类似效果。
