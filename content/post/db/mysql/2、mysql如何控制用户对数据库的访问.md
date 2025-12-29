**基础理解：通过对用户赋予某些权限就可以控制用户对数据库的访问  
更深层次的理解：当mysql对用户赋予某些权限时，mysql底层是如何控制用户对数据库的访问**

用户管理和权限管理 （基础理解）
================

用户管理
----

### 1.1登录mysql服务器

启动mysql服务器后，可以通过mysql命令来登录mysql服务器：  
`mysql –h hostname|hostIP –P port –u username –p DatabaseName –e "SQL语句"`

*   \-h参数 后面接主机名或者主机IP，hostname为主机，hostIP为主机IP。
*   \-P参数 后面接MySQL服务的端口，通过该参数连接到指定的端口。MySQL服务的默认端口是3306，不使用该参数时自动连接到3306端口，port为连接的端口号。
*   \-u参数 后面接用户名，username为用户名。
*   \-p参数 会提示输入密码。
*   DatabaseName参数 指明登录到哪一个数据库中。如果没有该参数，就会直接登录到MySQL数据库中，然后可以使用USE命令来选择数据库。
*   \-e参数 后面可以直接加SQL语句。登录MySQL服务器以后即可执行这个SQL语句，然后退出MySQL服务器。

### 1.2创建用户

CREATE USER语句的基本语法形式如下：  
`CREATE USER 用户名 [IDENTIFIED BY '密码'][,用户名 [IDENTIFIED BY '密码']];`

*   用户名参数表示新建用户的账户，由 用户（User） 和 主机名（Host：默认为%，代表所有地址） 构成；
*   “\[ \]”表示可选，也就是说，可以指定用户登录时需要密码验证，也可以不指定密码验证，这样用户可以直接登录。不过，不指定密码的方式不安全，不推荐使用。如果指定密码值，这里需要使用IDENTIFIED BY指定明文密码值。
*   CREATE USER语句可以同时创建多个用户。

### 1.3修改用户

修改用户名：  
`第一步：UPDATE mysql.user SET User='li4' WHERE User='wang5';`  
`第二步：FLUSH PRIVILEGES;`

### 1.4删除用户

两种方式：  
**方式1：使用DROP方式删除（推荐）**  
使用DROP USER语句来删除用户时，必须用于DROP USER权限。DROP USER语句的基本语法形式如下：  
`DROP USER user[,user];`如果host不写默认是%  
**方式2：使用DELETE方式删除**  
第一步：通过delete命令删除用户：  
`DELETE FROM mysql.user WHERE Host='hostnameAND' User='username';`  
第二步：执行完DELETE命令后要使用FLUSH命令来使用户生效，命令如下：  
`FLUSH PRIVILEGES;`

> 注意：不推荐通过 DELETE FROM mysql.user WHERE User='li4' 进行删除，系统会有残留信息保留。而drop user命令会删除用户以及对应的权限，执行命令后你会发现mysql.user表和mysql.db表的相应记录都消失了。

### 1.5用户的密码

#### 1.5.1设置用户的密码

方式一：使用ALTER USER命令来修改当前用户密码，USER()代表当前用户，基本语法如下： `ALTER USER USER() IDENTIFIED BY 'new_password';`  
方式二：使用SET语句来修改当前用户密码，具体SQL语句如下 **（该语句会自动将密码加密后再赋给当前用户。）**：  
`SET PASSWORD='new_password';`

> 方式三：PASSWORD()函数在mysql5.7之后失效，因此不推荐使用。  
> 修改当前用户的密码：（MySQL5.7测试有效）：SET PASSWORD = PASSWORD('123456');

#### 1.5.2修改其他用户的密码

方式一：使用ALTER语句来修改普通用户的密码。基本语法形式如下：  
`ALTER USER user [IDENTIFIED BY '新密码'][,user[IDENTIFIED BY '新密码']]…;`  
方式二：使用SET命令来修改普通用户的密码，使用root用户登录到MySQL服务器后，可以使用SET语句来修改其他普通用户的密码。SET语句的代码如下：  
`SET PASSWORD FOR 'username'@'hostname'='new_password';`

> 方式三：使用UPDATE语句修改普通用户的密码，PASSWORD()函数在mysql5.7之后失效，因此不推荐使用。  
> UPDATE MySQL.user SET authentication\_string=PASSWORD("123456") WHERE User = "username" AND Host = "hostname";

#### 1.5.3密码管理

##### 1.5.3.1密码过期策略

过期策略可以是**全局的** ，也可以为每个账号设置**单独的**过期策略，限制用户密码的有效时间。  
**全局的设置：**  
方式一：使用SQL语句更改该变量的值并持久化  
`SET PERSIST default_password_lifetime = 180; # 建立全局策略，设置密码每隔180天过期`  
方式二：配置文件my.cnf中进行维护  
`[mysqld]`  
`default_password_lifetime=180 #建立全局策略，设置密码每隔180天过期`  
**单独的设置**  
每个账号**既可延用全局密码过期策略，也可单独设置策略**。在 CREATE USER 和 ALTER USER 语句上**加入 PASSWORD EXPIRE 选项可实现单独设置策略**。下面是一些语句示例。

单独密码过期策略的代码

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

##### 1.5.3.2密码重用策略

mysql限制已经使用过的密码，重用策略基于密码更改的数量和使用时间，重用策略可以是**全局的** ，也可以为每个账号设置**单独的**重用策略。  
**全局的设置：**  
方式一：使用SQL语句更改该变量的值并持久化  
`SET PERSIST password_history = 6; #设置不能选择最近使用过的6个密码`  
`SET PERSIST password_reuse_interval = 365; #设置不能选择最近一年内的密码`  
方式二：配置文件my.cnf中进行维护  
`[mysqld]`  
`password_history=6`  
`password_reuse_interval=365`  
**单独的设置**  
每个账号**既可延用全局密码重用策略，也可单独设置策略**。在 CREATE USER 和 ALTER USER 语句上**加入 PASSWORD HISTORY 或者 PASSWORD\_REUSE\_INTERVAL 选项可实现单独设置策略**。下面是一些语句示例。

单独密码重用策略的代码

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

##### 1.5.3.3密码复杂度策略

*   在MySQL8.0之前，MySQL使用的是validate\_password插件检测、验证账号密码强度，保障账号的安全性。
*   MySQL8.0更加安全，引入了服务器组件（Components）这个特性，validate\_password插件已用服务器组件重新实现。8.0.25版本的数据库中，默认自动安装validate\_password组件。

权限管理
----

### 2.1权限列表

使用该命令可以展示出mysql中存在的权限。  
`show privileges;`  
（1）**CREATE和DROP权限**：可以创建新的数据库和表，或删除（移掉）已有的数据库和表。如果将 MySQL数据库中的DROP权限授予某用户，用户就可以删除MySQL访问权限保存的数据库。  
（2）**SELECT、INSERT、UPDATE和DELETE权限**：允许在一个数据库现有的表上实施操作。  
（3）**SELECT权限**：只有在它们真正从一个表中检索行时才被用到。  
（4）**INDEX权限**：允许创建或删除索引，INDEX适用于已有的表。如果具有某个表的CREATE权限，就可以在CREATE TABLE语句中包括索引定义。  
（5）**ALTER权限**：可以使用ALTER TABLE来更改表的结构和重新命名表。  
（6）**CREATE ROUTINE权限**：用来创建保存的程序（函数和程序），ALTER ROUTINE权限：用来更改和删除保存的程序， EXECUTE权限：用来执行保存的程序。  
（7）**GRANT权限**：允许授权给其他用户，可用于数据库、表和保存的程序。  
（8）**FILE权限**：使用户可以使用LOAD DATA INFILE和SELECT ... INTO OUTFILE语句读或写服务器上的文件，任何被授予FILE权限的用户都能读或写MySQL服务器上的任何文件（说明用户可以读任何数据库目录下的文件，因为服务器可以访问这些文件）。  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220123904589-418181875.png)

### 2.2授予权限的原则

**权限控制主要是出于安全因素，因此需要遵循以下几个经验原则 ：**  
1、只授予能**满足需要的最小权限**，防止用户干坏事。比如用户只是需要查询，那就只给select权限就可以了，不要给用户赋予update、insert或者delete权限。  
2、创建用户的时候**限制用户的登录主机**，一般是限制成指定IP或者内网IP段。  
3、为每个用户**设置满足密码复杂度的密码**。  
4、 定期清理不需要的用户，回收权限或者删除用户。

### 2.3授予权限

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

### 2.4查看权限

*   查看当前用户的权限  
    `SHOW GRANTS; 或者 SHOW GRANTS FOR CURRENT_USER; 或者 SHOW GRANTS FOR CURRENT_USER();`
*   查看某些用户的权限  
    `SHOW GRANTS FOR 'user'@'主机地址' ;`

### 2.5收回权限

**收回权限就是取消已经赋予用户的某些权限**。收回用户不必要的权限可以在一定程度上保证系统的安全性。

> MySQL中使用REVOKE语句取消用户的某些权限。使用REVOKE收回权限之后，用户账户的记录将从db、host、tables\_priv和columns\_priv表中删除，但是用户账户记录仍然在user表中保存（删除user表中的账户记录使用DROP USER语句，在将用户账户从user表删除之前，应该收回相应用户的所有权限。）

*   收回权限的命令（收回权限后需要用户重新登录才能生效）  
    `REVOKE 权限1,权限2,…权限n ON 数据库名称.表名称 FROM 用户名@用户地址;`  
    举例：  
    `收回全库全表的所有权限：REVOKE ALL PRIVILEGES ON *.* FROM joe@'%';`  
    `收回mysql库下的所有表的插删改查权限：REVOKE SELECT,INSERT,UPDATE,DELETE ON mysql.* FROM joe@localhost;`

权限表和访问控制（更深层次的理解）
=================

权限表
---

当msyql通过grant命令给用户赋予某些权限时，mysql会把用户的信息保存在相应的权限表中，**mysql服务器通过权限表来控制用户对数据库的访问**，权限表存放在**mysql数据库**中。  
**权限表主要有：user表、db表、table\_priv表、column\_priv表、proc\_priv表。当mysql启动时，服务器会将这些数据库表中的权限信息读入到内存中**

| msyql数据库的表名 | 描述 |
| --- | --- |
| user | 用户账号和权限信息 |
| db | 数据库层级的权限 |
| table\_priv | 表层级的权限 |
| columns\_priv | 列层级的权限 |
| procs\_priv | 存储的过程和函数的权限 |

### 3.1user表

**user表：是MySQL中最重要的一个权限表， 记录用户账号和权限信息。**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220154814619-880344668.png)  
这些字段可以分成4类，分别是范围列（或用户列）、权限列、安全列和资源控制列。  
**1、范围列（或用户列）**

*   host：表示连接类型  
    `1、% 表示所有远程通过 TCP方式的连接`  
    `2、IP 地址 如 (192.168.1.2、127.0.0.1) 通过制定ip地址进行的TCP方式的连接`  
    `3、机器名 通过制定网络中的机器名进行的TCP方式的连接`  
    `4、::1 IPv6的本地ip地址，等同于IPv4的 127.0.0.1`  
    `5、localhost 本地方式通过命令行方式的连接 ，比如mysql -u xxx -p xxx 方式的连接。`
*   user：表示用户名，同一用户通过不同方式链接的权限是不一样的。
*   password：密码  
    `1、所有密码串通过 password(明文字符串) 生成的密文字符串。mysql 5.7 及之后版本的密码保存到 authentication_string 字段中不再使用password字段。`  
    `2、MySQL 8.0 在用户管理方面增加了角色管理，默认的密码加密方式也做了调整，由之前的 SHA1 改为了 SHA2 ，不可逆 。同时加上MySQL 5.7的禁用用户和用户过期的功能，MySQL在用户管理方面的功能和安全性都较之前版本大大的增强了。`

**2、权限列**

*   Grant\_priv字段：表示是否拥有GRANT权限
*   Shutdown\_priv字段：表示是否拥有停止MySQL服务的权限
*   Super\_priv字段：表示是否拥有超级权限
*   Execute\_priv字段：表示是否拥有EXECUTE权限。拥有EXECUTE权限，可以执行存储过程和函数。
*   Select\_priv , Insert\_priv：为该用户所拥有的权限。

**3、安全列**

*   ssl\_type、ssl\_cipher：用于加密 ；
*   x509\_issuer、x509\_subject：用于标识用户 ；
*   Plugin字段：用于验证用户身份的插件，该字段不能为空。如果该字段为空，服务器就使用内建授权验证机制验证用户身份。

**4、资源控制列**  
**资源控制列的字段用来限制用户使用的资源。**

*   max\_questions，用户每小时允许执行的查询操作次数；
*   max\_updates，用户每小时允许执行的更新操作次数；
*   max\_connections，用户每小时允许执行的连接操作次数；
*   max\_user\_connections，用户允许同时建立的连接次数。

### 3.2db表

**db表：记录数据库层级的权限**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220152406519-649151221.png)  
**1、用户列**  
db表用户列有3个字段，分别是Host、User、Db。这3个字段分别表示主机名、用户名和数据库  
名。表示从某个主机连接某个用户对某个数据库的操作权限，这3个字段的组合构成了db表的主键。  
**2、权限列**  
Create\_routine\_priv和Alter\_routine\_priv这两个字段决定用户是否具有创建和修改存储过程的权限。

### 3.3tables\_priv表

**tables\_priv表：记录表层级的权限**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220152825187-262801799.png)

*   Host、Db、User 和 Table\_name 四个字段分别表示主机名、数据库名、用户名和表名。表示从某个主机连接某个用户对某个数据库中的某个表的操作权限，这4个字段的组合构成了tables\_priv表的主键。
*   Grantor表示修改该记录的用户。
*   Timestamp表示修改该记录的时间。
*   Table\_priv表示对表的操作权限。包括Select、Insert、Update、Delete、Create、Drop、Grant、References、Index和Alter。
*   Column\_priv表示对表中的列的操作权限，包括Select、Insert、Update和References。

### 3.4columns\_priv表

**columns\_priv表：记录列层级的权限**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220153703503-765466053.png)

*   Host、Db、User、Table\_name和column\_name 5个字段分别表示主机名、数据库名、用户名、表名和列名。表示从某个主机连接某个用户对某个数据库中的某个表中的某个列的操作权限，这5个字段的组合构成了columns\_priv表的主键。
*   Timestamp表示修改该记录的时间。
*   Column\_priv表示对表中的列的操作权限，包括Select、Insert、Update和References。

### 3.5procs\_priv表

**procs\_priv表：存储的过程和函数的权限**  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220154121398-1977529702.png)

*   Host、Db、User、Routine\_name和Routine\_type 5个字段分别表示主机名、数据库名、用户名、存储过程名和存储类型。表示从某个主机连接某个用户对某个数据库中的某个存储过程的操作权限，这5个字段的组合构成了procs\_priv表的主键。
*   Grantor表示修改该记录的用户。
*   proc\_priv表示对存储过程的操作权限，包括execute、alter routine、grant。
*   Timestamp表示修改该记录的时间。

访问控制
----

### 4.1连接核实阶段

客户端用户会在连接请求中提供用户名、主机地址、用户密码，MySQL服务器接收到用户请求后，**会使用user表中的host、user和authentication\_string这3个字段匹配客户端提供信息**。如果连接核实没有通过，服务器就完全拒绝访问；否则，服务器接受连接，然后进入请求核实阶段。

### 4.2请求核实阶段

一旦建立了连接，服务器就进入请求核实阶段。对此连接上进来的每个请求，服务器检查该请求要执行什么操作、是否有足够的权限来执行它，这正是需要授权表中的权限列发挥作用的地方。这些权限可以来自user、db、table\_priv和column\_priv表。  
![image](https://img2022.cnblogs.com/blog/2345397/202202/2345397-20220220161703950-393897996.png)

本文转自 <https://www.cnblogs.com/worldusemycode/p/15915303.html>，如有侵权，请联系删除。