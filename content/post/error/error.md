---
title: error
date: '2025-12-28T09:38:23+08:00'

categories:
- error
---

# error

## spring
##### 1.feign异步调用接口丢失header
解决:1.在主线程中先获取请求头参数
    2.传入子线程中
    3.由子线程将请求头参数设置到上下文中
    4.最后在feign转发处理中拿到子线程设置的上下文请求头数据，转发到下游

请求头上下文：

```java
public class InheritableThreadLocalHeader {
private InheritableThreadLocalHeader() {
}
private static final InheritableThreadLocal<HashMap<String, String>> HEADER = new InheritableThreadLocal<>();

public static void clear() {
    HEADER.remove();
}

public static void set(HashMap<String, String> headers) {
    HEADER.set(headers);
}

public static HashMap<String, String> get() {
    return HEADER.get();
}}
```

获取上下文参数：

```java
public static Map<String, String> getHeaderMap() {
    Map<String, String> headerMap = Maps.newLinkedHashMap();
	try {
		ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
		if (requestAttributes == null) {
			return headerMap;
		}
	HttpServletRequest request = requestAttributes.getRequest();
	Enumeration<String> enumeration = request.getHeaderNames();
	while (enumeration.hasMoreElements()) {
		String key = enumeration.nextElement();
		String value = request.getHeader(key);
		headerMap.put(key, value);
	}
	} catch (Exception e) {
	log.error("《RequestContextUtil》 获取请求头参数失败：", e);
}
	return headerMap;
}
```

feign转发处理：

```java
@Slf4j
@Configuration
public class FeignRequestInterceptor implements RequestInterceptor {
   @Override
   @SneakyThrows
   public void apply(RequestTemplate requestTemplate) {
      log.debug("========================== ↓↓↓↓↓↓ 《FeignRequestInterceptor》 Start... ↓↓↓↓↓↓ ==========================");
      Map<String, String> threadHeaderNameMap = RequestHeaderHandler.getHeaderMap();
      if (!CollectionUtils.isEmpty(threadHeaderNameMap)) {
          threadHeaderNameMap.forEach((headerName, headerValue) -> {
          log.debug("《FeignRequestInterceptor》 多线程 headerName:【{}】 headerValue:【{}】", headerName, headerValue);
          requestTemplate.header(headerName, headerValue);
           });
         }
      Map<String, String> headerMap = RequestContextUtil.getHeaderMap();
      headerMap.forEach((headerName, headerValue) -> {
      log.debug("《FeignRequestInterceptor》 headerName:【{}】 headerValue:【{}】", headerName, headerValue);
      requestTemplate.header(headerName, headerValue);
    });
       log.debug("========================== ↑↑↑↑↑↑ 《FeignRequestInterceptor》 End... ↑↑↑↑↑↑ ==========================");
     }
}
```


在项目中：
登录拦截器：

```java
public class BaseLoginCheckInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        HashMap<String, String> headers = this.getHeaders(request);
        InheritableThreadLocalHeader.set(headers);
    }

    protected HashMap<String, String> getHeaders(HttpServletRequest request) {
        HashMap<String, String> map = new LinkedHashMap<>();
        Enumeration<String> enumeration = request.getHeaderNames();
        while (enumeration.hasMoreElements()) {
            String key = enumeration.nextElement();
            String value = request.getHeader(key);
            map.put(key, value);
        }
        return map;
    }
}

```


feign转发：

```java
@Configuration
public class FeignConfig implements RequestInterceptor {
@Value("${server.gatewayUrl}")
private String gatewayUrl;

@Bean
public Logger.Level feignLoggerLevel() {
    return Logger.Level.FULL;
}

//转发header
@Override
public void apply(RequestTemplate requestTemplate) {
    try {
        HashMap<String, String> headers = InheritableThreadLocalHeader.get();
        for (String headerName : headers.keySet()) {
            requestTemplate.header(headerName, headers.get(headerName));
        }
        if (requestTemplate.feignTarget().url().equals(String.format("http://%s", requestTemplate.feignTarget().name()))) {
            requestTemplate.target(gatewayUrl);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
}
}
```



##### 2.Springboot ResponseBodyAdvice全局统一返回类型为String时，类型转换异常

贴上代码

**advice**

```java
@ControllerAdvice
public class ResponseAdvice implements ResponseBodyAdvice<Object> {

    private final static String RETURN_CLASS = "ResponseResult";

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return returnType.getContainingClass().isAnnotationPresent(RespResult.class) || returnType.hasMethodAnnotation(RespResult.class);
    }

    @Override
    public Object beforeBodyWrite(Object data, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        String msg;
        RespResult respResult = returnType.getMethodAnnotation(RespResult.class);
        if (Objects.isNull(respResult)) {
            respResult = returnType.getContainingClass().getAnnotation(RespResult.class);
        }
        msg = respResult.value();

        if (Objects.nonNull(data) && Objects.nonNull(data.getClass())) {
            String simpleName = data.getClass().getSimpleName();
            if (RETURN_CLASS.equals(simpleName)) {
                return data;
            }
            return ResponseResult.success(msg, data);
        }
        return ResponseResult.success(msg);
    }
}
```

**异常**

```java
java.lang.ClassCastException: class com.vren.common.common.core.domain.ResponseResult cannot be cast to class java.lang.String (com.vren.common.common.core.domain.ResponseResult is in unnamed module of loader 'app'; java.lang.String is in module java.base of loader 'bootstrap')
	at org.springframework.http.converter.StringHttpMessageConverter.addDefaultHeaders(StringHttpMessageConverter.java:44)
	at org.springframework.http.converter.AbstractHttpMessageConverter.write(AbstractHttpMessageConverter.java:211)
	at org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor.writeWithMessageConverters(AbstractMessageConverterMethodProcessor.java:293)
	at org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.handleReturnValue(RequestResponseBodyMethodProcessor.java:183)
	at org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite.handleReturnValue(HandlerMethodReturnValueHandlerComposite.java:78)
	at 

```

跟踪断点进行排查

首先发生异常的位置

![image-20230627144251652](..\img\err\processbody.png)



我们可以发现这个`converter`是一个`StringHttpMessageConverter`类型，然后我们进入到`addDefaultHeaders`方法

![image-20230627144402067](..\img\err\write.png)



然后，当代码执行到这个方法的 **260 行**时，就会发生刚才我们说的那个`ClassCastException`异常。

![image-20230627144843579](..\img\err\classcast.png)



最后，我们终于找到了发生异常的原因，因为 260 的代码会执行getContentLength(t, headers.getContentType())这个方法，而这个方法会去执行StringHttpMessageConverter的getContentLength方法，如下图所示：

![image-20230627144946025](..\img\err\stringconvert.png)

但是这个时候 `t`的类型已经被我们用`beforeBodyWrite`方法转为`ResponseResult`类型了，所以就发生了类型转换异常的错误



**异常原因**‘

spring默认给我们注入了一些转换器

然后，我们通过源码就能发现，这个转换的执行过程是：
1.遍历所有的转换器，判断当前转换器能不能使用
2.如果可以使用，才调用我们写的`beforeBodyWrite`进行处理

![image-20230627145750264](..\img\err\messageConverters.png)

最后，我们就发现了原来是因为处理过后的body已经从String类型转为Result类型，然后在调用实现类即StringHttpMessageConverter#getContentLength(String str, @Nullable MediaType contentType)方法时，第一个参数str发生了类型转换错误的异常。


解决办法：

**第一种方法**

就是我们在我们自己写的`beforeBodyWrite`的方法中，将返回值用 Result 包装过后再将Result 转为 String 类型进行返回。

```java

 @Override
    public Object beforeBodyWrite(Object data, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        String msg;
        RespResult respResult = returnType.getMethodAnnotation(RespResult.class);
        if (Objects.isNull(respResult)) {
            respResult = returnType.getContainingClass().getAnnotation(RespResult.class);
        }
        msg = respResult.value();

        if (Objects.nonNull(data) && Objects.nonNull(data.getClass())) {
            String simpleName = data.getClass().getSimpleName();
            if (RETURN_CLASS.equals(simpleName)) {
                return data;
            }
            if (data instanceof String){
                //不加这句设置，返回结果类型会变成text/plain
                response.getHeaders().add("Content-Type", "application/json");
                try {
                    // 这里
                    return objectMapper.writeValueAsString(ResponseResult.success(msg,data));
                } catch (JsonProcessingException e) {
                    throw new RuntimeException(e);
                }
            }
            return ResponseResult.success(msg, data);
        }
        return ResponseResult.success(msg);
    }

```

**第二种方式：**

```java
@Configuration
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    	  // 第一种方式是将 json 处理的转换器放到第一位，使得先让 json 转换器处理返回值，这样 String转换器就处理不了了。
//        converters.add(0, new MappingJackson2HttpMessageConverter());
		  // 第二种就是把String类型的转换器去掉，不使用String类型的转换器
//        converters.removeIf(httpMessageConverter -> httpMessageConverter.getClass() == StringHttpMessageConverter.class);
    }
}
```



## db
##### 1.sql 同时（更新）update和（查询）select同一张表
当要使用本表的数据更新本表时，容易出错：

如下：

update b set aaa=select max(MAX_def_60M) as max from b

[Err] 1064 - You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'select max(MAX_def_60M) as max from b' at line 2

解决办法：再嵌套一层查询

update b set aaa=(select max from (select max(MAX_def_60M) as max from b) as temp) 不能同时读写的原因：mysql读写锁锁定的问题

若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。

写锁：

若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。这保证了其他事务在T释放A上的锁之前不能再读取和修改A。

加了共享锁的对象，可以继续加共享锁，不能再加排它锁。加了排它锁后，不能再加任何锁。

那么说我在更新一个表的时候，我锁定了一行，这一行我是不能加读锁的了，所以这时我查询这张表，就会出现这种问题。

加一层子查询之后成功的原因（待补充）：

mysql在from子句中遇到子查询时，先执行子查询并将结果放到一个临时表中，我们通常称它为“派生表”；临时表是没有索引、无法加锁的。

##### 2.bat批文件执行.sql脚本文件导入MySQL数据库乱码问题
往mysql数据库中导入sql文件，数据库中竟然显示乱码，数据库格式以及脚本文件都设置为utf-8。不知为什么会这样？

可能的原因：

使用可视化工具导出MySQL数据时，当数据量大时，导出不会错误，但导入时会出现错误，比如MySQL数据库导入SQL文件时出现乱码。
使用命令行导入数据时会出现如下这类的错误：
ERROR 1064 (42000) at line 1: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use /*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_BSD SET_CLIENT */’ at line 1
这是因为命令行模式下不能认出SQL文件格式造成，可以将SQL文件另存为UTF-8 NO BOM格式，然后进行导入。
另外在导入数据时，如果目标数据库或表是UTF-8字符集的， 而导入SQL中有中文，可能在最终结果中出现乱码
尝试了很多方法后，有了一个解决方案，如下：

此时只需在导入的SQL文件第一行加入如下内容即可：
/*!40101 SET NAMES utf8 */;



##### 3.排查死锁

首先使用mysql的 show processlist查看长时间运行的查询或阻塞的线程；==》  如果是死锁，mysql中，锁等待可通过data_lock_waits表查看；阻塞的sql，可以通过events_statements_current表查看；这两张表都存在于performance_schema数据库中,因此可以写个联表sql查看阻塞的sql：：

```sql
select request_e.sql_text wait_sql, w.blocking_thread_id, block_e.sql_text block_sql, block_t.processlist_host block_ip
from performance_schema.data_lock_waits w
join performance_schema.events_statements_current request_e on request_e.thread_id = w.requesting_thread_id
join performance_schema.events_statements_current block_e on block_e.thread_id = w.blocking_thread_id
join performance_schema.threads block_t on block_t.thread_id = w.blocking_thread_id;


```

==> 查找到 阻塞sql和主机ip ==》可以使用

```linux
sudo nmap -o -Pn ip
```

 查看是主机信息；linux还是windows



##### 4.group by问题

查询语句使用了group_concat或其他聚合函数，但没有使用group by 会生成一条空记录



### mybatis

##### 1.mybatis-plus版本冲突

Spring Boot  3.2 报错 Invalid value type for attribute ‘factoryBeanObjectType‘: java.lang.String;

原因：项目中使用 `mybatis-plus-boot-starter` 当前最新版本 3.5.4.1 ，其中依赖的 `mybatis-spring` 版本为 2.1.1；在 mybatis-spring 2.1.1 版本的 ClassPathMapperScanner#processBeanDefinitions 方法里将 `BeanClassName `赋值给 String 变量

；并将 `beanClassName` 赋值给 `factoryBeanObjectType`；但是在 Spring Boot 3.2 版本中FactoryBeanRegistrySupport#getTypeForFactoryBeanFromAttributes方法已变更，如果 factoryBeanObjectType 不是 ResolvableType 或 Class 类型会抛出 IllegalArgumentException 异常。

此时因为 factoryBeanObjectType 是 String 类型，不符合条件而抛出异常。

==解决方案：==Mybatis-Plus 于 2023年12月24日发布 [mybatis-plus v3.5.5](https://github.com/baomidou/mybatis-plus/releases/tag/v3.5.5) 版本，发布日志声明 `升级spring-boot3版本mybatis-spring至3.0.3`

所以升级 [Mybatis-Plus](https://so.csdn.net/so/search?q=Mybatis-Plus&spm=1001.2101.3001.7020) 版本为 3.5.5 版本即可，需要注意下 Maven 的坐标标识 是`mybatis-plus-spring-boot3-starter`，这点和SpringBoot 2 的依赖坐标`mybatis-plus-boot-starter`有所区别。

```java
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
            <version>3.5.5</version>
        </dependency>

```

也可以

```java
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-boot-starter</artifactId>
  <version>3.5.4.1</version>
  <exclusions>
      <exclusion>
          <artifactId>mybatis-spring</artifactId>
          <groupId>org.mybatis</groupId>
      </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis-spring</artifactId>
  <version>3.0.3</version>
</dependency>

```



## git

##### 1.可以访问，无法推送

事情是酱紫的，本楼主刚刚在 Rstudio 耕了一篇博客，然而没法推送到 github 仓库上，push 那一步会报错，错误如下。

```vbnet
ssh: connect to host github.com port 22: Connection timed out
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

俺试了下把 wifi 切换成手机热点，不行。试了下挂梯子，可以正常上 github，但是也还是 push 失败。上网搜一通，按照[这里的一个答案](https://stackoverflow.com/questions/15589682/ssh-connect-to-host-github-com-port-22-connection-timed-out)，把`url=git@...:...`改成`yrl=https://.../...`，也还是不行。当然，也试了重启电脑，也是不行滴。

不知有没有哪位小伙伴能想到别的撒招破解这个问题？

后来这个问题是这样解决的。

1. 执行`ssh -T git@github.com`，得到下面的结果，这可能是22这个端口不能用了。

   ```sql
   ssh: connect to host github.com port 22: Connection timed out
   ```

2. 执行`ssh -T -p 443 git@ssh.github.com`，回答完 yes 后会得到下面的内容，说明443这个端口可用。

   ```vbnet
   Hi earfanfan! You've successfully authenticated, but GitHub does not provide shell access.
   ```

3. 执行`vim ~/.ssh/config`，在打开的文件里面贴入下面的内容，然后摁 ESC 键，输入`:wq`保存，如果再执行`ssh -T git@github.com`，又可以看到 github 回复 Hi 那段，那就可以正常推送了。

   ```undefined
   Host github.com
     Hostname ssh.github.com
     Port 443
   ```
