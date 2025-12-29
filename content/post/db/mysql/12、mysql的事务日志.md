mysql的事务日志
==========

**事务有4种特性：原子性、一致性、隔离性和持久性。那么事务的四种特性到底是基于什么机制实现呢？**

*   事务的隔离性由**锁机制**实现。
*   事务的原子性、一致性和持久性由事务的**redo日志和undo日志**来保证。  
    `REDO LOG称为重做日志：提供再写入操作，恢复提交事务修改的页操作，用来保证事务的持久性。`  
    `UNDO LOG称为回滚日志：事务的回滚(undo log记录的是每个修改操作的逆操作)和一致性非锁定读(undo log回滚行记录到某种特定的版本)，用来保证事务的原子性、一致性。`

**有的同学认为undo log是redo log的逆过程，这种理解是错误的。**

*   redo log：是存储引擎层(innodb)生成的日志，记录的是"物理级别"上的页修改操作，比如页号、偏移量、数据，主要为了保证数据的可靠性
*   undo log：是存储引擎层(innodb)生成的日志，记录的是逻辑操作日志，并不是真正回滚到执行该事务之前的物理状态，比如对某一行数据进行了INSERT语句操作，那么undo log就记录一条与之相反的DELETE操作，

redo日志
------

### 1.1为什么需要redo日志

**1、在传统的情况下**：把更改的数据写到缓冲池，当执行提交事务操作后，数据库宕机了，缓冲池中的数据就会丢失、无法把数据刷入到磁盘上，违反了事务的持久性。  
`事务的持久性：一个事务一旦被提交，它对数据库中数据的改变就是永久性的，接下来的其他操作和数据库故障不应该对其有任何影响。`  
![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220304231750490-563015138.png)  
**2、如何保证事务的持久性（两种方式）：**  
**方式一（简单的方式）**：在事务提交完成之前把该事务所修改的所有页面都刷新到磁盘，这样就可以保持事务的持久性。但是存在问题：  
`1、innodb是以页作为磁盘和内存交互的基本单位，当仅仅修改某个页面中的一个字节时，就需要将整个页面刷新到磁盘当中，这样会增加磁盘IO的时间，使服务器性能降低。`  
`2、当执行一条sql语句时可能会修改很多页面，这些页面可能是不相邻的，在将缓冲池中修改过的页面刷新到磁盘上时，需要进行随机IO，随机IO比顺序IO慢。`

**方式二（使用redo日志）**：**我们只是想让已经提交了的事务对数据库中数据所做的修改永久生效**，即使后来系统崩溃，在重启后也能把这种修改恢复出来。所以我们其实没有必要在每次事务提交时就把该事务在内存中修改过的全部页面刷新到磁盘，只需要使用redo日志把修改的数据记录下来，当系统重启后直接使用redo日志重新执行。

### 1.2redo日志的特点

*   redo日志降低了刷盘频率
*   redo日志占用的空间非常小：  
    `存储表空间ID、页号、偏移量、需要更新的值，因此使用的存储空间小`
*   redo日志是顺序写入磁盘的：  
    `在执行事务的过程中，每执行一条语句，就可能产生若干条redo日志，这些日志是按照产生的顺序写入到磁盘中，也就是使用顺序IO写入`
*   事务执行过程中，redo log不断记录：  
    `在事务执行过程中redo log可以不断写入redo log file文件中，而binlog只有在提交事务时才可以写入bin log file文件中，所以redo log与binlog的写入时机不一样。`

### 1.3redo的组成

**redo log可以分为两个部分：**

*   重做日志缓冲（redo log buffer）：保存在内存中，容易丢失。  
    `默认大小是16M，最大值是4096M，最小值为1M`  
    ![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305010921603-296008198.png)
    
*   重做日志文件（redo log file）：保存在硬盘中，是持久的。  
    `在/var/lib/mysql目录下可以查看到ib_logfile0和ib_logfile1日志文件`  
    ![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305011410267-1776523466.png)
    

### 1.4redo的整体流程

![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305012619815-985895468.png)

*   第1步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝
*   第2步：生成一条redo log重做日志并写入redo log buffer，记录的是数据被修改后的值
*   第3步：当事务commit时，将redo log buffer中的内容刷盘到 redo log file，对 redo log file采用追加写的方式
*   第4步：定期将内存中修改的数据刷新到磁盘中。当发生宕机且数据未刷到磁盘的时候，可以使用redo log来恢复，保证ACID的持久性。

### 1.5redo log刷盘策略

**1、redo log的写入并不是直接写入磁盘的，InnoDB引擎会在写redo log的时候先写入redo log buffer缓冲区，之后按照刷盘策略（innodb\_flush\_log\_at\_trx\_commit）刷入到真正的redo log file中。**

> redo log buffer刷盘到redo log file的过程并不是真正的刷到磁盘中去，只是写入到文件系统缓存（page cache）中去，真正的刷盘到redo log file中会交给系统自己来决定

**innodb\_flush\_log\_at\_trx\_commit （默认值为1）**：表示何时将缓冲区的数据写入日志文件，并且将日志文件写入磁盘中。该参数对于innoDB引擎非常重要。**（系统默认master thread每隔1秒会将数据写入日志文件并将日志文件写入磁盘）**

*   **值为 0 时**，表示**每隔1秒**将`数据写入日志文件（把redo log buffer中的redo日志写入到page cache，再将page cache中的redo日志刷盘到redo log file日志文件中）`并将`日志文件写入磁盘`。该模式速度最快，但不太安全，**在mysqld进程的崩溃或者操作系统宕机情况下会导致redo logo buffer中所有已提交事务的数据丢失，违反了事务的持久性**。  
    ![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305120734445-1549959344.png)
    
*   **值为 1 时**，表示**每次提交事务时**将`数据写入日志文件（把redo log buffer中的redo日志写入到page cache，再将page cache中的redo日志刷盘到redo log file日志文件中）`并将`日志文件写入磁盘`。该模式是最安全的，但也是最慢的一种方式。  
    ![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305122450318-1794178965.png)
    
*   **值为 2 时**，表示**每次提交事务时**将`数据写入日志文件（只是把redo log buffer中的redo日志写入到page cache；并不会把page cache中的redo日志刷盘到redo log file日志文件，该操作交给系统默认的master thread去执行）`，**每隔1秒**将`日志文件写入磁盘`。该模式速度较快，也比0安全一点，**在操作系统宕机的情况下会导致page cache中上一秒钟所有已提交事务的数据丢失，违反了事务的持久性**。  
    ![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305123737346-656901832.png)
    

### 1.7写入redo log buffer中的过程

#### 1.7.1Mini-Transaction

MysQL把对底层页面中的一次原子访问的过程称之为一个Mini-Transaction，简称mtr。一个事务可以包含若干条语句，每一条语句可能操作若干个页面，每个页面对应一个mtr，一个mtr可以包含一组redo日志，在进行崩溃恢复时这一组redo日志作为一个不可分割的整体。  
![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305125331323-1226600094.png)

#### 1.7.2redo 日志写入redo log buffer中

`1、一个mtr执行过程中可能产生若干条redo日志，这些redo日志是一个不可分割的组，所以其实并不是每生成一条redo日志，就将其插入到log buffer中，而是每个mtr运行过程中产生的日志先暂时存到一个地方，当该mtr结束的时候，将过程中产生的一组redo日志再全部复制到log buffer中。`

`2、向redo log buffer中写入每组redo日志的过程是顺序的，也就是先往前边的block中写，当该block的空闲空间用完之后再往下一个block中写。buf_free全局变量指明后续写入的redo日志应该写入到redo log buffer中的哪个位置，如图所示:`  
![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305130256353-127583212.png)

#### 1.7.3redo log block的结构

**一个redo log block是由日志头、日志体、日志尾组成**。日志头占用12字节，日志尾占用8字节，日志体来真正存储的数据占用512-12-8=492字节。  
![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305130711160-1748996978.png)  
**为什么将redo log block设置为512字节**

> 这个和磁盘的扇区有关，机械磁盘默认的扇区就是512字节，如果你要写入的数据大于512字节，那么要写入的扇区肯定不止一个，这时就要涉及到盘片的转动，找到下一个扇区，假设现在需要写入两个扇区A和B，如果扇区A写入成功，而扇区B写入失败，那么就会出现非原子性的写入，而如果每次只写入和扇区的大小一样的512字节，那么每次的写入都是原子性的。

### 1.8 redo log file

1.8.1 参数设置

*   innodb\_log\_group\_home\_dir ：指定 redo log 文件组所在的路径，默认值为 ./ ，表示在数据库的数据目录下。MySQL的默认数据目录（ var/lib/mysql ）下默认有两个名为 ib\_logfile0 和ib\_logfile1 的文件，log buffer中的日志默认情况下就是刷新到这两个磁盘文件中。此redo日志文件位置还可以修改。
*   innodb\_log\_files\_in\_group：指明redo log file的个数，命名方式如：ib\_logfile0，iblogfile1...iblogfilen。默认2个，最大100个。  
    ![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305131233059-1475738030.png)
*   innodb\_log\_file\_size：单个 redo log 文件设置大小，默认值为 48M 。最大值为512G，注意最大值指的是整个 redo log 系列文件之和，即（innodb\_log\_files\_in\_group \*innodb\_log\_file\_size ）不能大于最大值512G。  
    ![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305131347840-74695278.png)

#### 1.8.2日志文件组

*   总共的redo日志文件大小其实就是: innodb\_log\_file\_size x innodb\_log\_files\_in\_group。
*   采用循环使用的方式向redo日志文件组里写数据  
    ![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305131630451-2013426184.png)

#### 1.8.3 checkpoint

1、checkpoint的作用：当采用循环使用的方式向redo日志文件组里写数据，保证不会覆盖以前写入的redo日志。  
2、每次刷盘redo log记录到日志文件组中，write pos位置就会后移更新。每次MySQL加载日志文件组恢复数据时，会清空加载过的redo log记录，并把checkpoint后移更新。write pos和checkpoint之间的还空着的部分可以用来写入新的redo log记录。  
3、当write pos追上checkpoint，表示日志文件组满了，这时候不能再写入新的redo log记录，需要等待mysql清空一些redo log记录，把check point往后移动。  
![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305132204665-996825363.png)

undo日志
------

redo log是事务持久性的保证，undo log是事务原子性的保证。**在事务中更新数据之前先把旧的数据写入一个undo log。**

### 2.1undo日志的理解

**1、在事务执行到一半的时候，当出现服务器本身的错误、操作系统错误、断电、程序员手动输入rollback情况时，需要使用undo 日志进行回滚操作，来保证事务的原子性，从逻辑上看该事务什么也没有做。**

### 2.2undo日志中存储了哪些数据

**当我们要对一条记录做改动时(这里的改动可以指INSERT、DELETE、UPDATE)需要往redo日志中记录一些数据：** `undo log的产生会伴随着redo log的产生，这是因为undo log也需要持久性的保护，在 重启之后还可以使用undo log进行回滚操作。`

*   你插入了一条记录时，至少要把这条记录的主键值记录下来，之后回滚的时候只需要把这个主键值对应的记录删掉就好了。(对于每个INSERT，lnnoDB存储引擎会完成一个DELETE)
*   你删除了一条记录时，至少要把这条记录中的内容都记录下来，这样之后回滚时再把由这些内容组成的记录插入到表中就好了。(对于每个DELETE，InnoDB存储引擎会执行一个INSERT)
*   你更新了一条记录时，至少要把修改这条记录前的旧值都记录下来，这样之后回滚时再把这条记录更新为旧值就好了。(对于每个UPDATE，InnoDB存储引擎会执行一个相反的UPDATE，将修改前的行放回去)

> 注意，由于查询操作( SELECT ）并不会修改任何用户记录，所以在查询操作执行时，并不需要记录相应的undo日志。

### 2.3undo的类型

**在InnoDB存储引擎中,undo log分为:**

*   **insert undo log是指在insert操作中产生的undo log**。因为insert操作的记录，只对事务本身可见，对其他事务不可见(这是事务隔离性的要求)，**故该undo log可以在事务提交时直接删除**。不需要purge线程删除。
*   **update undo log记录的是对delete和update操作产生的undo log**。**该undo log可能需要提供MVCC机制，因此不能在事务提交时就进行删除**。提交事务时**放入undo log链表**，等待purge线程进行最后的删除。

### 2.4undo的存储结构

#### 2.4.1回滚段与undo页

##### 回滚段

**InnoDB对undo log的管理采用段的方式，也就是回滚段（rollback segment）**。每个回滚段记录了1024个undo log segment ，而在每个undo log segment段中进行undo页的申请。

*   在 InnoDB1.1版本之前（不包括1.1版本），只有一个rollback segment，因此支持同时在线的事务限制为1024。虽然对绝大多数的应用来说都已经够用。
*   从1.1版本开始InnoDB支持最大128个rollback segment，故其支持同时在线的事务限制提高到了 128\*1024 。

虽然InnoDB1.1版本支持了128个rollback segment，但是这些rollback segment都存储于共享表空间ibdata中。从lnnoDB1.2版本开始，可通过参数对rollback segment做进一步的设置。这些参数包括:

*   innodb\_undo\_directory :设置rollback segment文件所在的路径。这意味着rollback segment可以可以自己设置独立表空间。该参数的默认值为“./"，表示当前InnoDB存储引擎的目录。
*   innodb\_undo\_logs :设置rollback segment的个数，默认值为128。在InnoDB1.2版本中，该参数用来替换之前版本的参数innodb\_rollback\_segments。
*   innodb\_undo\_tablespaces:设置存储rollback segment的文件数量，这样rollback segment可以较为平均地分布在多个文件中。设置该参数后，会在路径innodb\_undo\_directory看到undo为前缀的文件，该文件就代表存储rollback segment的文件。

##### undo页的重用

**前提：当我们开启一个事务需要写undo log的时候，就得先去undo log segment中去找到一个空闲的位置，有空位时就申请undo页，把undo log的写入到申请到的undo页。我们知道mysql默认一页的大小是16k。为每一个事务分配一个页，是非常浪费的（除非你的事务非常长)。**  
`于是undo页就被设计的可以重用了，当事务提交时，并不会立刻删除undo页。因为重用，所以这个undo页可能混杂着其他事务的undo log。undo log在事务commit时，会被放到一个undo log链表中，然后判断undo页的使用空间是否小于3 /4，如果小于3/4的话，则表示当前的undo页可以被重用，分配给其他事务使用，其他事务的undo log可以记录在当前undo页的后面。由于undo log是离散的，所以清理对应的磁盘空间时，效率不高。`

#### 2.4.2回滚段与事务的关系

1.  每个事务只会使用一个回滚段，一个回滚段在同一时刻可能会服务于多个事务。
2.  当一个事务开始的时候，会制定一个回滚段，在事务进行的过程中，当数据被修改时，原始的数据会被复制到回滚段。
3.  在回滚段中，事务会不断填充盘区，直到事务结束或所有的空间被用完。如果当前的盘区不够用，事务会在段中请求扩展下一个盘区，如果所有已分配的盘区都被用完，事务会覆盖最初的盘区或者在回滚段允许的情况下扩展新的盘区来使用。
4.  回滚段存在于undo表空间中，在数据库中可以存在多个undo表空间，但同一时刻只能使用一个undo表空间。
5.  当事务提交时，InnoDB存储引擎会做以下两件事情：  
    `将undo log放入undo log链表中，可以提供MVCC，等待purge线程进行最后的删除`  
    `判断undo log所在的页是否可以重用，若可以分配给下个事务使用`

#### 2.4.3回滚段中的数据分类

*   `未提交事务的回滚数据(uncommitted undo information)`：该数据所关联的事务并未提交，用于实现读一致性，所以该数据不能被其他事务的数据覆盖。
*   `已经提交事务但未过期的回滚数据(committed undo information)相当于是undo log链表中未过期的数据`：该数据关联的事务已经提交，但是仍受到undo retention参数的保持时间的影响。
*   `已经提交事务并过期的数据(expired undo information)相当于是undo log链表中已经过期的数据`：事务已经提交，而且数据保存时间已经超过undo retention参数指定的时间，属于已经过期的数据。当回滚段满了之后，会优先覆盖"事务已经提交并过期的数据"。

### 2.5undo日志的作用

*   回滚数据  
    `undo是逻辑日志，因此只是将数据库逻辑地恢复到原来的样子，所有修改都被逻辑地取消了，数据结构和页本身在回滚之后可能和执行该事务开始的状态存在差异。逻辑上不能将一个页回滚到事务开始的样子，因为这样会影响其他事务正在进行的工作。`
*   MVCC  
    `在InnoDB存储引擎中MVCC的实现是通过undo日志来完成。当用户读取一行记录时，若该记录已经被其他未提交的事务正在占用，当前事务可以通过undo读取之前已经提交事务的行版本信息，以此实现非锁定读取。`

### 2.6undo日志是如何生成的

**通过回滚指针DB\_ROLL\_PTR指向每次更改前的数据，形成一条undo log日志记录**  
![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305221904047-75385510.png)

当事务使用undo日志和redo日志的示意图
----------------------

**undo日志用于保证原子性，redo日志保证持久性**  
![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305221054382-1885532325.png)

### 总结

`undo log是逻辑日志，对事务回滚时，只是将数据库逻辑地恢复到原来的样子。`  
`redo log是物理日志，记录的是数据页的物理变化，undo log不是redo log的逆过程。`  
![image](https://img2022.cnblogs.com/blog/2345397/202203/2345397-20220305222606448-914790459.png)

本文转自 <https://www.cnblogs.com/worldusemycode/p/15966502.html>，如有侵权，请联系删除。