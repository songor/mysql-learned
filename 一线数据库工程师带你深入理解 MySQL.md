# 一线数据库工程师带你深入理解 MySQL

### SQL 优化

* 定位慢查询

  * 查看慢查询日志确定已经执行完的慢查询

    MySQL 的慢查询日志用来记录在 MySQL 中响应时间超过参数 long_query_time（单位秒，默认值 10）设置的值并且扫描记录数不小于 min_examined_row_limit（默认值 0）的语句。

    默认情况下，不会记录查询时间不超过 long_query_time 但是不使用索引的语句，可通过配置 log_queries_not_using_indexes = on 让不使用索引的 SQL 都被记录到慢查询日志中。

    如果需要使用慢查询日志，一般分为四步：开启慢查询日志、设置慢查询阀值、确定慢查询日志路径、确定慢查询日志的文件名。

    ```sql
    mysql> set global slow_query_log = on;
    Query OK, 0 rows affected
    
    mysql> set global long_query_time = 1;
    Query OK, 0 rows affected
    
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