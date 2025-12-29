---
title: 常用环境搭建
date: '2025-12-28T09:38:23+08:00'

categories:
  - 部署
---
# JDK

### apt方式安装

参考： https://cn.linux-console.net/?p=34553

```sh
# 查看可用版本
apt search openjdk-*

# 查看官方仓库中的 JDK 包
apt list -a 'openjdk-*-jdk*'

# 安装jdk
apt install openjdk-17-jdk

# 查看是否安装成功
java -version

# 安装的目录
root@debian:/usr/local/jdk# cd /usr/lib/jvm
root@debian:/usr/lib/jvm# ls
java-1.17.0-openjdk-amd64  java-17-openjdk-amd64  openjdk-17

# 配置环境
vi /etc/environment
添加：
JAVA_HOME="/usr/lib/jvm/java-17-openjdk-amd64"

# 验证是否更改成功
source /etc/environment
echo $JAVA_HOME

# 执行以下命令，会看到
ls -l $JAVA_HOME

root@debian:/usr/lib/jvm# ls -l $JAVA_HOME
总计 32
drwxr-xr-x  2 root root 4096  7月15日 17:21 bin
drwxr-xr-x  4 root root 4096  7月15日 17:20 conf
lrwxrwxrwx  1 root root   42  5月 1日 16:40 docs -> ../../../share/doc/openjdk-17-jre-headless
drwxr-xr-x  3 root root 4096  7月15日 17:21 include
drwxr-xr-x  2 root root 4096  7月15日 17:21 jmods
drwxr-xr-x 72 root root 4096  7月15日 17:20 legal
drwxr-xr-x  5 root root 4096  7月15日 17:21 lib
drwxr-xr-x  3 root root 4096  7月15日 17:20 man
-rw-r--r--  1 root root 1229  5月 1日 16:40 release
```



### 离线安装

参考：[Debian安装JDK-17.0.5教程_debian安装java17-CSDN博客](https://blog.csdn.net/weixin_57792864/article/details/128519792)

https://www.cnblogs.com/lwlnice/p/18298579

```sh
# 下载 https://download.oracle.com/otn/java/jdk/17.0.16+12/91eb813b5d15455fbbf4d46dcf0b9d6a/jdk-17.0.16_linux-x64_bin.tar.gz

# 也可以
wget https://download.oracle.com/otn/java/jdk/17.0.16%2B12/91eb813b5d15455fbbf4d46dcf0b9d6a/jdk-17.0.16_linux-x64_bin.tar.gz

# 创建/usr/lib/jdk，并将下载jdk复制到该路径（可以选择其他路径，我一般选择/usr/local/mysql/）

# 解压
tar -zxvf jdk-17.0.16_linux-x64_bin.tar.gz    


# 配置环境

# 方式一（更推荐）：
nano /etc/environment
# 添加
JAVA_HOME="/usr/local/jdk/jdk-17.0.16"
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:$JAVA_HOME/bin"

# 刷新 或重启系统 reboot
source /etc/environment  

# 验证
java -version



# 方式二
vi /etc/profile

#在最下方输入
export JAVA_HOME=/java/jdk-17.0.5
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH

# 刷新
source /etc/profile

# 验证
java -version

https://download.oracle.com/otn/java/jdk/17.0.16%2B12/91eb813b5d15455fbbf4d46dcf0b9d6a/jdk-17.0.16_linux-x64_bin.tar.gz
```

| 特性         | `/etc/environment`              | `/etc/profile`                     |
| ------------ | ------------------------------- | ---------------------------------- |
| **类型**     | 纯文本键值对文件                | Shell 脚本                         |
| **执行方式** | 系统启动时读取并解析            | 登录 shell 启动时执行              |
| **支持逻辑** | 不支持                          | 支持 shell 脚本逻辑 (`if`, `$VAR`) |
| **作用范围** | 系统范围，对所有进程和用户      | 仅对登录 shell 用户                |
| **常用场景** | 设置全局的、不依赖 shell 的变量 | 配置 shell 相关的变量、别名、函数  |





### 卸载

```sh
# 首先查看所有本地安装的 Java 软件包
sudo dpkg -l | grep 'jdk\|jre'

root@debian:/usr/lib/jvm# sudo dpkg -l | grep 'jdk\|jre'
ii  openjdk-17-jdk:amd64                    17.0.15+6-1~deb12u1                 amd64        OpenJDK Development Kit (JDK)
ii  openjdk-17-jdk-headless:amd64           17.0.15+6-1~deb12u1                 amd64        OpenJDK Development Kit (JDK) (headless)
ii  openjdk-17-jre:amd64                    17.0.15+6-1~deb12u1                 amd64        OpenJDK Java runtime, using Hotspot JIT
ii  openjdk-17-jre-headless:amd64           17.0.15+6-1~deb12u1                 amd64        OpenJDK Java runtime, using Hotspot JIT (headless)

# 执行删除操作
sudo apt purge openjdk-17-jdk:amd64 openjdk-17-jdk-headless:amd64 openjdk-17-jre:amd64 openjdk-17-jre-headless:amd64

# 清除系统中所有剩余的依赖项
sudo apt autoremove --purge

# 最后在 “/etc/environment”文件中删除JAVA_HOME变量
```





# MySQL

### apt方式

参考：https://blog.csdn.net/qq_45738379/article/details/135392581

```sh
# 下载对应系统的mysql-apt-config包
apt install wget -y
wget https://repo.mysql.com/apt/debian/pool/mysql-apt-config/m/mysql-apt-config/mysql-apt-config_0.8.29-1_all.deb

# 安装下载的deb包
apt install dpkg -y
apt install gnupg -y
sudo dpkg -i mysql-apt-config_0.8.29-1_all.deb

# 更新apt
sudo apt update

# 安装mysql-server
sudo apt-get -y install mysql-community-server mysql-server

#安装过程中保持空密码，保持默认强密码模式，直接回车

# 登录mysql8.0，空密码登录，第4步没有设置密码
sudo mysql -uroot

# 进入数据库
use mysql
# 修改root@localhost用户的密码，密码：大写英文 + 特殊字符 + 数字
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的密码';
# 创建 root 远程账号，用于远程登录
CREATE USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '你的密码';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
# 重载授权表
FLUSH PRIVILEGES;
# 退出
exit

# 设置开机自启动
systemctl enable mysql && systemctl daemon-reload && systemctl restart mysql && systemctl status mysql



```



若想修改my.cnf的配置

```sh
systemctl stop mysql
rm -rf /var/lib/mysql
vi /etc/mysql/my.cnf

```



```sh
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

```



### 离线安装

**环境依赖：**

```bash
apt-get update # 先更新一下包列表
apt-get install -y libaio1 libncurses6 libnuma1 sysv-rc-conf
```

（ Systemd 的服务文件方式）

**获取并解压安装包：**

```bash
# 假设你当前在 /root 目录下，且已经上传了压缩包
# 先创建好目标安装目录
mkdir -p /usr/local/mysql

# 解压到 /usr/local/mysql 目录，并剥离顶层目录
tar -xvf mysql-8.0.33-linux-glibc2.17-x86_64-minimal.tar.xz -C /usr/local/mysql --strip-components 1
```

**创建用户及用户组：**

```bash
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

**添加环境变量：**

```bash
echo 'export PATH=$PATH:/usr/local/mysql/bin' >> /etc/profile
source /etc/profile
```

**创建相应目录并赋予合适权限：**

```bash
mkdir -p /data/mysql/data /data/mysql/log

# 注意 PID 和 SOCK 文件创建的位置，与 my.cnf 保持一致
touch /data/mysql/log/mysql.err
touch /data/mysql/mysql.pid
touch /data/mysql/mysql.sock

chown mysql:mysql -R /usr/local/mysql
chown mysql:mysql -R /data/mysql

# 创建软链接，方便应用程序查找
ln -s /data/mysql/mysql.sock /tmp/mysql.sock
```

**配置文件编写 (`/etc/my.cnf`)：**

```ini
[mysqld]
user=mysql
bind-address=0.0.0.0
port=3306
basedir=/usr/local/mysql
datadir=/data/mysql/data
socket=/data/mysql/mysql.sock
log_error=/data/mysql/log/mysql.err
pid_file=/data/mysql/mysql.pid # 修正了 /datal 到 /data
max_connections=100
max_connect_errors=100
character-set-server=utf8mb4
default-storage-engine=INNODB
# 推荐使用默认的 caching_sha2_password，如果你要兼容旧客户端，再改为 mysql_native_password
default_authentication_plugin=caching_sha2_password 
lower_case_table_names = 1
interactive_timeout = 1800
wait_timeout = 1800
lock_wait_timeout = 3600
tmp_table_size = 64M
max_heap_table_size = 64M
innodb_autoinc_lock_mode = 0

[mysql]
default-character-set=utf8mb4

[client]
port=3306
default-character-set=utf8mb4
```

**初始化 MySQL：**

```bash
/usr/local/mysql/bin/mysqld --initialize --user=mysql
# 显式指定 --user=mysql 确保以正确的用户进行初始化，避免权限问题
```

**查看密码：**

```bash
grep 'root@localhost:' /data/mysql/log/mysql.err | cut -d':' -f2-
```

**创建 MySQL Systemd 服务（推荐替代 SysVinit）：**

```bash
# 创建服务文件
nano /etc/systemd/system/mysql.service
```

输入以下内容（注意这里的 `ExecStart` 等路径要与你的安装路径匹配）：

```bash
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target

[Service]
User=mysql
Group=mysql

# 设置为 forking 类型，表示 ExecStart 启动的进程会 fork 出一个子进程作为主进程
Type=forking

# 指定 PID 文件的位置，systemd 会通过这个文件来跟踪主进程
PIDFile=/data/mysql/mysql.pid 

PermissionsStartOnly=true

# ExecStart 命令不再使用 --daemonize
# 它会直接启动 mysqld，并让它读取 my.cnf 中的配置
ExecStart=/usr/local/mysql/bin/mysqld --defaults-file=/etc/my.cnf --pid-file=/data/mysql/mysql.pid

Restart=on-failure
RestartPreventExitStatus=1

# Allow time to shut down gracefully
TimeoutStopSec=60

LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

保存并退出。

**刷新 Systemd 配置并启用服务：**

```bash
systemctl daemon-reload
systemctl enable mysql # 设置开机自启
systemctl start mysql  # 启动MySQL服务
systemctl status mysql # 检查服务状态
```

如果服务启动失败，查看日志：`journalctl -u mysql -xe`。

**登录 MySQL 并修改密码：**

```bash
mysql -u root -p
# 输入之前获取的临时密码

# 如果你 my.cnf 中设置了 default_authentication_plugin=caching_sha2_password
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456'; 
# 如果你 my.cnf 中设置了 default_authentication_plugin=mysql_native_password
# ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456'; 

USE mysql;
UPDATE user set host = '%' where user = 'root';
FLUSH PRIVILEGES;
EXIT;
```





ps: **如果密码输入一直失败**

重新确认临时密码

由于密码是 `ERROR 1045` 的主要原因，让我们获取**最新**的临时密码。

1. **确保 MySQL 已停止。** 这在重新初始化之前至关重要，以避免冲突。

   ```bash
   systemctl stop mysql
   ```

   确认它已停止：

   ```bash
   systemctl status mysql
   # 应该显示 Active: inactive (dead)
   ```

2. **删除旧数据和日志（请务必小心！）。** 这能确保真正地进行全新初始化。

   ```bash
   rm -rf /data/mysql/data/*
   rm -f /data/mysql/log/mysql.err # 删除旧的错误日志文件
   ```

   **注意：** `rm -rf /data/mysql/data/*` 会删除所有 MySQL 数据。在生产环境中，请**务必备份**数据。对于你当前的学习安装，这通常是安全的。

3. **重新初始化 MySQL。**

   ```bash
   /usr/local/mysql/bin/mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/data/mysql/data
   ```

   这条命令会再次初始化数据目录并生成一个新的临时密码。

4. **再次查看新的临时密码。**

   ```bash
   grep 'root@localhost:' /data/mysql/log/mysql.err | cut -d':' -f2-
   ```

   这一次，**请仔细**复制输出中密码部分，确保没有额外的空格或不可见字符。你可以将其复制到一个文本编辑器中，确保只复制实际的密码。

5. **启动 MySQL 服务。**

   ```bash
   systemctl start mysql
   systemctl status mysql # 确认是 active (running)
   ```

6. **使用新密码登录。**

   ```bash
   mysql -u root -p
   ```

   在提示符 `Enter password:` 后，**小心地粘贴**你刚刚获取到的新临时密码，然后按回车。

------



2. 如果以上步骤仍然失败（极少见但可能）

如果上述步骤未能解决问题，这表明可能存在更深层次的认证机制或环境问题。

你可以尝试以**跳过权限认证**的方式登录 MySQL，这样你可以进入数据库，然后手动重置 `root` 密码。**注意：这会使你的数据库完全不安全，只用于紧急故障排除。**

1. **停止 MySQL 服务：**

   ```bash
   systemctl stop mysql
   ```

2. **以跳过权限的方式启动 MySQL（手动启动，不通过 systemd）：**

   ```bash
   /usr/local/mysql/bin/mysqld_safe --skip-grant-tables --skip-networking &
   ```

   - `--skip-grant-tables`：跳过权限验证，任何用户无需密码即可登录。
   - `--skip-networking`：禁用网络连接，只允许本地连接，增加安全性。
   - `&`：将进程放到后台运行。

3. **无密码登录 MySQL：**

   ```bash
   mysql -u root
   ```

   这次应该不需要密码就能直接进入 MySQL 命令行。

4. **在 MySQL 内部修改 root 密码并刷新权限：**

   ```sql
   FLUSH PRIVILEGES; -- 刷新权限，确保后续命令生效
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewSecurePasswordHere'; -- 替换为你的新密码
   -- 或者，如果你的 my.cnf 中是 default_authentication_plugin=mysql_native_password
   -- ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'YourNewSecurePasswordHere';
   FLUSH PRIVILEGES;
   EXIT;
   ```

   **注意：** 将 `YourNewSecurePasswordHere` 替换为你自己设置的新密码！

5. **停止以不安全模式运行的 MySQL 进程：** 先找到 `mysqld_safe` 的进程 ID：

   ```bash
   ps aux | grep mysqld_safe
   ```

   找到对应的 PID（通常在第二列），然后：

   ```bash
   kill <PID> # 替换为实际的 PID
   ```

   或者更暴力但有效的方式：

   ```bash
   killall mysqld_safe
   ```

6. **正常启动 MySQL 服务：**

   ```bash
   systemctl start mysql
   systemctl status mysql # 确认是 active (running)
   ```

7. **使用新密码登录：**

   ```bash
   mysql -u root -p
   ```







### 卸载

https://blog.csdn.net/weixin_43938497/article/details/110124730

```sh
# 使用命令罗列出本机上的所有MySQL相关软件
dpkg --get-selections | grep mysql

# 然后逐一卸载
sudo apt-get --purge remove mysql-server
sudo apt-get --purge remove mysql-client
sudo apt-get --purge remove mysql-common

# 在通过下列命令清除残余
apt-get autoremove
apt-get autoclean
rm /etc/mysql/ -R
rm /var/lib/mysql/ -R


49:18.118428Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: ruNn3Yc<82kx
```





# Redis

### apt方式

参考：https://blog.csdn.net/SunBoy_1409/article/details/134996932

```sh
# 更新系统
sudo apt update -y

# 安装Redis
sudo apt install redis-server

# 查看Redis状态
sudo systemctl status redis

# 验证
redis-cli

127.0.0.1:6379> keys *

# 设置密码
在 /etc/redis/redis.conf中 ，释放注释 requirepass foobared 
设置密码root，requirepass root

# 开启远程连接
在 /etc/redis/redis.conf中，注释掉 bind 127.0.0.1 -1:1

# 重启
systemctl restart redis

# 测试
root@debian:/usr/local# redis-cli
127.0.0.1:6379> set name tom
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth 123456
OK
127.0.0.1:6379> set name tom
OK
127.0.0.1:6379> get name
"tom"
```



### 离线安装

```sh
#  下载
wget http://download.redis.io/releases/redis-7.0.2.tar.gz
或 官网下载通过xftp传到服务器

#  解压到/uer/local目录下
tar -zxvf redis-7.0.2.tar.gz -C /usr/local

# 更新软件包列表
sudo apt update

# 安装编译工具和依赖项
sudo apt install -y build-essential tcl
apt install -y pkg-config libssl-dev

#  切换到/uer/local/redis-7.0.2目录下,编译
make

#  安装并指定安装目录
make install PREFIX=/usr/local/redis


#  从 redis 的源码目录中复制 redis.conf 到 redis 的安装目录
cp /usr/local/redis-5.0.3/redis.conf /usr/local/redis/bin/

# 开启远程连接 & 设置密码
# 在 /etc/redis/redis.conf中，注释掉 bind 127.0.0.1 -1:1
# 修改 redis.conf 文件，把 daemonize no 改为 daemonize yes
# 设置密码 requirepass 123456
vim redis.conf

#  后台启动
./redis-server redis.conf

#  查看是否启动
ps -ef|grep redis

#  停止
./redis-cli shutdown

# 添加服务
vim /etc/systemd/system/redis.service

# 添加配置数据
[Unit]
Description=redis-server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/bin/redis.conf
PrivateTmp=true

[Install]
WantedBy=multi-user.target
 
 
 
### 解析
Description:描述服务
After:描述服务类别
[Service]服务运行参数的设置
Type=forking是后台运行的形式
ExecStart为服务的具体运行命令
ExecReload为重启命令
ExecStop为停止命令
PrivateTmp=True表示给服务分配独立的临时空间
注意：[Service]的启动、重启、停止命令全部要求使用绝对路径
[Install]运行级别下服务安装的相关设置，可设置为多用户，即系统运行级别为3

# 开机自启
systemctl daemon-reload
systemctl start redis.service
systemctl enable redis.service


```

```sh
-- 添加完开机启动服务后,以下命令生效

systemctl start redis.service   	#启动redis服务

systemctl stop redis.service   		#停止redis服务

systemctl restart redis.service   	#重新启动服务

systemctl status redis.service   	#查看服务当前状态

systemctl enable redis.service   	#设置开机自启动

systemctl disable redis.service   	#停止开机自启动


```







# Nginx

### apt方式



```sh
# 更新包
sudo apt update

# 安装nginx
sudo apt install nginx

# 查看安装路径
whereis nginx

# nginx.conf路径
/etc/nginx

# html路径
/usr/share/nginx/html

# 启动
sudo systemctl start nginx

# 停止
sudo systemctl stop nginx

# 状态
sudo systemctl status nginx

# 开机自启动
sudo systemctl enable nginx
```



### 离线安装（也非离线）

1. **确定 Nginx 版本和架构：** 首先，你需要知道你想要安装的Nginx版本，以及目标Debian 12系统的架构（例如 `amd64`、`arm64` 等）。Debian 12 (Bookworm) 仓库中通常有 `nginx-full`, `nginx-common`, `nginx-core` 等包。

2. **下载 Nginx 及其依赖：** 在一个与目标机器相同架构、相同Debian 12版本的联网机器上，执行以下命令来下载Nginx及其所有依赖包：

   Bash

   ```
   # 更新包列表 (确保获取最新的依赖信息)
   sudo apt update
   
   # 下载 Nginx 及其所有依赖包到当前目录
   # 注意：这会将所有下载的 .deb 包存放在 /var/cache/apt/archives/ 目录下
   sudo apt install --download-only nginx
   ```

   `--download-only` 选项只会下载包，而不会进行安装。

3. **查找并复制所有 .deb 包：** 所有下载的 .deb 包都位于 `/var/cache/apt/archives/` 目录下。你需要将这些文件全部复制到一个可移动存储设备（如U盘）上，或者通过其他方式传输到离线机器上。

   ```Bash
   # 进入缓存目录
   cd /var/cache/apt/archives/
   
   # 将所有 .deb 包复制到你的可移动设备（例如 /mnt/usb_drive）
   # 请根据实际路径修改 /mnt/usb_drive
   cp *.deb /mnt/usb_drive/
   ```

   **重要提示：** Nginx 的依赖包可能包括 `libssl-dev`, `zlib1g-dev`, `libpcre3-dev`, `build-essential` 等，具体取决于你安装的 Nginx 版本和系统环境。最可靠的方法是使用 `apt install --download-only nginx`，它会自动帮你收集所有必需的 `.deb` 文件。



步骤2：在离线机器上安装 Nginx

1. **将 .deb 包传输到离线机器：** 将你复制的 `.deb` 包传输到离线Debian 12机器上的一个目录，例如 `/home/user/nginx_offline_install`。

2. **安装所有 .deb 包：** 使用 `dpkg` 命令安装所有下载的 .deb 包。由于存在依赖关系，最好一次性安装所有包，或者使用 `dpkg -i --recursive` 或 `apt install ./`。

   Bash

   ```Bash
   # 进入存放 .deb 包的目录
   cd /usr/local/nginx
   
   # 安装所有 .deb 包
   sudo dpkg -i *.deb
   
   # 如果有依赖问题，使用以下命令尝试修复（通常在线安装时才用，但离线可以尝试，可能需要手动解决依赖）
   # sudo apt --fix-broken install
   ```

   `dpkg -i *.deb` 会尝试安装当前目录下所有的 .deb 包。如果遇到依赖问题，`dpkg` 可能会报错。通常，当你下载了所有正确的依赖包时，这个命令会成功。

3. **验证 Nginx 安装：** 安装完成后，Nginx 服务应该会自动启动。你可以检查其状态：

   Bash

   ```sh
   sudo systemctl status nginx
   ```



```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       8080;
        server_name  192.168.111.155;

        root /home/tom/app/dist;
        index index.html;

        location = /favicon.ico {
            try_files /favicon.ico =404;
        }

        location / {
            try_files $uri $uri/ /index.html;
        }

        location /api/ {
            proxy_pass http://localhost:8090;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```



# Tomcat







# 宝塔







# 项目部署：

nginx配置

```sh
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       8080;
        server_name  192.168.111.155;

        root /home/tom/app/dist;
        index index.html;

        location = /favicon.ico {
            try_files /favicon.ico =404;
        }

        location / {
            try_files $uri $uri/ /index.html;
        }

        location /api/ {
            proxy_pass http://localhost:8090;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```



权限报错：

```sh
server: 192.168.111.155, request: "GET /favicon.ico HTTP/1.1", host: "192.168.111.155:8080", referrer: "http://192.168.111.155:8080/" 2025/07/17 11:11:37 [crit] 25260#25260: *6 stat() "/home/tom/app/dist/index.html" failed (13: Permission denied), client: 192.168.111.152,
```

项目存放路径例：

/home/tom/app；

nginx配置中用户user www-data;

```sh
# 检查项目文件目录权限 确保这些目录和文件的权限允许 www-data 用户访问
ls -ld /home/tom /home/tom/app /home/tom/app/dist /home/tom/app/dist/index.html

# 修改权限 为 www-data 用户授予读取权限
sudo chown -R tom:www-data /home/tom/app/dist
sudo chmod -R g+r /home/tom/app/dist

# 确保目录可访问
sudo chmod o+x /home/tom
sudo chmod o+x /home/tom/app
sudo chmod o+x /home/tom/app/dist
```

