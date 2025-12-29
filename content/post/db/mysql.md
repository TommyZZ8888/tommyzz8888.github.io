---
title: MySQL
date: '2025-12-28T09:38:23+08:00'

categories:
- DB
---

# MySql

mysql执行流程

![image-20230613135151431](/img/db/执行流程.png)

###  1、InnoDB

**InnoDB**是一个将表中数据存储到磁盘上的存储引擎，所以即使关机后重启我们的数据还是存在的，而真正处理数据的过程是发生在内存中的，所以需要把磁盘中的数据加载到内存中，如果是处理写入或修改请求的话，还需要把内存中的内容刷新到磁盘上。



**字符集和比较规则**

- ASCII字符集：共收录128个字符，包含空格，标点符号，数字，大小写字母

- ISO 8895-1字符集：共收录256个字符

- GB2312字符集：收录了汉字以及拉丁字母，希腊字母等，其中收录汉字6763个，其他文字符号682个，其中这种字符集兼容ASCII字符集。

- GBK字符集：只是在GB2312字符集上进行扩充，编码规则向上兼容GB2312字符集

- utf8字符集：收录地球上能想到的所有字符。

  

  - `utf8mb3`：阉割过的`utf8`字符集，只使用1～3个字节表示字符。
  - `utf8mb4`：正宗的`utf8`字符集，使用1～4个字节表示字符。

  | 字符集名称 | Maxlen |
  | ---------- | ------ |
  | `ascii`    | `1`    |
  | `latin1`   | `1`    |
  | `gb2312`   | `2`    |
  | `gbk`      | `2`    |
  | `utf8`     | `3`    |
  | `utf8mb4`  | `4`    |

- 名称后缀意味着该比较规则是否区分语言中的重音、大小写什么的，具体可以用的值如下：

| 后缀   | 英文释义             | 描述             |
| ------ | -------------------- | ---------------- |
| `_ai`  | `accent insensitive` | 不区分重音       |
| `_as`  | `accent sensitive`   | 区分重音         |
| `_ci`  | `case insensitive`   | 不区分大小写     |
| `_cs`  | `case sensitive`     | 区分大小写       |
| `_bin` | `binary`             | 以二进制方式比较 |







**innoDB数据页**

innoDB都是以页为单位存放数据，每页大小为16kb。

innoDB始终会维护一条记录的单链表，链表中的各个节点是按照主键值由小到大的顺序连接起来的。

一个数据页大概分为7部分：

- `File Header`，表示页的一些通用信息，占固定的38字节。
- `Page Header`，表示数据页专有的一些信息，占固定的56个字节。
- `Infimum + Supremum`，两个虚拟的伪记录，分别表示页中的最小和最大记录，占固定的`26`个字节。
- `User Records`：真实存储我们插入的记录的部分，大小不固定。
- `Free Space`：页中尚未使用的部分，大小不确定。
- `Page Directory`：页中的某些记录相对位置，也就是各个槽在页面中的地址偏移量，大小不固定，插入的记录越多，这个部分占用的空间越多。
- `File Trailer`：用于检验页是否完整的部分，占用固定的8个字节。



innoDB的分组：对于最小记录所在的分组只能有1条记录，最大记录所在的分组拥有的记录条数只能在1-8条之间，剩下的分组中记录的条数范围只能在4-8条之间。`InnoDB`会为把页中的记录划分为若干个组，每个组的最后一个记录的地址偏移量作为一个`槽`，存放在`Page Directory`中。

所以在一个数据页中查找指定主键的记录分为两步：

​    1、通过二分法确定该记录所在的槽，并找到该槽中主键值最小的那条记录。

​	2、通过记录的next_record属性遍历该槽所在的组中的各个记录。



**B+树**

在对页中的记录进行增删改查操作过程中，，我们必须通过一些诸如记录移动的操作来始终保证 **下一个数据页中用户记录的主键必须大于上一个数据页中用户记录的主键值**  这个过程叫做页分裂

B+树本身就是一个目录，或者说本身就是一个索引。

**索引**

==innoDB引擎会自动为我们创建聚簇索引，在innoDB存储引擎中，聚簇索引就是数据的存储方式，索引即数据，数据即索引==。

- 聚簇索引：

   1、使用主键值的大小进行记录和页的排序

   2、B+树的叶子节点存储的是完整的用户数据

- 非聚簇索引

   回表：根据以c2列大小排序的B+树来确定要查找的主键值，然后根据查找到的主键值在聚簇索引中查找一下完整的用户记录。

  

  1. `B+`树索引在空间和时间上都有代价，所以没事儿别瞎建索引。
  2. `B+`树索引适用于下面这些情况：
     - 全值匹配
     - 匹配左边的列
     - 匹配范围值
     - 精确匹配某一列并范围匹配另外一列
     - 用于排序
     - 用于分组
  3. 在使用索引时需要注意下面这些事项：
     - 只为用于搜索、排序或分组的列创建索引
     - 为列的基数大的列创建索引
     - 索引列的类型尽量小
     - 可以只对字符串值的前缀建立索引
     - 只有索引列在比较表达式中单独出现才可以适用索引
     - 为了尽可能少的让`聚簇索引`发生页面分裂和记录移位的情况，建议让主键拥有`AUTO_INCREMENT`属性。
     - 定位并删除表中的重复和冗余索引
     - 尽量使用`覆盖索引`进行查询，避免`回表`

  

**MySql数据结构的类型**



单表访问效率

| 访问效率    | 访问方式                                                     |
| ----------- | ------------------------------------------------------------ |
| const       | 主键索引，唯一二级索引                                       |
| ref         | 普通二级索引                                                 |
| ref_or_null | 二级索引列的值和该列的null值                                 |
| range       | 搜索条件复杂（WHERE key2 IN (1438, 6328) OR (key2 >= 38 AND key2 <= 79） |
| index       | 遍历二级索引记录的执行方式（SELECT key_part1, key_part2, key_part3 FROM single_table WHERE key_part2 = 'abc';）  1-2-3是联合索引，2是唯一索引 |
| all         | 全表扫描                                                     |

**一般情况下执行一个查询只会用到单个二级索引**





**InnoDB表空间**

 InnoDB是以页为单位管理存储空间的，为了更好的管理页，提出了区的概念，，对于16kb的页来说，连续的64个页就是一个区，每26个区划分成一组。

当数据足够多时，B+树的叶子节点和非叶子节点被区别对待，，存放叶子节点的区的集合算是一个段，存放非叶子节点的区的集合也算是一个段。也就是说一个索引会生成两个段，一个叶子节点段，一个非叶子节点段。









### 面试点

##### 1、update会锁表吗

 A:实际例子：

1.没有索引

运行命令：begin;开启事务，然后运行命令：update tb_user set phone=11 where name="c1";修改，先别commit事务。

再开一个窗口，直接运行命令：update tb_user set phone=22 where name="c2";会发现命令卡住了，但是当前面一个事务通过commit提交了，命令就会正常运行结束，说明是被锁表了。

2.给name字段加索引

create index index_name on tb_user(name);

然后继续如1里面的操作，也就是一个开启事务，运行update tb_user set phone=11 where name="c1"；先不提交

然后另一个运行update tb_user set phone=22 where name="c2";发现命令不会卡住，说明没有锁表

但是如果另一个也是update tb_user set phone=22 where name="c1";更新同一行，说明是锁行了

3.总结

如果没有索引，所以update会锁表，如果加了索引，就会锁行





### 2、Mysql用户管理和权限管理

#### 用户管理

##### 2.1登录服务器

启动Mysql服务器后可以通过mysql命令来登录mysql服务器

```sql
mysql -h hostName/hostIP -P port -u userName -p DataBaseName -e "sql语句"
```

- -h参数 后面接主机名或者主机IP，hostname为主机，hostIP为主机IP。
- -P参数 后面接MySQL服务的端口，通过该参数连接到指定的端口。MySQL服务的默认端口是3306，不使用该参数时自动连接到3306端口，port为连接的端口号。
- -u参数 后面接用户名，username为用户名。
- -p参数 会提示输入密码。
- DatabaseName参数 指明登录到哪一个数据库中。如果没有该参数，就会直接登录到MySQL数据库中，然后可以使用USE命令来选择数据库。
- -e参数 后面可以直接加SQL语句。登录MySQL服务器以后即可执行这个SQL语句，然后退出MySQL服务器。

##### 2.2创建用户

```sql
CREATE USER 用户名 [[IDENTIFIED BY '密码'][,用户名 [IDENTIFIED BY '密码']];
```

- 用户名参数表示新建用户的账户，由 用户（User） 和 主机名（Host：默认为%，代表所有地址） 构成；
- “[ ]”表示可选，也就是说，可以指定用户登录时需要密码验证，也可以不指定密码验证，这样用户可以直接登录。不过，不指定密码的方式不安全，不推荐使用。如果指定密码值，这里需要使用IDENTIFIED BY指定明文密码值。
- CREATE USER语句可以同时创建多个用户。

##### 2.3修改用户

修改用户名：

第一步：

```sql
 update mysql.user set User = 'li4' where User = 'wang5'
```

第二步：

```sql
FLUSH PRIVILEGES
```



##### 2.4删除用户

两种方式：
**方式1：使用DROP方式删除（推荐）**
使用DROP USER语句来删除用户时，必须用于DROP USER权限。DROP USER语句的基本语法形式如下：

```sql
DROP USER user[,user]
```

**方式2：使用DELETE方式删除**
第一步：通过delete命令删除用户：

```sql
delete from mysql.user where host='hostname' and user = 'username'
```

注意：不推荐通过 DELETE FROM mysql.user WHERE User='li4' 进行删除，系统会有残留信息保留。而drop user命令会删除用户以及对应的权限，执行命令后你会发现mysql.user表和mysql.db表的相应记录都消失了。

##### 2.5用户的密码

###### 2.5.1密码过期策略

过期策略可以是**全局的** ，也可以为每个账号设置**单独的**过期策略，限制用户密码的有效时间。
**全局的设置：**
方式一：使用SQL语句更改该变量的值并持久化

```sql
SET PERSIST default_password_lifetime = 180; # 建立全局策略，设置密码每隔180天过期
```

方式二：配置文件my.cnf中进行维护

```sql
[mysqld]default_password_lifetime=180 #建立全局策略，设置密码每隔180天过期
```

**单独的设置**
每个账号**既可延用全局密码过期策略，也可单独设置策略**。在 CREATE USER 和 ALTER USER 语句上**加入 PASSWORD EXPIRE 选项可实现单独设置策略**。下面是一些语句示例。

```sql
1、不能使用最近5个密码： 
CREATE USER 'kangshifu'@'localhost' PASSWORD HISTORY 5; 
ALTER USER 'kangshifu'@'localhost' PASSWORD HISTORY 5; 
2、不能使用最近365天内的密码： 
CREATE USER 'kangshifu'@'localhost' PASSWORD REUSE INTERVAL 365 DAY; 
ALTER USER 'kangshifu'@'localhost' PASSWORD REUSE INTERVAL 365 DAY; 
3、既不能使用最近5个密码，也不能使用365天内的密码： 
CREATE USER 'kangshifu'@'localhost' PASSWORD HISTORY 5 
PASSWORD REUSE INTERVAL 365 DAY; 
ALTER USER 'kangshifu'@'localhost' PASSWORD HISTORY 5 
PASSWORD REUSE INTERVAL 365 DAY; 
4、延用全局密码重用策略： 
CREATE USER 'kangshifu'@'localhost' PASSWORD HISTORY DEFAULT 
PASSWORD REUSE INTERVAL DEFAULT; 
ALTER USER 'kangshifu'@'localhost' PASSWORD HISTORY DEFAULT 
PASSWORD REUSE INTERVAL DEFAULT; 
```



###### 2.5.2密码重用策略

mysql限制已经使用过的密码，重用策略基于密码更改的数量和使用时间，重用策略可以是**全局的** ，也可以为每个账号设置**单独的**重用策略。
**全局的设置：**
方式一：使用SQL语句更改该变量的值并持久化

```
`SET PERSIST password_history = 6; #设置不能选择最近使用过的6个密码`
`SET PERSIST password_reuse_interval = 365; #设置不能选择最近一年内的密码`
```

方式二：配置文件my.cnf中进行维护

```
[mysqld]
password_history=6
password_reuse_interval=365
```

**单独的设置**
每个账号**既可延用全局密码重用策略，也可单独设置策略**。在 CREATE USER 和 ALTER USER 语句上**加入 PASSWORD HISTORY 或者 PASSWORD_REUSE_INTERVAL 选项可实现单独设置策略**。下面是一些语句示例。

```sql
1、设置kangshifu账号密码每90天过期： 
CREATE USER 'kangshifu'@'localhost' PASSWORD EXPIRE INTERVAL 90 DAY; 
ALTER USER 'kangshifu'@'localhost' PASSWORD EXPIRE INTERVAL 90 DAY; 
2、设置密码永不过期： 
CREATE USER 'kangshifu'@'localhost' PASSWORD EXPIRE NEVER; 
ALTER USER 'kangshifu'@'localhost' PASSWORD EXPIRE NEVER; 
3、延用全局密码过期策略： 
CREATE USER 'kangshifu'@'localhost' PASSWORD EXPIRE DEFAULT; 
ALTER USER 'kangshifu'@'localhost' PASSWORD EXPIRE DEFAULT;
4、设置用户密码直接过期： 
ALTER USER 'kangshifu'@'localhost' PASSWORD EXPIRE;
```



###### 2.5.4密码复杂度策略

- 在MySQL8.0之前，MySQL使用的是validate_password插件检测、验证账号密码强度，保障账号的安全性。
- MySQL8.0更加安全，引入了服务器组件（Components）这个特性，validate_password插件已用服务器组件重新实现。8.0.25版本的数据库中，默认自动安装validate_password组件。

#### 权限管理

##### 2.6权限列表

使用该命令可以展示出mysql中存在的权限

```sql
show priviledges
```

（1）**CREATE和DROP权限**：可以创建新的数据库和表，或删除（移掉）已有的数据库和表。如果将 MySQL数据库中的DROP权限授予某用户，用户就可以删除MySQL访问权限保存的数据库。
（2）**SELECT、INSERT、UPDATE和DELETE权限**：允许在一个数据库现有的表上实施操作。
（3）**SELECT权限**：只有在它们真正从一个表中检索行时才被用到。
（4）**INDEX权限**：允许创建或删除索引，INDEX适用于已有的表。如果具有某个表的CREATE权限，就可以在CREATE TABLE语句中包括索引定义。
（5）**ALTER权限**：可以使用ALTER TABLE来更改表的结构和重新命名表。
（6）**CREATE ROUTINE权限**：用来创建保存的程序（函数和程序），ALTER ROUTINE权限：用来更改和删除保存的程序， EXECUTE权限：用来执行保存的程序。
（7）**GRANT权限**：允许授权给其他用户，可用于数据库、表和保存的程序。
（8）**FILE权限**：使用户可以使用LOAD DATA INFILE和SELECT ... INTO OUTFILE语句读或写服务器上的文件，任何被授予FILE权限的用户都能读或写MySQL服务器上的任何文件（说明用户可以读任何数据库目录下的文件，因为服务器可以访问这些文件）。
[![img](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220123904589-418181875.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220123904589-418181875.png)

##### 2.7授予权限的原则

**权限控制主要是出于安全因素，因此需要遵循以下几个经验原则 ：**
1、只授予能**满足需要的最小权限**，防止用户干坏事。比如用户只是需要查询，那就只给select权限就可以了，不要给用户赋予update、insert或者delete权限。
2、创建用户的时候**限制用户的登录主机**，一般是限制成指定IP或者内网IP段。
3、为每个用户**设置满足密码复杂度的密码**。
4、 定期清理不需要的用户，回收权限或者删除用户。

##### 2.8授予权限

给用户授权的方式有 2 种，分别是通过**把角色赋予用户给用户授权（mysql8.0的特性）** 和 **直接给用户授权**。用户是数据库的使用者，我们可以通过给用户授予访问数据库中资源的权限，来控制用户对数据库的访问，消除安全隐患。
**授权命令：该权限如果发现没有该用户，则会直接新建一个用户。**
`GRANT 权限1,权限2,…权限n ON 数据库名称.表名称 TO 用户名@用户地址 [IDENTIFIED BY ‘密码口令’];`
举例：
1、给li4用户用本地命令行方式，授予atguigudb这个库下的所有表的插删改查的权限。
`GRANT SELECT,INSERT,DELETE,UPDATE ON atguigudb.* TO li4@localhost ;`
2、授予通过网络方式登录的joe用户 ，对所有库所有表的全部权限，密码设为123。如果需要赋予包括grant权限，添加参数“WITH GRANT OPTION”这个选项即可，表示该用户可以将自己拥有的权限授权给其他用户，可以使用grant重复给用户添加权限。
`GRANT ALL PRIVILEGES ON *.* TO joe@'%' IDENTIFIED BY '123' with grant option;`

> 我们在开发应用的时候，经常会遇到一种需求，就是要根据用户的不同，对数据进行横向和纵向的分组。
> 1、所谓横向的分组，就是指用户可以接触到的数据的范围，比如可以看到哪些表的数据；
> 2、所谓纵向的分组，就是指用户对接触到的数据能访问到什么程度，比如能看、能改，甚至是删除

##### 2.9查看权限

- 查看当前用户的权限
  `SHOW GRANTS; 或者 SHOW GRANTS FOR CURRENT_USER; 或者 SHOW GRANTS FOR CURRENT_USER();`
- 查看某些用户的权限
  `SHOW GRANTS FOR 'user'@'主机地址' ;`

##### 2.10收回权限

**收回权限就是取消已经赋予用户的某些权限**。收回用户不必要的权限可以在一定程度上保证系统的安全性。

> MySQL中使用REVOKE语句取消用户的某些权限。使用REVOKE收回权限之后，用户账户的记录将从db、host、tables_priv和columns_priv表中删除，但是用户账户记录仍然在user表中保存（删除user表中的账户记录使用DROP USER语句，在将用户账户从user表删除之前，应该收回相应用户的所有权限。）

- 收回权限的命令（收回权限后需要用户重新登录才能生效）
  `REVOKE 权限1,权限2,…权限n ON 数据库名称.表名称 FROM 用户名@用户地址;`
  举例：
  `收回全库全表的所有权限：REVOKE ALL PRIVILEGES ON *.* FROM joe@'%';`
  `收回mysql库下的所有表的插删改查权限：REVOKE SELECT,INSERT,UPDATE,DELETE ON mysql.* FROM joe@localhost;`

#### 权限表和访问控制（更深层次的理解）

#### 权限表

当msyql通过grant命令给用户赋予某些权限时，mysql会把用户的信息保存在相应的权限表中，**mysql服务器通过权限表来控制用户对数据库的访问**，权限表存放在**mysql数据库**中。
**权限表主要有：user表、db表、table_priv表、column_priv表、proc_priv表。当mysql启动时，服务器会将这些数据库表中的权限信息读入到内存中**

| msyql数据库的表名 | 描述                   |
| :---------------- | :--------------------- |
| user              | 用户账号和权限信息     |
| db                | 数据库层级的权限       |
| table_priv        | 表层级的权限           |
| columns_priv      | 列层级的权限           |
| procs_priv        | 存储的过程和函数的权限 |

##### 2.11user表

**user表：是MySQL中最重要的一个权限表， 记录用户账号和权限信息。**
[![img](/img/db/mysql_01.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220154814619-880344668.png)
这些字段可以分成4类，分别是范围列（或用户列）、权限列、安全列和资源控制列。
**1、范围列（或用户列）**

- host：表示连接类型
  `1、% 表示所有远程通过 TCP方式的连接`
  `2、IP 地址 如 (192.168.1.2、127.0.0.1) 通过制定ip地址进行的TCP方式的连接`
  `3、机器名 通过制定网络中的机器名进行的TCP方式的连接`
  `4、::1 IPv6的本地ip地址，等同于IPv4的 127.0.0.1`
  `5、localhost 本地方式通过命令行方式的连接 ，比如mysql -u xxx -p xxx 方式的连接。`
- user：表示用户名，同一用户通过不同方式链接的权限是不一样的。
- password：密码
  `1、所有密码串通过 password(明文字符串) 生成的密文字符串。mysql 5.7 及之后版本的密码保存到 authentication_string 字段中不再使用password字段。`
  `2、MySQL 8.0 在用户管理方面增加了角色管理，默认的密码加密方式也做了调整，由之前的 SHA1 改为了 SHA2 ，不可逆 。同时加上MySQL 5.7的禁用用户和用户过期的功能，MySQL在用户管理方面的功能和安全性都较之前版本大大的增强了。`

**2、权限列**

- Grant_priv字段：表示是否拥有GRANT权限
- Shutdown_priv字段：表示是否拥有停止MySQL服务的权限
- Super_priv字段：表示是否拥有超级权限
- Execute_priv字段：表示是否拥有EXECUTE权限。拥有EXECUTE权限，可以执行存储过程和函数。
- Select_priv , Insert_priv：为该用户所拥有的权限。

**3、安全列**

- ssl_type、ssl_cipher：用于加密 ；
- x509_issuer、x509_subject：用于标识用户 ；
- Plugin字段：用于验证用户身份的插件，该字段不能为空。如果该字段为空，服务器就使用内建授权验证机制验证用户身份。

**4、资源控制列**
**资源控制列的字段用来限制用户使用的资源。**

- max_questions，用户每小时允许执行的查询操作次数；
- max_updates，用户每小时允许执行的更新操作次数；
- max_connections，用户每小时允许执行的连接操作次数；
- max_user_connections，用户允许同时建立的连接次数。

##### 2.12db表

**db表：记录数据库层级的权限**
[![img](/img/db/记录数据库层级的权限.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220152406519-649151221.png)
**1、用户列**
db表用户列有3个字段，分别是Host、User、Db。这3个字段分别表示主机名、用户名和数据库
名。表示从某个主机连接某个用户对某个数据库的操作权限，这3个字段的组合构成了db表的主键。
**2、权限列**
Create_routine_priv和Alter_routine_priv这两个字段决定用户是否具有创建和修改存储过程的权限。

##### 2.13tables_priv表

**tables_priv表：记录表层级的权限**
[![img](/img/db/记录表层级的权限.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220152825187-262801799.png)

- Host、Db、User 和 Table_name 四个字段分别表示主机名、数据库名、用户名和表名。表示从某个主机连接某个用户对某个数据库中的某个表的操作权限，这4个字段的组合构成了tables_priv表的主键。
- Grantor表示修改该记录的用户。
- Timestamp表示修改该记录的时间。
- Table_priv表示对表的操作权限。包括Select、Insert、Update、Delete、Create、Drop、Grant、References、Index和Alter。
- Column_priv表示对表中的列的操作权限，包括Select、Insert、Update和References。

##### 2.14columns_priv表

**columns_priv表：记录列层级的权限**
[![img](/img/db/记录列层级的权限.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220153703503-765466053.png)

- Host、Db、User、Table_name和column_name 5个字段分别表示主机名、数据库名、用户名、表名和列名。表示从某个主机连接某个用户对某个数据库中的某个表中的某个列的操作权限，这5个字段的组合构成了columns_priv表的主键。
- Timestamp表示修改该记录的时间。
- Column_priv表示对表中的列的操作权限，包括Select、Insert、Update和References。

##### 2.15procs_priv表

**procs_priv表：存储的过程和函数的权限**
[![img](/img/db/存储的过程和函数的权限.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220154121398-1977529702.png)

- Host、Db、User、Routine_name和Routine_type 5个字段分别表示主机名、数据库名、用户名、存储过程名和存储类型。表示从某个主机连接某个用户对某个数据库中的某个存储过程的操作权限，这5个字段的组合构成了procs_priv表的主键。
- Grantor表示修改该记录的用户。
- proc_priv表示对存储过程的操作权限，包括execute、alter routine、grant。
- Timestamp表示修改该记录的时间。

#### 访问控制

##### 2.16连接核实阶段

客户端用户会在连接请求中提供用户名、主机地址、用户密码，MySQL服务器接收到用户请求后，**会使用user表中的host、user和authentication_string这3个字段匹配客户端提供信息**。如果连接核实没有通过，服务器就完全拒绝访问；否则，服务器接受连接，然后进入请求核实阶段。

##### 2.17请求核实阶段

一旦建立了连接，服务器就进入请求核实阶段。对此连接上进来的每个请求，服务器检查该请求要执行什么操作、是否有足够的权限来执行它，这正是需要授权表中的权限列发挥作用的地方。这些权限可以来自user、db、table_priv和column_priv表。
[![img](/img/db/请求核实阶段流程图.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220161703950-393897996.png)

### 3、Mysql的数据目录结构

##### 3.1查看mysql的主要目录结构

通过命名查看mysql的目录结构：`find / -name mysql`
[![img](/img/db/mysql目录结构\mysql主要目录结构.png)

##### 3.2数据库文件的存放路径

MySQL数据库文件的存放路径：/var/lib/mysql/
[![img](/img/db/mysql目录结构\数据库文件的存放路径.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220163111437-1807887880.png)

##### 3.3相关命令的存放路径

相关命令目录：/usr/bin和/usr/sbin。
**/usr/bin目录**
[![img](/img/db/mysql目录结构\相关命令的存放路径.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220163537093-646907271.png)
**/usr/sbin目录**
[![img](/img/db/mysql目录结构\目录.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220163652279-1335793189.png)
**启动命令能读取的配置文件选项组**
[![img](/img/db/mysql目录结构\2345397-20220220175719237-1534110880.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220175719237-1534110880.png)

##### 3.4配置文件目录

配置文件目录：/usr/share/mysql（命令及配置文件）和 /etc（如my.cnf）
**/usr/share/mysql（命令及配置文件）**
[![img](/img/db/mysql目录结构\配置文件目录.png)
**/etc目录**
[![img](/img/db/mysql目录结构\2345397-20220220164107873-1172310117.png)](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220164107873-1172310117.png)

#### 数据库和文件系统的关系

数据库的文件信息会存在磁盘上，文件系统来管理这些文件信息。

##### 3.5查看默认的数据库

查看命令：SHOW DATABASES;
[![img](/img/db/mysql目录结构\查看默认的数据库.png)

- **mysql**：MySQL系统自带的核心数据库，它存储了MySQL的**用户账户和权限信息**，一些**存储过程、事件的定义信息**，一些运行过程中产生的**日志信息**，一些帮助信息以及时区信息等。
- **information_schema**：MySQL系统自带的数据库，**这个数据库保存着MySQL服务器维护的所有其他数据库的信息 ，比如有哪些表、哪些视图、哪些触发器、哪些列、哪些索引**。这些信息并不是真实的用户数据，而是**一些描述性信息**，有时候也称之为 元数据 。在系统数据库information_schema 中提供了一些以innodb_sys 开头的表，用于表示内部系统表。
- **performance_schema**：MySQL系统自带的数据库，这个数据库里主要保存MySQL服务器运行过程中的一些状态信息，**可以用来监控MySQL服务的各类性能指标** 。包括统计最近执行了哪些语句，在执行过程的每个阶段都花费了多长时间，内存的使用情况等信息。
- **sys**：MySQL系统自带的数据库，这个数据库主要是**通过视图的形式把information_schema和performance_schema结合起来**，帮助系统管理员和开发人员监控MySQL的技术性能。

##### 3.6数据库在文件系统中的表示

使用`create database 数据库名`语句创建一个数据库的时候，在数据库目录/var/lib/mysql下会创建一个和数据库同名的子目录。
**除了information_schema这个系统数据库外，其他的数据库在数据目录下都有对应的子目录。**
[![img](..\db\mysql\mysql目录结构\数据库在文件系统中表示.png)

##### 3.7表在文件系统中的标识

**使用数据库a中的表b描述在INNODB和MYISAM中的区别。**

- db.opt：存储的是数据库的字符集和比较规则
- ibdata1：系统表空间，存储数据和索引信息
- b.frm：描述的是表结构文件、字段长度
- b.xxx.sdi：描述的是表结构文件、字段长度
- b.idb：独立表空间，存储数据和索引信息
- b.MYD(MYDATA)：存储数据信息
- b.MYI（MYINDEX）：存储索引信息

**在INNODB中：聚簇索引（数据和索引在一起）**

- mysql5.5.7（包括）至mysql5.6.6（不包括）版本：使用的是系统表空间
  `存在的文件：db.opt、b.frm、ibdata1`
- mysql5.6.6（包括）至mysql5.7（包括）版本：使用的是独立表空间
  `存在的文件：db.opt、b.frm、b.idb`
- mysql5.7（不包括）至mysql8.0（不包括）版本：使用的是独立表空间
  `存在的文件：b.frm、b.idb`
- mysql8.0以及之后的版本（把frm的信息合并到idb中）
  `存在的文件：b.idb`
  `在b.idb文件的目录下执行以下命令：idb2sdi --dump-flie=b.txt b.idb（查看b.txt存在表结构信息）`

**在MYISAM中：非聚簇索引（数据和索引不在一起）**

- mysql5.5.7（包括）至mysql5.6.6（不包括）版本：使用的是系统表空间
  `存在的文件：db.opt、b.frm、ibdata1`
- mysql5.6.6（包括）至mysql5.7（包括）版本：使用的是独立表空间
  `存在的文件：db.opt、b.frm、b.MYD、b.MYI`
- mysql5.7（不包括）至mysql8.0（不包括）版本：使用的是独立表空间
  `存在的文件：b.frm、b.MYD、b.MYI`
- mysql8.0以及之后的版本（把frm的信息移动到sdi）
  `存在的文件：b.xxx.sdi、b.MYD、b.MYI`

  类型转换
                          0 + CAST(cast(third_size as decimal) / 100 as char)
                          0 + cast(substring_index(substring_index(specification, '*', 1), 'δ', -1) as char)

