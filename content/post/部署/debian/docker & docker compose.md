---
title: Docker
date: '2025-12-28T09:38:23+08:00'

categories:
  - 部署
---
# Docker

### 1. 安装docker：

参考：https://www.wonote.com/post/debian-12-xi-tong-nei-an-zhuang-docker-xiang-xi-bu-zou/

也可参考： [Debian12 安装Docker 和 docker-compose - 陈伦刚的个人博客](https://chenlungang.com/?p=bc50b235-7f68-4489-946c-bd59776f64f8)

```sh
# 更新现有的软件包和包缓存
sudo apt update
sudo apt upgrade

# 安装依赖包  安装一些需要的依赖包，这些包允许 apt 使用 HTTPS 协议来访问 Docker 仓库
sudo apt install apt-transport-https ca-certificates curl software-properties-common

# 添加 Docker 官方 GPG 密钥
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -

# 添加 Docker 存储库
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"

# 再次更新软件包列表
sudo apt update

# 安装 Docker 社区版
sudo apt install docker-ce

# 验证
sudo systemctl status docker

# 添加用户到 docker 组（可选）
sudo usermod -aG docker $USER

# 重启
systemctl restart docker
```

也可以使用这种方式：

参考： [Debian12安装Docker及docker-compose教程 - 有一本小书](http://www.tanorqi.cn/archives/wei-ming-ming-wen-zhang#title-4)

```sh
# 更新软件包索引，确保可以获取到最新的软件包列表和版本信息
sudo apt-get update

# 安装必要的工具和依赖项
# ca-certificates 用于管理 SSL 证书，确保安全的网络通信
# curl 是一个用于传输数据的工具，常用于从网络下载文件
# gnupg 用于处理 GPG 密钥，确保软件源的签名验证
# software-properties-common 提供了管理软件源的工具
sudo apt-get install ca-certificates curl gnupg software-properties-common

# 创建一个具有指定权限的目录，用于存储 APT 密钥环
# -m 0755 表示设置目录权限为 0755，即所有者有读写执行权限，组用户和其他用户有读和执行权限
# -d 表示创建目录
# /etc/apt/keyrings 是存储 APT 密钥环的目录
sudo install -m 0755 -d /etc/apt/keyrings

# 从华为云的 Docker 官方软件源下载 GPG 密钥，并将其添加到系统的 APT 密钥管理系统中
# -fsSL 是 curl 的选项，-f 表示失败时不显示 HTTP 错误信息，-s 表示静默模式，-S 表示即使在 -s 模式下也显示错误信息，-L 表示跟随重定向
# apt-key add - 表示从标准输入读取 GPG 密钥并添加到系统中
curl -fsSL https://mirrors.huaweicloud.com/docker-ce/linux/debian/gpg | sudo apt-key add -

# 向系统的软件源列表中添加 Docker 官方的 Debian 软件源
# deb 表示这是一个 Debian 风格的软件源
# [arch=amd64] 表示该软件源仅适用于 64 位架构的系统
# https://mirrors.huaweicloud.com/docker-ce/linux/debian 是软件源的基础 URL
# $(lsb_release -cs) 是当前系统的发行版代号，如 buster、bullseye 等
# stable 表示使用稳定版本的软件包
sudo add-apt-repository "deb [arch=amd64] https://mirrors.huaweicloud.com/docker-ce/linux/debian $(lsb_release -cs) stable"

# 再次更新软件包索引，以包含新添加的 Docker 软件源
sudo apt-get update
安装docker-ce
# 使用 apt-get 命令安装 Docker 相关组件，-y 参数表示在安装过程中遇到询问时自动回答 "yes"
# docker-ce 是 Docker 社区版，提供了完整的 Docker 功能
# docker-ce-cli 是 Docker 社区版的命令行工具，用于与 Docker 守护进程交互
# containerd.io 是一个容器运行时，负责管理容器的生命周期
# docker-compose-plugin 是 Docker Compose 插件，用于定义和运行多容器的 Docker 应用
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 验证 Docker 是否成功安装
# 执行 docker -v 命令会输出 Docker 的版本信息
# 如果能正常输出版本号，说明 Docker 已成功安装
docker -v
```



/etc/docker/daemon.json

```js
# 添加下面内容到 daemon.json 中
# 此 JSON 配置文件包含了 Docker 的一些重要配置参数
{
    # registry-mirrors 用于配置 Docker 镜像源
    # 国内网络访问 Docker 官方镜像源可能较慢，配置国内镜像源可以加速镜像下载
    # 以下是多个国内镜像源地址，Docker 会依次尝试从这些镜像源下载镜像
    "registry-mirrors": [
        "https://ccr.ccs.tencentyun.com",
        "https://docker.rainbond.cc",
        "https://elastic.m.daocloud.io",
        "https://elastic.m.daocloud.io",
        "https://docker.m.daocloud.io",
        "https://gcr.m.daocloud.io",
        "https://ghcr.m.daocloud.io",
        "https://k8s-gcr.m.daocloud.io",
        "https://k8s.m.daocloud.io",
        "https://mcr.m.daocloud.io",
        "https://nvcr.m.daocloud.io",
        "https://quay.m.daocloud.io"
    ],
    # log-driver 指定 Docker 容器日志的驱动类型
    # json-file 表示使用 JSON 文件来存储容器日志
    "log-driver": "json-file",
    # log-opts 用于配置日志驱动的选项
    "log-opts": {
        # max-size 指定每个日志文件的最大大小
        # 这里设置为 10m，表示每个日志文件最大为 10MB
        "max-size": "10m",
        # max-file 指定日志文件的最大数量
        # 这里设置为 3，表示最多保留 3 个日志文件
        "max-file": "3"
    }
}
```



### 2.安装 docker compose

参考： [Debian12安装Docker及docker-compose教程 - 有一本小书](http://www.tanorqi.cn/archives/wei-ming-ming-wen-zhang#title-4)

```sh
# 手动下载：docker-compose-linux-x86_64手动下载： 地址：https://github.com/docker/compose/releases  ==》离线下载linux-x86_64 执行包docker-compose-linux-x86_64

# 将其执行文件改名放到/usr/local/bin下
sudo mv docker-compose-Linux-x86_64 /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version

```



### 3.常用命令

```sh
# 查看所有镜像
docker images

# 查询可用镜像
docker search [镜像名]

# 拉取镜像
docker pull mysql:8.0.23

# 根据Dockerfile创建镜像 docker build -t [镜像名称:标签] [Dockerfile路径]
docker build -t my-web-app:1.0 .

# docker rmi [镜像ID或名称:标签]
docker rmi my-web-app:1.0

# docker run [OPTIONS] [镜像名称:标签] [COMMAND]
docker run -p 3307:3306 --name mysql -v /usr/local/docker/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.0.23

# 查看当前允许容器，docker ps -a查看所有容器
docker ps

# 停止容器
docker stop [容器ID或名称]

# 启动容器
docker start [容器ID或名称]

# 重启容器
docker restart [容器ID或名称]

# 删除容器
docker rm [容器ID或名称]

# 进入容器
docker exec -it [容器ID或名称] [COMMAND]

# 查看容器的日志输出 -f 或 --follow: 实时跟踪日志输出
docker logs [容器ID或名称]

# 将容器的当前状态保存为一个新的Docker 镜像
docker commit my_container my_new_image
运行docker run -d -p 外部：内部 my_new_image

# 将一个或多个 Docker 镜像保存到一个 tar 归档文件中，以便在其他环境中分发或备份
docker save -o multiple_images.tar myimage:latest anotherimage:latest

# 用于从由 docker save 命令生成的 tar 文件中加载 Docker 镜像。它可以将存档中的镜像和所有层加载到 Docker 中，使其可以在新环境中使用。
docker load -i myimage.tar
```

![image-20250718171948360](..\..\img\部署\docker.png)





==Dockerfile==参数

```sh
最基本的几个指令为FROM，RUN，COPY，ADD，CMD，ENTRYPOINT,WORKDIR

FROM 表示基于哪个镜像构建
RUN 表示运行shell命令
COPY 表示复制文件到镜像中
ADD 与COPY类似，稍有区别
CMD 指定镜像的启动脚本
ENTRYPOINT 与CMD类似，稍有区别
WORKDIR 指定运行的主目录
EXPOSE 暴露端口
VOLUME 映射文件
ENV 参数


FROM alpine
RUN echo 'hello'
ENV A=123
COPY .  /code
WORKDIR /code
EXPOSE 80         # 暴露端口
VOLUME /usr/nginx # 映射文件
CMD ./code/server



# Use a multi-stage build for a smaller final image

FROM openjdk:17
WORKDIR /app
COPY api.jar api.jar

EXPOSE 8080
ENV JAVA_OPTS="-Xmx512m -Xms256m" \
    SPRING_PROFILES_ACTIVE=prod

ENTRYPOINT ["java", "-jar", "api.jar"]
```

==docker run==参数

```sh
docker run [参数] 镜像名:版本 [运行的shell指令]

-d                           后台运行
-e MYSQL_ROOT_PASSWOR=123456 环境变量
-p 80:80                     端口映射
-it                          阻塞运行
-v /suzhu:/rongqi            文件[夹]映射
Copy
--link mysql:mysql           链接其他容器
--rm                         运行结束后自动销毁
--name                       指定容器运行后的名字
--restart always             机器重启后是否自动运行
```



### 4.镜像下载 & 运行

https://hub.docker.com/

mysql参考： https://juejin.cn/post/7031188920511660063

nginx参考：https://www.cnblogs.com/msjhw/p/17420830.html

```sh
# jdk
docker pull openjdk:17


====================================================================

# mysql
docker pull mysql:8.0.23

mkdir /usr/local/docker/mysql/mysql-files
mkdir /usr/local/docker/mysql/conf
mkdir /usr/local/docker/mysql/logs
mkdir /usr/local/docker/mysql/data

# 需要在conf目录下 新建my.cnf, 进行如下配置（可选，但最好配置下表名忽略大小写）
[mysqld]
max_connections = 1000
wait_timeout = 7200
interactive_timeout = 7200
# 1、需要保证这个datadir目录下为空；
# 2、需要保证对datadir目录赋予mysql 权限，执行命令：chown -R mysql:mysql /data/mysql
# datadir=/var/lib/mysql
bind-address=0.0.0.0
default_authentication_plugin=mysql_native_password
# 排除掉 NO_ZERO_DATE and NO_ZERO_IN_DATE
sql_mode=''
# 表名忽略大小写
lower_case_table_names = 1
# 只输出 error 级别的日志
log_error_verbosity = 1
default-time-zone = '+08:00'
[mysql.server]
default-character-set = utf8
[mysqld_safe]
default-character-set = utf8
[client]
default-character-set = utf8


docker run -p 3307:3306 --name mysql \
-v /usr/local/docker/mysql/mysql-files:/var/lib/mysql-files \
-v /usr/local/docker/mysql/conf:/etc/mysql \
-v /usr/local/docker/mysql/logs:/var/log/mysql \
-v /usr/local/docker/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
-d mysql:8.0.23

# 一行命令
docker run -p 3307:3306 --name mysql -v /usr/local/docker/mysql/mysql-files:/var/lib/mysql-files -v /usr/local/docker/mysql/conf:/etc/mysql -v /usr/local/docker/mysql/logs:/var/log/mysql -v /usr/local/docker/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.0.23


=====================================================================

# redis
docker pull redis:7.2-alpine

docker run -d -p 6380:6379 --name redis redis --requirepass 123456


==================================================================

# nginx
docker pull nginx

# 创建挂载目录  启动前需要先创建Nginx外部挂载的配置文件（ /home/nginx/conf/nginx.conf）
# 之所以要先创建 , 是因为Nginx本身容器只存在/etc/nginx 目录 , 本身就不创建 nginx.conf 文件
# 当服务器和容器都不存在 nginx.conf 文件时, 执行启动命令的时候 docker会将nginx.conf 作为目录创建 , 这并不是我们想要的结果 。
mkdir -p /home/nginx/conf
mkdir -p /home/nginx/log
mkdir -p /home/nginx/html

# 将容器nginx.conf文件复制到宿主机
docker cp nginx:/etc/nginx/nginx.conf /home/tom/app/nginx/conf/nginx.conf
# 将容器conf.d文件夹下内容复制到宿主机
docker cp nginx:/etc/nginx/conf.d /home/tom/app/nginx/conf/conf.d
# 将容器中的html文件夹复制到宿主机
docker cp nginx:/usr/share/nginx/html /home/tom/app/nginx/

# 直接执行docker rm nginx或者以容器id方式关闭容器
# 找到nginx对应的容器id
docker ps -a
# 关闭该容器
docker stop nginx
# 删除该容器
docker rm nginx<br># 删除正在运行的nginx容器<br>docker rm -f nginx

# 运行容器
docker run -d -p 9002:80 --name nginx -v /home/tom/app/nginx/conf/nginx.conf:/etc/nginx/nginx.conf -v /home/tom/app/nginx/conf/conf.d:/etc/nginx/conf.d -v /home/tom/app/nginx/log:/var/log/nginx -v /home/tom/app/nginx/html:/usr/share/nginx/html nginx:latest

```

conf是主文件，conf.d配置各种server

**主配置文件** (`nginx.conf`)

```nginx
# 必须包含以下指令才能加载conf.d
http {
    include /etc/nginx/conf.d/*.conf;
    # 其他全局配置...
}
```

**片段配置** (`conf.d/app.conf`)

```nginx
server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://app:8080;
    }
}
```



### 5.项目部署

目录结构：

```bash
/home/tom/app目录下 存放 easy.jar application.yml application-dev.yml Dockerfile
```



Dockerfile文件， 创建镜像命令：docker build -t easyapi . /bin/bash 

```dockerfile
FROM openjdk:17
WORKDIR /home/tom/app/
CMD ["java", "-jar", "easy-api.jar"]
```

```sh
/home/tom/app/nginx/html下存放dist目录下的文件

# springboot项目启动命令
docker run -d -p 8090:8090 -v /home/tom/app/docker/easy-api.jar:/home/tom/app/easy-api.jar -v /home/tom/app/docker/application-dev.yml:/home/tom/app/application-dev.yml -v /home/tom/app/docker/application.yml:/home/tom/app/application.yml --name easyapi easy-api
```



nginx的主配置：

```nginx
# 全局块
user  nginx;
worker_processes  auto;  # 自动根据CPU核心数设置worker数量

# 错误日志设置 (会挂载到宿主机)
error_log  /var/log/nginx/error.log  warn;  # 只记录warning及以上级别
pid        /var/run/nginx.pid;

# 事件块
events {
    worker_connections  1024;  # 每个worker的最大连接数
    multi_accept       on;     # 同时接受多个新连接
    use                epoll;  # 使用epoll高效模型(Linux)
}

# HTTP块
http {
    # 基础设置
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    
    # 日志格式 (会挂载到宿主机)
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;

    # 性能优化
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;
    server_tokens       off;  # 隐藏Nginx版本号

    # Gzip压缩
    gzip              on;
    gzip_vary         on;
    gzip_proxied      any;
    gzip_comp_level   6;
    gzip_types        text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # 包含子配置
    include /etc/nginx/conf.d/*.conf;
}
```

conf.d目录下conf配置

```nginx
# 主服务配置
server {
    listen       8099;
    server_name  172.20.10.4;  # 替换为您的实际IP或域名
    
    # 静态资源根目录 (挂载到/usr/share/nginx/html)
    root   /usr/share/nginx/html;
    index  index.html;

    # 字符集
    charset utf-8;

    # 前端路由处理 (适用于Vue/React等SPA)
    location / {
        try_files $uri $uri/ /index.html;
        
        # 静态资源缓存控制
        location ~* \.(?:css|js|jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc)$ {
            expires 1y;
            access_log off;
            add_header Cache-Control "public";
        }
    }

    # API代理配置
    location /api/ {
        proxy_pass http://172.20.10.4:8090;  # Docker特殊域名访问宿主机
        proxy_http_version  1.1;
        
        # 代理头设置
        proxy_set_header Host               $host;
        proxy_set_header X-Real-IP          $remote_addr;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_set_header Upgrade            $http_upgrade;
        proxy_set_header Connection         "upgrade";
        
        # 超时设置
        proxy_connect_timeout       60s;
        proxy_send_timeout          60s;
        proxy_read_timeout          60s;
    }

    # Favicon处理
    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    # 错误页面
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}

```



```sh
# 保存镜像到tar
docker save -o tom.tar openjdk:17 mysql:8.0.23 redis:7.2-alpine nginx:latest easyapi:latest

# 解压镜像
docker load -i tom.tar
```



docker-compose.yml文件

```yml
version: '3.8' # 推荐使用较新的 Docker Compose 版本，例如 3.8

services:
  # MySQL 服务
  mysql:
    image: mysql:8.0.23 # 使用指定的 MySQL 镜像版本
    container_name: mysql # 指定容器名称
    ports:
      - "3307:3306" # 端口映射：宿主机3307 -> 容器3306
    environment:
      MYSQL_ROOT_PASSWORD: 123456 # 设置 MySQL root 用户的密码
    volumes:
      # 数据卷映射：将宿主机路径映射到容器内部
      - /usr/local/docker/mysql/mysql-files:/var/lib/mysql-files # MySQL文件目录
      - /usr/local/docker/mysql/conf:/etc/mysql # MySQL配置文件目录
      - /usr/local/docker/mysql/logs:/var/log/mysql # MySQL日志目录
      - /usr/local/docker/mysql/data:/var/lib/mysql # MySQL数据目录
    restart: unless-stopped # 容器退出时自动重启，除非手动停止
    # networks: # 如果需要自定义网络，可以在这里添加
    #   - my_network

  # Nginx 服务
  nginx:
    image: nginx:latest # 使用最新的 Nginx 镜像
    container_name: nginx # 指定容器名称
    ports:
      - "8099:8099" # 端口映射：宿主机8099 -> 容器8099
    volumes:
      # 配置文件映射：注意宿主机路径与容器内路径的对应
      - /home/tom/app/nginx/conf/nginx.conf:/etc/nginx/nginx.conf # Nginx主配置文件
      - /home/tom/app/nginx/conf/conf.d:/etc/nginx/conf.d # Nginx额外配置目录
      - /home/tom/app/nginx/log:/var/log/nginx # Nginx日志目录
      - /home/tom/app/nginx/html:/usr/share/nginx/html # Nginx静态文件目录
    restart: unless-stopped
    depends_on: # 依赖 MySQL 和 EasyAPI 服务，确保它们先启动
      - mysql
      - easyapi
    # networks:
    #   - my_network

  # Redis 服务
  redis:
    image: redis # 默认使用 latest 标签
    container_name: redis # 指定容器名称
    ports:
      - "6380:6379" # 端口映射：宿主机6380 -> 容器6379
    command: redis-server --requirepass 123456 # 容器启动时执行的命令，设置密码
    restart: unless-stopped
    # networks:
    #   - my_network

  # EasyAPI 服务
  easyapi:
    image: easy-api # 使用自定义的 easy-api 镜像
    container_name: easyapi # 指定容器名称
    ports:
      - "8090:8090" # 端口映射：宿主机8090 -> 容器8090
    volumes:
      # 应用文件和配置文件映射
      - /home/tom/app/docker/easy-api.jar:/home/tom/app/easy-api.jar # Java Jar 包
      - /home/tom/app/docker/application-dev.yml:/home/tom/app/application-dev.yml # 开发环境配置文件
      - /home/tom/app/docker/application.yml:/home/tom/app/application.yml # 生产环境配置文件
    restart: unless-stopped
    depends_on: # 依赖 MySQL 和 Redis 服务，确保它们先启动
      - mysql
      - redis
    # networks:
    #   - my_network

# 如果需要自定义网络，可以在这里定义
# networks:
#   my_network:
#     driver: bridge

```

```sh
docker-compose up -d
docker-compose down
```

