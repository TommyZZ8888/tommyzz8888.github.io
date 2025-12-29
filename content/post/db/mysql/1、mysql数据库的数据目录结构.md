查看mysql的主要目录结构
--------------

通过命名查看mysql的目录结构：`find / -name mysql`  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220162614805-910847939.png)

### 1.1数据库文件的存放路径

MySQL数据库文件的存放路径：/var/lib/mysql/  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220163111437-1807887880.png)

### 1.2相关命令的存放路径

相关命令目录：/usr/bin和/usr/sbin。  
**/usr/bin目录**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220163537093-646907271.png)  
**/usr/sbin目录**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220163652279-1335793189.png)  
**启动命令能读取的配置文件选项组**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220175719237-1534110880.png)

### 1.3配置文件目录

配置文件目录：/usr/share/mysql（命令及配置文件）和 /etc（如my.cnf）  
**/usr/share/mysql（命令及配置文件）**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220163915839-1761112202.png)  
**/etc目录**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220164107873-1172310117.png)

数据库和文件系统的关系
-----------

数据库的文件信息会存在磁盘上，文件系统来管理这些文件信息。

### 2.1查看默认的数据库

查看命令：SHOW DATABASES;  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220164808378-314876593.png)

*   **mysql**：MySQL系统自带的核心数据库，它存储了MySQL的**用户账户和权限信息**，一些**存储过程、事件的定义信息**，一些运行过程中产生的**日志信息**，一些帮助信息以及时区信息等。
*   **information\_schema**：MySQL系统自带的数据库，**这个数据库保存着MySQL服务器维护的所有其他数据库的信息 ，比如有哪些表、哪些视图、哪些触发器、哪些列、哪些索引**。这些信息并不是真实的用户数据，而是**一些描述性信息**，有时候也称之为 元数据 。在系统数据库information\_schema 中提供了一些以innodb\_sys 开头的表，用于表示内部系统表。
*   **performance\_schema**：MySQL系统自带的数据库，这个数据库里主要保存MySQL服务器运行过程中的一些状态信息，**可以用来监控MySQL服务的各类性能指标** 。包括统计最近执行了哪些语句，在执行过程的每个阶段都花费了多长时间，内存的使用情况等信息。
*   **sys**：MySQL系统自带的数据库，这个数据库主要是**通过视图的形式把information\_schema和performance\_schema结合起来**，帮助系统管理员和开发人员监控MySQL的技术性能。

### 2.2数据库在文件系统中的表示

使用`create database 数据库名`语句创建一个数据库的时候，在数据库目录/var/lib/mysql下会创建一个和数据库同名的子目录。  
**除了information\_schema这个系统数据库外，其他的数据库在数据目录下都有对应的子目录。**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220165921881-726155399.png)

### 2.3表在文件系统中的标识

**使用数据库a中的表b描述在INNODB和MYISAM中的区别。**

*   db.opt：存储的是数据库的字符集和比较规则
*   ibdata1：系统表空间，存储数据和索引信息
*   b.frm：描述的是表结构文件、字段长度
*   b.xxx.sdi：描述的是表结构文件、字段长度
*   b.idb：独立表空间，存储数据和索引信息
*   b.MYD(MYDATA)：存储数据信息
*   b.MYI（MYINDEX）：存储索引信息

**在INNODB中：聚簇索引（数据和索引在一起）**

*   mysql5.5.7（包括）至mysql5.6.6（不包括）版本：使用的是系统表空间  
    `存在的文件：db.opt、b.frm、ibdata1`
*   mysql5.6.6（包括）至mysql5.7（包括）版本：使用的是独立表空间  
    `存在的文件：db.opt、b.frm、b.idb`
*   mysql5.7（不包括）至mysql8.0（不包括）版本：使用的是独立表空间  
    `存在的文件：b.frm、b.idb`
*   mysql8.0以及之后的版本（把frm的信息合并到idb中）  
    `存在的文件：b.idb`  
    `在b.idb文件的目录下执行以下命令：idb2sdi --dump-flie=b.txt b.idb（查看b.txt存在表结构信息）`

**在MYISAM中：非聚簇索引（数据和索引不在一起）**

*   mysql5.5.7（包括）至mysql5.6.6（不包括）版本：使用的是系统表空间  
    `存在的文件：db.opt、b.frm、ibdata1`
*   mysql5.6.6（包括）至mysql5.7（包括）版本：使用的是独立表空间  
    `存在的文件：db.opt、b.frm、b.MYD、b.MYI`
*   mysql5.7（不包括）至mysql8.0（不包括）版本：使用的是独立表空间  
    `存在的文件：b.frm、b.MYD、b.MYI`
*   mysql8.0以及之后的版本（把frm的信息移动到sdi）  
    `存在的文件：b.xxx.sdi、b.MYD、b.MYI`

本文转自 <https://www.cnblogs.com/worldusemycode/p/15915938.html>，如有侵权，请联系删除。