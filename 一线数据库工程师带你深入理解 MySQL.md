# 一线数据库工程师带你深入理解 MySQL

### SQL 优化

* 定位慢查询

  * 查看慢查询日志确定已经执行完的慢查询

    MySQL 的慢查询日志用来记录在 MySQL 中响应时间超过参数 long_query_time（单位秒，默认值 10）设置的值并且扫描记录数不小于 min_examined_row_limit（默认值 0）的语句。

    默认情况下，不会记录查询时间不超过 long_query_time 但是不使用索引的语句，可通过配置 log_queries_not_using_indexes = on 让不使用索引的 SQL 都被记录到慢查询日志中。

    如果需要使用慢查询日志，一般分为四步：开启慢查询日志、设置慢查询阀值、确定慢查询日志路径、确定慢查询日志的文件名。

    ```sql
    mysql> set global slow_query_log = on;
    
    mysql> set global long_query_time = 1;
    
    mysql> show global variables like 'datadir';
    
    mysql> show global variables like 'slow_query_log_file';
    ```
    
  * show processlist 查看正在执行的慢查询

* 使用 explain 分析慢查询

  * select_type

  * type

    | 值              | 解释                                                         |
    | --------------- | ------------------------------------------------------------ |
    | system          | 查询对象表只有一行数据，且只能用于 MyISAM 和 Memory 引擎的表，这是最好的情况 |
    | const           | 基于主键或唯一索引查询，最多返回一条结果                     |
    | eq_ref          | 表连接时基于主键或非 NULL 的唯一索引完成扫描                 |
    | ref             | 基于普通索引的等值查询，或者表间等值连接                     |
    | fulltext        | 全文检索                                                     |
    | ref_or_null     | 表连接类型是 ref，但进行扫描的索引列中可能包含 NULL 值       |
    | index_merge     | 利用多个索引                                                 |
    | unique_subquery | 子查询中使用唯一索引                                         |
    | index_subquery  | 子查询中使用普通索引                                         |
    | range           | 利用索引进行范围查询                                         |
    | index           | 全索引扫描                                                   |
    | ALL             | 全表扫描                                                     |

  * Extra

    | 值                                    | 解释                                                         |
    | ------------------------------------- | ------------------------------------------------------------ |
    | Using filesort                        | 将用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘完成排序 |
    | Using temporary                       | 需要创建一个临时表来存储结构，通常发生对没有索引的列进行 GROUP BY 时 |
    | Using index                           | 使用覆盖索引                                                 |
    | Using where                           | 使用 where 语句来处理结果                                    |
    | Impossible WHERE                      | 对 where 子句判断的结果总是 false 而不能选择任何数据         |
    | Using join buffer (Block Nested Loop) | 关联查询中，被驱动表的关联字段没索引                         |
    | Using index condition                 | 先条件过滤索引，再查数据                                     |
    | Select tables optimized away          | 使用某些聚合函数（比如 max、min）来访问存在索引的某个字段时  |

* show profile 分析慢查询

  可以通过配置参数 profiling = 1 来启用 SQL 分析。该参数开启后，后续执行的 SQL 语句都将记录其资源开销，如 IO、上下文切换、CPU、Memory 等等。

  ```sql
  mysql> select @@have_profiling;
  
  mysql> select @@profiling;
  
  mysql> set profiling = 1;
  
  mysql> SQL
  
  mysql> show profiles;
  
  mysql> show profile for query 1;
  ```

* trace 分析 SQL 优化器

  ```sql
  mysql> set session optimizer_trace = "enabled=on", end_markers_in_json = on;
  
  mysql> SQL
  
  mysql> select * from information_schema.OPTIMIZER_TRACE;
  
  mysql> set session optimizer_trace = "enabled=off";
  ```

  TRACE 字段中整个文本大致分为三个过程，准备阶段（join_preparation），优化阶段（join_optimization），执行阶段（join_execution），使用时，重点关注优化阶段和执行阶段。

  table_scan - 全表扫描

  potential_range_indexes - 列出表中所有的索引并分析其是否可用

  group_index_range - 评估在使用了 GROUP BY 或者是 DISTINCT 的时候是否有适合的索引

  analyzing_range_alternatives - 分析可选方案的代价

  chosen_range_access_summary - 确认最后的方案

  considered_execution_plans - 对比各可行计划的代价，选择相对最优的执行计划

  * MySQL 常见排序模式

    <sort_key, rowid> 双路排序：首先根据相应的条件取出相应的排序字段和可以直接定位行数据的行 ID，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段。

    <sort_key, additional_fields> 单路排序：是一次性取出满足条件行的所有字段，然后在 sort buffer 中进行排序。

    <sort_key, packed_additional_fields> 打包数据排序：将 char 和 varchar 字段存到 sort buffer 中时，更加紧缩。

    第二种模式相对第一种模式，避免了二次回表，可以理解为用空间换时间。由于 sort buffer 有限，如果需要查询的数据比较大的话，会增加磁盘排序时间，效率可能比第一种方式更低。

    MySQL 提供了一个参数：max_length_for_sort_data，当“排序的键值对大小” > max_length_for_sort_data 时，MySQL 认为磁盘外部排序的 IO 效率不如回表的效率，会选择第一种模式，否则，会选择第二种模式。

    第三种模式主要解决变长字符数据存储空间浪费的问题。

  * 优化器估计符合条件的行数

    index diver

    index statistics

* 条件字段有索引但是查询不走索引导致查询慢

  * 函数操作

    ```sql
    where date(current_date) = '2019-05-21';
    
    where current_date >= '2019-05-21 00:00:00' and current_date <= '2019-05-21 23:59:59';
    ```

  * 隐式转换

    ```sql
    where tele_phone = 13032940358;
    
    where cast(tele_phone as signed int) = 13032940358;
    
    where tele_phone = '13032940358';
    ```

  * 模糊查询

    ```sql
    like '%abc%';
    
    like 'abc%'
    ```

  * 范围查询

    优化器会根据检索比例、表大小、I/O 块大小等进行评估是否使用索引。比如单次查询的数据量过大，优化器将不走索引。

    降低单次查询范围，分多次查询。

  * 计算操作

    ```sql
    where a - 1 = 1000;
    
    where a = 1000 + 1;
    ```

    一般需要对条件字段做计算时，建议通过程序代码实现，而不是通过 MySQL 实现。如果在 MySQL 中计算的情况避免不了，那必须把计算放在等号后面。

* 优化批量数据导入

  * 一次插入多行的值

    有大批量导入时，推荐一条 insert 语句插入多行数据。

  * 关闭自动提交

    ```sql
    set autocommit = 0;
    SQL
    commit;
    ```

    批量导入大部分时间耗费在客户端与服务端通信，所以多条 insert 语句合并提交可以减少客户端与服务端通信的时间，并且合并提交还可以减少数据落盘的次数。

  * 参数调整

    innodb_flush_log_at_trx_commit，控制 redo log 刷新到磁盘的策略

    0 - master 线程每秒把 redo log buffer 写到操作系统缓存，再刷到磁盘

    1 - 每次事务提交都将 redo log buffer 写到操作系统缓存，再刷到磁盘

    2 - 每次事务提交都将 redo log buffer 写到操作系统缓存，由操作系统来管理刷盘

    sync_binlog，控制 bin log 的刷盘时机

    0 - 二进制日志从不同步到磁盘，依赖 OS 刷盘机制

    1 - 二进制日志每次提交都会刷盘

    n -  每 n 次提交落盘一次

    innodb_flush_log_at_trx_commit 设置为 0，同时 sync_binlog 设置为 0 时，写入数据的速度是最快的。如果对数据库安全性要求不高（测试环境），可以尝试都设置为 0 后再导入数据，能大大提升导入速度。

* 