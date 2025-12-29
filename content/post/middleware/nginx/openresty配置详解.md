
---
title: openresty配置详解
date: '2025-12-28T09:38:23+08:00'

categories:
- 中间件
---
- # 概述：一共三大部分配置。

# 其中#注释掉的可以在需要的时候开启并修改，没有注释掉的（除了下面location示例）不要删掉，基本都是必须的配置项。

###############################第一部分 全局配置############################

```
#user  nobody;                        指定启动进程的用户，默认不用指定即可。
#error_log  logs/error.log;           配置日志输出，虽然叫error_log但是可以定义输出的级别，默认不写是ERROR级别
#error_log  logs/error.log  notice;   
#error_log  logs/error.log  info;
#pid        logs/nginx.pid;           记录pid的文件，默认就是放到这个位置，可以修改。
```



# 只启动一个进程，nginx是多进程单线程模型，但是使用了epoll sendfile 非阻塞io

worker_processes  1;

###############################第二部分 event配置############################
#主要是网络连接相关的配置
events {

  # 每个worker能连接1024个链接

```
  worker_connections  1024;
  #use epoll; 事件驱动模型select|poll|kqueue|epoll|resig
}
```

###############################第三部分 http配置############################

​    

    http {
      include       mime.types;  #文件扩展名与文件类型映射表
      default_type  text/html;   #默认的返回类型，可以在server.location里面改
      sendfile        on;        #开启sendfile系统调用
      keepalive_timeout  65;     #连接超时时间65s


​    

      server {
        listen       80;
        # 下面展示多个demo，demo之间互相没有依赖关系，可以单独配置来进行测试。
        # 其中demo1到demo6 是nginx相关的。
    
    ############### demo1 展示location路径的不同写法优先级 ###############
    # =最高优先级 表示路径完全等于，可以匹配/demo1/a/b的请求
    location =/demo1/a/b {
      echo "=/demo1/a/b";
    } 
    # ^~第二高  表示startsWith，可以匹配/demo1/a/b/c和/demo1/abc请求
    location ^~/demo1/a {
      echo "^~/demo1/a";
    }
    # ~等四个符号第三高  表示正则，如果要用{}是特殊符号，需要整个加引号，建议直接加双引号，防止出错，可以匹配/demo1/bcd
    # 其他三个：~*不区分大小写正则，!~正则不匹配的，!~*不分大小写的正则不匹配
    location "~/demo1/\w{3}$" {
      echo "regex";
    }
    # 最低 没有前置符号 /demo1 /demo111 /demo1/b/c 不符合上面三种，就会匹配到这
    location /demo1{
      echo "/demo1";
    }
    
    ############### demo2 展示serve静态文件夹 ###############
    location / {
       root   html;                 # root就是根目录是当前html目录
       index  index.html index.htm; # index表示默认不写的时候转到的文件
    }
    
    ############## demo3 指定错误文件 ###############
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
       root   html;
    }
    
    ############# demo4 rewrite重写url rewrite也可以是server级别 ####################
    location /demo4 {
      # 一般放到最后一行
      rewrite ^/(.*) /$1/api permanent; # permanent301, redirect302, break不在匹配后面rewrite规则，last继续向下匹配。
    }
    location /demo4/api {
      echo "/demo4/api";
    }
    
    ############# demo5 demo6 proxy_pass反向代理 ####################
    # /demo5 => baidu.com/demo5
    # /demo5/a/b => baidu.com/demo5/a/b
    location /demo5 {
      proxy_pass  https://www.baidu.com;
    }
    # /demo6 => baidu.com
    # /demo6/a/b => baidu.com/a/b
    location /demo6 {
      # proxy_set_header Host $http_host; 如果有请求头改动的需求可以搜索proxy_set_header去了解
      proxy_pass  https://www.baidu.com/;
    }

  }

  # 下面demo7到demo11是openresty lua的一些配置demo

      server {
        listen       81;


​    

    ############# demo7 init_by_lua_block 用来加载经常用到的库 或者 用来对多进程shared变量赋值 ####################
    init_by_lua_block {
      cjson = require("cjson")       --后续的lua流程中可以直接使用cjson
      local myname = ngx.shared.info --可以认为是静态变量，通过info:get获取变量值
      info:set("name", "frank")
      info:set("age", 77)
    }
    
    ############# demo8 demo9 rewrite_by_lua_block 配合ngx.redirect用来替换rewrite指令 ####################
    # 注意rewrite_by_lua和因为作用阶段是nginx原生rewrite之后，所以容易和原生一起用的时候出错，最好的方式就是只用lua的不要用nginx的了。
    location /demo8 {
      set $a 1;
      set $b "";
      rewrite_by_lua_block {
        ngx.var.b = tonumber(ngx.var.a) + 1
        if tonumber(ngx.var.b) == 2 then
          return ngx.redirect("/demo9") --默认是302，如果要301或307可以再加一个第二参数即可
        end
      }
      echo "demo8"; # 注意echo是content阶段的，rewrite阶段重定向了请求，就走不到这里了
    }
    location /demo9 {
      echo "demo9";
    }
    ############# demo10 access_by_lua_block 用来做一些加载内容前的准备工作例如访问redis看看用户身份是不是合法 ip是不是合法等 ####################
    location /demo10 {
      access_by_lua_block {
        local res = ngx.location.capture("/auth") -- ngx.location.capture是作为客户端发起http请求拿到结果
        if res.status == ngx.HTTP_OK then
          return  -- 正常return就能走到content阶段
        end
        if res.status == ngx.HTTP_FORBIDDEN then
          ngx.exit(res.status) -- exit + 状态码 就直接返回状态码了
        end
        ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
      }
      echo "demo10"; # 如果合法的话就返回demo10字样
    }
    location /auth {
      return 200; # 换成403 or 500试试
    }
    
    ############# demo10 content_by_lua_block 用来作为content阶段的脚本，一般测试用的多 ####################
    #不要和 echo proxy_pass等content阶段指令一起用
    location /demo10 {
      content_by_lua_block{
        ngx.say("/demo10");
        ngx.say("/demo11"); -- 和外部用俩echo效果类似。ngx.say ngx.print区别是前者会多个回车在最后
      }
      # echo "echo10";   如果外面用了echo，则只有echo的效果
      # proxy_pass http://www.baidu.com; 如果外面用了proxy_pass也是只有proxy_pass效果了，因为都是content阶段，content只能一个生效。
    }
    
    ############# demo11 rewrite_by_lua与proxy_pass配合 根据参数进行转发 ####################
    location /demo11 {
    	  default_type text/html;
    		set $proxy "";
      rewrite_by_lua '            # 千万别用content，因为content和proxy_pass阶段犯冲
    			local h = ngx.var.host    # 这里从host中提出第一个.之前的部分看是不是a来决定转发到哪
    			local dot = h:find("%.")
    			local prefix = h:sub(1,dot-1)
    			if prefix == "a" then
    				ngx.var.proxy="127.0.0.1:3000"
    			else
    				ngx.var.proxy="127.0.0.1:5500"
    			end
      ';
    		proxy_pass http://$proxy$uri;
    }
      }
    }


### ngx

上面的配置中已经看到ngx的一些属性和方法了，例如say print可以作为content返回，redirect可以实现跳转，ngx.location.capture可以请求一个url，一般是内部url。exit可以返回状态码，shared可以作为一个所有进程共享的缓存kv池。

再来看看其他的一些常见用法。

### log

ngx.log(ngx.INFO, "日志内容") 级别有ngx.STDERR ngx.EMERG ngx.ALERT ngx.CRIT ngx.ERR ngx.WARN ngx.NOTICE ngx.INFO ngx.DEBUG

### var与ctx

ngx.var可以获取或者修改nginx上下文的$xx变量，包括内置的变量，和在location范围内set的变量。nginx上下文的变量参考

一个request上下文的ngx.ctx是一个table可以自由的赋值和获取。例如rewrite_by_lua中ngx.ctx.a=10, content_by_lua中能拿到ngx.ctx.a。

```
location.capture
local res = ngx.location.capture('/foo?a=1&b=3&c=%3a') -- get请求, res有status body header和truncated属性

ngx.req.set_header("Content-Type", "application/json")
ngx.location.capture('/foo/bar',{ method = ngx.HTTP_POST, body = cjson.encode({a = 'hello, world' })) --post
req
ngx.req.get_method() 返回大写的字符串GET
ngx.req.get_headers() 返回 kv table
ngx.req.set_header(k, v)
ngx.req.get_body_data() 返回是string
ngx.req.get_uri_args() get查询参数，返回table
ngx.req.get_post_args() form类型的post，返回table,err
```


### hash与编码

```
newstr = ngx.encode_base64(str, no_padding?)
newstr = ngx.decode_base64(str)
digest = ngx.md5(str)
str = ngx.encode_args(table)
table = ngx.decode_args(str, max_args?)
newstr = ngx.escape_uri(str)
newstr = ngx.unescape_uri(str)
```


### re正则（个人感觉lua自带的正则已经够用了）

```
local m, err = ngx.re.match("hello, 1234", "[0-9]{3}") -- m[0] == "123" 这个正则比lua自带的正则强一点
local from, to, err = ngx.re.find(s, "([0-9]+)", "jo")
local iterator, err = ngx.re.gmatch("hello, world!", "([a-z]+)", "i")
local newstr, n, err = ngx.re.sub("hello, 1234", "([0-9])[0-9]", "[$0][$1]")
```


### os相关的

```
ngx.sleep(n) 单位是s
ngx.time() ngx.utctime()ngx.now() ngx.today()... 时间相关的
ngx.exec("shell")
```

