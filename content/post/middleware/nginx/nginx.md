---
title: Nginx
date: '2025-12-28T09:38:23+08:00'

categories:
- 中间件
--- 
## Nginx

主要用于**http服务器**    **反向代理**    **负载均衡**



### **wbe服务器**

```nginx
events {
    //全局
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    server {
        listen       8081;
        server_name  localhost;

        //默认类型
        default_type text/html;

        //最底层的匹配 可以匹配任何url
        location / {
           echo "hello nginx";
        }

        // = 开头表示精确匹配
		location = /a {
		   echo " =/a";
		}
		
        //^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）。
		location ^~ /a {
			echo " =^~ /a";
		}
        
        location ^~ /a/b {
			echo " =^~ /a/b";
		}
		
        //~ 开头表示区分大小写的正则匹配 以/a-z开头的路径
		location ~ ^/[a-z] {
			echo " ~/[a-z]";
		}
		
		location ~ ^/\w {
			echo " ~/\w";
		}
		
		
    }

}

```

**匹配规则**

- =  ~ ^~ 普通文本四个优先级较高的先匹配
- 同优先级的，匹配程度较高的先匹配
- 匹配程度一样的，则写在前面的先匹配



##### 语法规则： location [=|~|~*|^~] /uri/ { … }

= 开头表示精确匹配

^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）。

~ 开头表示区分大小写的正则匹配

~* 开头表示不区分大小写的正则匹配

!~和!~*分别为区分大小写不匹配及不区分大小写不匹配 的正则

/ 通用匹配，任何请求都会匹配到。

多个location配置的情况下匹配顺序为（参考资料而来，还未实际验证，试试就知道了，不必拘泥，仅供参考）：

首先匹配 =，其次匹配^~, 其次是按文件中顺序的正则匹配，最后是交给 / 通用匹配。当有匹配成功时候，停止匹配，按当前匹配规则处理请求。

例子，有如下匹配规则：

**1**

```nginx
location = / {精确匹配，必须是127.0.0.1/

#规则A

}

location = /login {精确匹配，必须是127.0.0.1/login

#规则B

}

location ^~ /static/ {非精确匹配，并且不区分大小写，比如127.0.0.1/static/js.

#规则C

}

location ~ \.(gif|jpg|png|js|css)$ {区分大小写，以gif,jpg,js结尾

#规则D

}

location ~* \.png$ {不区分大小写，匹配.png结尾的

#规则E

}

location !~ \.xhtml$ {区分大小写，匹配不已.xhtml结尾的

#规则F

}

location !~* \.xhtml$ {

#规则G

}

location / {什么都可以

#规则H

}
```

那么产生的效果如下：

访问根目录/， 比如http://localhost/ 将匹配规则A

访问 http://localhost/login 将匹配规则B，http://localhost/register 则匹配规则H

访问 http://localhost/static/a.html 将匹配规则C

访问 http://localhost/a.gif, http://localhost/b.jpg 将匹配规则D和规则E，但是规则D顺序优先，规则E不起作用， 而 http://localhost/static/c.png 则优先匹配到 规则C

访问 http://localhost/a.PNG 则匹配规则E， 而不会匹配规则D，因为规则E不区分大小写。

访问 http://localhost/a.xhtml 不会匹配规则F和规则G，http://localhost/a.XHTML不会匹配规则G，因为不区分大小写。规则F，规则G属于排除法，符合匹配规则但是不会匹配到，所以想想看实际应用中哪里会用到。

访问 http://localhost/category/id/1111 则最终匹配到规则H，因为以上规则都不匹配，这个时候应该是nginx转发请求给后端应用服务器，比如FastCGI（php），tomcat（jsp），nginx作为方向代理服务器存在。

###### 所以实际使用中，个人觉得至少有三个匹配规则定义，如下：

```nginx
#这里是直接转发给后端应用服务器了，也可以是一个静态首页

# 第一个必选规则

location = / {

proxy_pass http://tomcat:8080/index

}

# 第二个必选规则是处理静态文件请求，这是nginx作为http服务器的强项

# 有两种配置模式，目录匹配或后缀匹配,任选其一或搭配使用

location ^~ /static/ {

root /webroot/static/;

}

location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {

root /webroot/res/;

}

#第三个规则就是通用规则，用来转发动态请求到后端应用服务器

#非静态文件请求就默认是动态请求，自己根据实际把握

#毕竟目前的一些框架的流行，带.php,.jsp后缀的情况很少了

location / {

proxy_pass http://tomcat:8080/

}

#直接匹配网站根，通过域名访问网站首页比较频繁，使用这个会加速处理，官网如是说。

#这里是直接转发给后端应用服务器了，也可以是一个静态首页

# 第一个必选规则

location = / {

proxy_pass http://tomcat:8080/index

}

# 第二个必选规则是处理静态文件请求，这是nginx作为http服务器的强项

# 有两种配置模式，目录匹配或后缀匹配,任选其一或搭配使用

location ^~ /static/ {

root /webroot/static/;

}

location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {

root /webroot/res/;

}

#第三个规则就是通用规则，用来转发动态请求到后端应用服务器

#非静态文件请求就默认是动态请求，自己根据实际把握

#毕竟目前的一些框架的流行，带.php,.jsp后缀的情况很少了

location / {

proxy_pass http://tomcat:8080/

}
```



### 反向代理 负载均衡

```nginx
events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    server {
        listen       8081;
        server_name  localhost;

        default_type text/html;

        location / {
           echo "hello nginx!";
        }
		
        location /a/ {
           proxy_pass https://www.baidu.com/;
        }

		location /b {
           proxy_pass https://www.baidu.com/;
        }

    }
}
```

**反向代理小结**

```
        location /a {
           proxy_pass https://ip;
        }

		//注意 /b后的反斜杠 和 ip后的反斜杠
		location /b/ {
           proxy_pass https://ip/;
        }
      上述配置会导致
      /a/x -> https://ip/a/x
      /b/x -> https://ip/x
        
```



```
events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
	#keepalive_timeout  65;
	
	upstream group1{
		server www.baidu.com weight=1;
		server cn.bing.com weight=10;
	}

    server {
        listen       8081;
        server_name  localhost;

        default_type text/html;
		
		location / {
			proxy_pass https://group1;
		}

    }
}
```







## [1.正向代理和反向代理](https://lanmei123.github.io/blog/server/nginx.html#_1-正向代理和反向代理)

参考博客 [漫画图解](https://cloud.tencent.com/developer/article/1418457)
这个概念很容易混淆,先看下面这个图，区别还是挺明显的。
![img.png](..\..\img\middleware\nginx\代理.png)

虽然正向代理服务器和反向代理服务器所处的位置都是客户端和真实服务器之间，所做的事情也都是把客户端的请求转发给服务器，再把服务器的响应转发给客户端，但是二者之间还是有一定的差异的。

1、正向代理其实是客户端的代理，帮助客户端访问其无法访问的服务器资源。反向代理则是服务器的代理，帮助服务器做负载均衡，安全防护等。
2、正向代理一般是客户端架设的，比如在自己的机器上安装一个代理软件。而反向代理一般是服务器架设的，比如在自己的机器集群中部署一个反向代理服务器。
3、正向代理中，服务器不知道真正的客户端到底是谁，以为访问自己的就是真实的客户端。而在反向代理中，客户端不知道真正的服务器是谁，以为自己访问的就是真实的服务器。
4、正向代理和反向代理的作用和目的不同。正向代理主要是用来解决访问限制问题。而反向代理则是提供负载均衡、安全防护等作用。二者均能提高访问速度。

## [2.配置文件导读](https://lanmei123.github.io/blog/server/nginx.html#_2-配置文件导读)

### [2.1 单机前后端文件](https://lanmei123.github.io/blog/server/nginx.html#_2-1-单机前后端文件)

理想的情况如下 ![img.png](..\..\img\middleware\nginx\单机.png)

nginx配置



```
# 只启动一个工作进程
worker_processes  1;    

# 每个工作进程的最大连接为1024                         
events {
    worker_connections  1024;               
}

http {
    # 引入MIME类型映射表文件
    include       mime.types;        
    # 全局默认映射类型为application/octet-stream            
    default_type  application/octet-stream;   

    server {
        # 监听80端口的网络连接请求
        listen       80;            
        # 虚拟主机名为localhost                  
        server_name  localhost;   
                 
        # 匹配前端访问的路径
        # 比如 访问localhost:80/frontend这个路由,nginx会匹配到这个路径,然后在alias的路径中找前端的界面，然后返回给浏览器
        location /frontend/ {
            alias  /data/view/frontend/;
            index index.html index.htm;
        }
        # 如果前端访问后端发起的http请求是localhost:80/backend/
        location /backend/{
             proxy_set_header Host $http_host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header REMOTE-HOST $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             #核心配置代理接口到jar包的8080的项目上
             proxy_pass localhost:8080;
             proxy_set_header Upgrade $http_upgrade; 
        }           
    }
}
```

### [2.2 nginx高可用(主从 / 负载均衡)](https://lanmei123.github.io/blog/server/nginx.html#_2-2-nginx高可用-主从-负载均衡)

实现nginx高可用需要多台服务器



```
定义:
    10.0.0.0 作为负载均衡服务器  
    11.0.0.1 作为应用服务器A  
    11.0.0.1 作为应用服务器B
```

实现效果 ![img.png](F:\java_study\github\Tommy\notes\img\middleware\nginx\高可用.png) 10.0.0.0 的负载均衡的nginx配置



```
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    upstream balanceServer {
        server 11.0.0.0:80;
        # 这里是主从配置,去掉backup就是轮询,可以设置weigh作为轮询的权重
        server 12.0.0.0:80 backup;
    }
    server {
        listen       80;
        server_name  10.0.0.0;

        location /frontend {
            proxy_pass   http://balanceServer/frontend/;
        }

        location /backend {
            proxy_pass   http://balanceServer/backend/;
        }
    }
}
```

11.0.0.0和12.0.0.0的负载均衡的nginx配置 两个配置文件一样



```
# 只启动一个工作进程
worker_processes  1;    

# 每个工作进程的最大连接为1024                         
events {
    worker_connections  1024;               
}

http {
    # 引入MIME类型映射表文件
    include       mime.types;        
    # 全局默认映射类型为application/octet-stream            
    default_type  application/octet-stream;   

    server {
        # 监听80端口的网络连接请求
        listen       80;            
        # 虚拟主机名为localhost                  
        server_name  localhost;   
                 
        # 匹配前端访问的路径
        # 比如 访问localhost:80/frontend这个路由,nginx会匹配到这个路径,然后在alias的路径中找前端的界面，然后返回给浏览器
        location /frontend/ {
            alias  /data/view/frontend/;
            index index.html index.htm;
        }
        # 如果前端访问后端发起的http请求是localhost:80/backend/
        location /backend/{
             proxy_set_header Host $http_host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header REMOTE-HOST $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             #核心配置代理接口到jar包的8080的项目上
             proxy_pass localhost:8080;
             proxy_set_header Upgrade $http_upgrade; 
        }           
    }
}
```

## [3.Docker安装 (还需完善)](https://lanmei123.github.io/blog/server/nginx.html#_3-docker安装-还需完善)

推荐使用docker安装，简单快捷，需要注意2点

1. nginx配置文件的里面写的路径是容器nginx中的路径，而不是服务器本身的路径
2. nginx配置文件里面的开放端口，需要在docker中也开放，否则无法访问



```
# 创建拉取容器
docker pull nginx

# 创建挂载路径
mkdir -p /data/docker/project-volumes/nginx.conf
mkdir -p /data/docker/project-volumes/logs
mkdir -p /data/nginx/project-volumes/html

docker run -d \
  --name nginx \
  --restart always \
  -p 9467:9467 \
  -v /data/docker/project-volumes:/data/docker/project-volumes \
  -v ./conf/nginx.conf:/etc/nginx/nginx.conf \
  -v ./nginx/logs:/var/log/nginx \
  -v ./nginx/conf.d:/etc/nginx/conf.d \
  --build-arg context=. \
  --build-arg dockerfile=nginx-dockerfile \
  nginx
  
  
```
