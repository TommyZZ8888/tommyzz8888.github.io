---
title: debian使用
date: '2025-12-28T09:38:23+08:00'

categories:
  - 部署
---

### 目录

**根目录** 和 **主要顶层目录**

```nginx
根目录 (/)
所有文件和目录的起点，整个文件系统的“根”。所有其他目录都挂载在这个目录下。

主要顶层目录
/bin：基本用户命令
包含所有用户都能使用的基本命令，例如 ls (列出文件), cat (查看文件内容), cp (复制文件), mv (移动文件), rm (删除文件) 等。这些命令在单用户模式下也必须可用。

/sbin：系统管理命令
包含系统**管理员（root 用户）**才能使用的基本系统管理命令，例如 fdisk (磁盘分区), mount (挂载文件系统), reboot (重启), shutdown (关机), ip (网络配置) 等。

/etc：配置文件
存放所有系统范围的配置文件。例如，用户账户信息 (passwd), 网络配置 (network/interfaces), APT 源列表 (apt/sources.list) 等。通常是文本文件，不包含二进制可执行文件。

/dev：设备文件
包含硬件设备文件。这些文件并非实际的数据文件，而是用于与硬件设备进行交互的接口。例如，/dev/sda (第一个 SCSI/SATA 硬盘), /dev/tty0 (第一个虚拟终端), /dev/null (空设备) 等。

/proc：进程信息和内核文件系统
一个虚拟文件系统，不存储在硬盘上，而是实时映射内核和进程的信息。可以通过查看这个目录下的文件来获取系统运行状态、硬件信息等。例如，/proc/cpuinfo (CPU 信息), /proc/meminfo (内存信息), 对应进程的目录 (/proc/PID) 等。

/sys：系统信息
另一个虚拟文件系统，与 /proc 类似，也提供对内核信息的访问，但通常更侧重于设备的硬件配置和状态，采用更结构化的方式。

/tmp：临时文件
存放临时文件。任何用户都可以向此目录写入。通常，这个目录下的内容在系统重启后会被清空。
```



**用户相关目录**

```nginx
用户相关目录
/home：用户主目录
普通用户的个人主目录的父目录。每个用户通常都有一个以其用户名命名的子目录，例如 /home/yourusername。用户的个人文件、文档、配置（如 .bashrc, .config）都存储在这里。

/root：Root 用户主目录
**系统管理员（root 用户）**的专属主目录，与普通用户的 /home 分开。
```

**程序和数据目录**

```nginx
程序和数据目录
/usr：Unix System Resources（用户系统资源）
包含大部分用户程序、库、文档和只读的、可共享的数据。它是系统上所有“用户级别”软件的主要存储库。

/usr/bin：非基本的用户命令（比 /bin 中的命令更复杂或不那么关键）。

/usr/sbin：非基本的系统管理命令。

/usr/lib：程序库文件。

/usr/local：用于本地安装的软件，即那些不是通过发行版包管理器安装的软件。通常编译安装的程序会放到这里。

/usr/share：与架构无关的共享数据，如文档、man 手册页、图标、字体等。

/usr/include：C/C++ 语言的头文件。

/usr/src：存放源代码，例如内核源代码。

/var：Variable Data（可变数据）
包含在系统运行过程中会经常发生变化的数据，如日志文件、缓存文件、邮件队列、打印队列、数据库文件等。

/var/log：系统和应用程序的日志文件。

/var/cache：应用程序的缓存数据（可安全删除）。

/var/lib：应用程序的状态信息和持久化数据，例如数据库文件、包管理器的状态文件等。

/var/mail：用户邮箱。

/var/spool：待处理的数据，如打印队列、邮件队列、cron 任务队列。

/var/tmp：与 /tmp 类似，但通常在系统重启后不会被清空。

/var/run (或 /run)：存放运行时数据，如进程的 PID 文件、套接字文件。在较新的 Linux 版本中，/var/run 通常是 /run 的符号链接。

/var/www: 如果系统是Web服务器，存放网站的默认根目录。


```

**挂载点和可选目录**

```nginx
挂载点和可选目录
/mnt：临时文件系统挂载点
用于临时挂载文件系统的目录，例如手动挂载的 USB 驱动器、光盘或网络共享。

/media：可移动媒体挂载点
用于自动挂载可移动媒体设备（如 USB 闪存盘、SD 卡、CD/DVD）的目录。当插入这些设备时，它们通常会自动挂载到此目录下的子目录中。

/opt：第三方应用程序
用于安装可选的、独立的第三方软件包。这些软件包通常是自包含的，不依赖于系统中的其他库，例如一些商业软件或大型应用程序。

/srv：服务数据
存放由系统提供的服务所使用的数据。例如，Web 服务器的数据可能会放在 /srv/www，FTP 服务器的数据放在 /srv/ftp。

软链接/兼容性目录
/lib：基本共享库（通常是符号链接）
在现代 Debian 系统中，/lib 通常是指向 /usr/lib 的符号链接，以保持向后兼容性，确保旧程序能找到它们的库。它包含 /bin 和 /sbin 中的程序所需的基本库。

/lib64：64位库（在某些系统上存在）
在某些 64 位系统上，可能存在 /lib64 用于存放 64 位库，但 Debian 通常将所有库都放在 /lib (链接到 /usr/lib)。


```







### 快捷键

```sh 
# nano：

**保存并退出**（`Ctrl + X` → `Y` → `Enter`）



```





```sh
# vi 快捷键
# 查询
按下 /
输入 search_word
按 Enter
用 n/N 在结果间导航

^回到一行开始位置 $回到一行结束位置

yy  复制内容 类似ctrl+c
p	粘贴内容 类似ctrl+v
dd	删除内容 类似ctrl+x
u 撤销

ctrl+f 向前翻页
ctrl+b 向后翻页
ctrl+u 向上翻半页
ctrl+d 向下翻半页



```





### 切换源

当前使用的源：/etc/apt/sources.list

一键换源脚本：bash <(curl -sSL https://linuxmirrors.cn/main.sh)

地址：[使用方法 - LinuxMirrors](https://linuxmirrors.cn/use/)



**阿里云**

```sh
deb http://mirrors.aliyun.com/debian/ bookworm main contrib non-free
deb-src http://mirrors.aliyun.com/debian/ bookworm main contrib non-free
 
deb http://mirrors.aliyun.com/debian-security/ bookworm/updates main contrib non-free
deb-src http://mirrors.aliyun.com/debian-security/ bookworm/updates main contrib non-free
 
deb http://mirrors.aliyun.com/debian/ bookworm-updates main contrib non-free
deb-src http://mirrors.aliyun.com/debian/ bookworm-updates main contrib non-free
 
deb http://mirrors.aliyun.com/debian/ bookworm-backports main contrib non-free
deb-src http://mirrors.aliyun.com/debian/ bookworm-backports main contrib non-free
```



清华大学

```sh

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free non-free-firmware
deb https://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware


```

or

```sh
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free
 
deb https://mirrors.tuna.tsinghua.edu.cn/debian-security/ bookworm/updates main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security/ bookworm/updates main contrib non-free
 
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free
 
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-backports main contrib non-free

```



### 允许远程访问

```sh
# 查看ssh是否开启
systemctl status sshd

# 修改配置
nano /etc/ssh/sshd_config

PermitRootLogin yes
PasswordAuthentication yes

systemctl restart sshd
```

Www12580..
