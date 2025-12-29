性能分析工具
======

1服务器优化的步骤
---------

2查询系统参数
-------

在MySQL中，可以使用 SHOW STATUS 语句查询一些MySQL数据库服务器的性能参数、执行频率 。  
SHOW STATUS语句语法如下：  
`SHOW [GLOBAL|SESSION] STATUS LIKE '参数';`  
参数如下：

*   Connections：连接MySQL服务器的次数。
*   Uptime：MySQL服务器的上线时间。
*   Slow\_queries：慢查询的次数。
*   Innodb\_rows\_read：Select查询返回的行数
*   Innodb\_rows\_inserted：执行INSERT操作插入的行数
*   Innodb\_rows\_updated：执行UPDATE操作更新的行数
*   Innodb\_rows\_deleted：执行DELETE操作删除的行数
*   Com\_select：查询操作的次数。
*   Com\_insert：插入操作的次数。对于批量插入的 INSERT 操作，只累加一次。
*   Com\_update：更新操作的次数。
*   Com\_delete：删除操作的次数。

3、统计sql语句的查询成本（last\_query\_cost）
---------------------------------

当有多种mysql语句查询方式时，可以依据mysql语句执行后的last\_query\_cost大小（值越小越好）来判断选用那条sql语句。查询语句如下：  
`SHOW STATUS LIKE 'last_query_cost';`

本文转自 <https://www.cnblogs.com/worldusemycode/p/15954936.html>，如有侵权，请联系删除。