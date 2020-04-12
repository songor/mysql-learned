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

* order by 原理

  * MySQL 排序方式

    通过有序索引直接返回有序数据（Extra - Using index）

    通过 Filesort 进行排序（Extra - Using filesort）

  * Filesort 内存排序 or 磁盘排序

    内存排序还是磁盘排序取决于排序的数据大小和  sort_buffer_size 配置的大小

    内存排序：“排序的数据大小” < sort_buffer_size

    磁盘排序：“排序的数据大小” > sort_buffer_size

    TRACE（filesort_summary）：number_of_tmp_files - 使用临时文件的个数

  * Filesort 排序模式

    单路排序会把所有需要查询的字段都放到 sort buffer 中，而双路排序只会把主键和需要排序的字段放到 sort buffer 中进行排序，然后再通过主键回到原表查询需要的字段。

    如果 MySQL 排序内存配置的比较小并且没有条件继续增加了，可以适当把 max_length_for_sort_data 配置小点，让优化器选择使用双路排序算法，可以在 sort_buffer 中一次排序更多的行，只是需要再根据主键回到原表取数据。

    如果 MySQL 排序内存有条件可以配置比较大，可以适当增大 max_length_for_sort_data 的值，让优化器优先选择单路排序，把需要的字段放到 sort_buffer 中，这样排序后就会直接从内存里返回查询结果了。

* order by 优化

  * 添加合适索引

    在排序字段上添加索引来优化排序语句。

    对于多个字段排序，可以在多个排序字段上添加联合索引来优化排序语句。

    对于先等值查询再排序的语句，可以通过在条件字段和排序字段添加联合索引来优化此类排序语句。

  * 去掉不必要的返回字段

    扫描整个索引并查找到没索引的行的成本比扫描全表的成本更高，所以优化器放弃使用索引。

  * 修改参数

    max_length_for_sort_data，sort_buffer_size

  * 避免无法利用索引排序的情况

    使用范围查询再排序：a、b 两个字段的联合索引，对于单个 a 的值，b 是有序的。而对于 a 字段的范围查询，也就是 a 字段会有多个值，取到 a，b 的值 b 就不一定有序了，因此要额外进行排序。

    ASC 和 DESC 混合使用将无法使用索引

* group by 优化

  默认情况，会对 group by 字段排序，因此优化方式与 order by 基本一致，如果目的只是分组而不用排序，可以指定 order by null 禁止排序。

* 根据自增且连续主键排序的分页查询

  主键自增且连续；结果是按照主键排序的

  ```sql
  select * from t limit 99000, 2;
  
  select * from t where id > 99000 limit 2;
  ```

* 查询根据非主键字段排序的分页查询

  ```sql
  select * from t order by a limit 99000, 2;
  ```

  扫描整个索引并查找到没索引的行的成本比扫描全表的成本更高，所以优化器放弃使用索引。

  让排序时返回的字段尽可能少，所以可以让排序和分页操作先查出主键，然后根据主键查到对应的记录。

  ```sql
  select * from t s0 inner join (select id from t order by a limit 99000, 2) s1 on s0.id = s1.id;
  ```

* 关联查询算法

  * Nested-Loop Join 算法

    一个简单的 Nested-Loop Join(NLJ) 算法一次一行循环地从第一张表（称为驱动表）中读取行，在这行数据中取到关联字段，根据关联字段在另一张表（被驱动表）里取出满足条件的行，然后取出两张表的结果合集。

    MySQL 在关联字段有索引时，才会使用 NLJ，如果没索引，就会使用 Block Nested-Loop Join。

    explain 分析 join 语句时，在第一行的就是驱动表。

    如果没固定连接方式（比如没加 straight_join），优化器会优先选择小表做驱动表，所以使用 inner join 时，前面的表并不一定就是驱动表。

    一般 join 语句中，如果执行计划 Extra 中未出现 Using join buffer (Block Nested Loop)，则表示使用的 join 算法是 NLJ。

  * Block Nested-Loop Join 算法

    把驱动表的数据读入到 join_buffer 中，然后扫描被驱动表，把被驱动表每一行取出来跟 join_buffer 中的数据做对比，如果满足 join 条件，则返回结果给客户端。

  * Batched Key Access 算法

* 优化关联查询

  * 关联字段添加索引

    建议在被驱动表的关联字段上添加索引，让 BNL 变成 NLJ 或者 BKA，可明显优化关联查询。

  * 小表驱动大表

    当使用 Index Nested-Loop Join 算法时，扫描行数跟驱动表的数据量成正比。所以在写 SQL 时，如果确定被关联字段有索引的情况下，建议用小表做驱动表。

  * 临时表

    当遇到 BNL 的 join 语句，如果不方便在关联字段上添加索引，不妨尝试创建临时表，然后在临时表中的关联字段上添加索引，通过临时表来做关联查询。

* 重新认识 count()

  * count(a) 和 count(\*) 的区别

    count(a) 不统计 null，而 count(\*) 无论是否包含空值，都会统计。

  * MyISAM 引擎和 InnoDB 引擎 count(\*) 的区别

    MyISAM 会维护表的总行数，放在磁盘中，如果有 count(\*) 的需求，直接返回这个数据（Extra - Select tables optimized away）。

    InnoDB 会去遍历普通索引树，计算表数据总量。

  * MySQL 5.7.18 前后 count(\*) 的区别

    在 MySQL 5.7.18 之前，InnoDB 通过扫描聚簇索引来处理 count(\*) 语句。

    从 MySQL 5.7.18 开始，通过遍历最小的可用二级索引来处理 count(\*) 语句。如果不存在二级索引，则扫描聚簇索引。

    InnoDB 二级索引树的叶子节点上存放的是主键，而主键索引树的叶子节点上存放的是整行数据，所以二级索引树比主键索引树小。因此优化器基于成本的考虑，优先选择的是二级索引。所以 count(主键) 其实没 count (\*) 快。

  * count(1) 比 count(\*) 快吗？

    count(1) 中的 1 是恒真表达式，因此也会统计所有结果。

    count(1) 和 count(\*) 统计结果没差别。

* 哪些方法可以加快 count()

  * show table status

    ```sql
    show table status like 't';
    ```

    能快速获取结果，但是结果不准确。

  * 用 Redis 做计数器

    能快速获取结果，比 show table status 结果准确，但是并发场景计数可能不准确。

  * 增加 InnoDB 计数表

    能快速获取结果，利用了事务特性确保了计数的准确，也是比较推荐的方法。

### MySQL 索引

* 跟索引相关的一些算法

  * 二分查找法

    将记录按顺序排列，查找时先以有序列的中点位置为比较对象，如果要找的元素值小于该中点元素，则将查询范围缩小为左半部分；如果要找的元素值大于该中点元素，则将查询范围缩小为右半部分。以此类推，直到查到需要的值。

  * 二叉查找树

    二叉查找树中，左子树的键值总是小于根的键值，右子树的键值总是大于根的键值，并且每个节点最多只有两颗子树。

  * 平衡二叉树

    满足二叉查找树的定义，另外必须满足任何节点的两个子树的高度差最大为 1。

  * B 树

    B 树可以理解为一个节点可以拥有多于 2 个子节点的多叉查找树。

    B 树中同一键值不会出现多次，要么在叶子节点，要么在非叶子节点上。

    与平衡二叉树相比，B 树利用多个分支（平衡二叉树只有两个分支）节点，减少获取记录时所经历的节点数。

    B 树也是有缺点的，因为每个节点都包含 key 值和 data 值，因此如果 data 比较大时，每一页存储的 key 会比较少，当数据比较多时，同样会有”要经历多层节点才能查询在叶子节点的数据“的问题。

  * B+ 树

    与 B 树的不同点：

    所有叶子节点中包含了全部关键字的信息；

    各叶子节点用指针进行连接；

    非叶子节点上只存储 key 的信息，这样相对 B 树，可以增加每一页中存储 key 的数量；

    B 树是纵向扩展，最终变成一个“瘦高个”，而 B+ 树是横向扩展的，最终会变成一个“矮胖子”。

    它的键一定会出现在叶子节点上，同时也有可能在非叶子节点中重复出现。而 B 树中同一键值不会出现多次。

* B+ 树索引

  在数据库中，B+ 树的高度一般都在 2 ~ 4 层，所以查找某一行数据最多只需要 2 到 4 次 IO。而没索引的情况，需要逐行扫描，明显效率低很多，这也就是为什么添加索引能提高查询速度。

  B+ 树索引并不能找到一个给定键值的具体行，B+ 树索引能找到的只是被查找数据行所在的页。然后数据库通过把页读入到缓冲池中，在内存中通过二分查找法进行查找，得到需要的数据。

  * 聚集索引

    InnoDB 的数据是按照主键顺序存放的，而聚集索引就是按照每张表的主键构造一颗 B+ 树，它的叶子节点存放的是整行数据。

    InnoDB 的主键一定是聚集索引。如果没有定义主键，聚集索引可能是第一个不允许为 null 的唯一索引，也有可能是 row id。

    由于实际的数据页只能按照一颗 B+ 树进行排序，因此每张表只能有一个聚集索引。查询优化器倾向于采用聚集索引，因为聚集索引能够在 B+ 树索引的叶子节点上直接找到数据。

  * 辅助索引

    InnoDB 存储引擎辅助索引的叶子节点并不会放整行数据，而存放的是键值和主键 ID。

    当通过辅助索引来寻找数据时，InnoDB 存储引擎会遍历辅助索引树查找到对应记录的主键，然后通过主键索引来找到对应的行数据。

* 比较常见需要创建索引的场景

  * 数据检索

    建议数据检索时，在条件字段添加索引。

  * 聚合函数

    ```sql
    select max(a) from t;
    ```

  * 排序

    如果对单个字段排序，则可以在这个排序字段上添加索引来优化排序语句；

    如果是多个字段排序，可以在多个排序字段上添加联合索引来优化排序语句；

    如果是先等值查询再排序，可以通过在条件字段和排序字段添加联合索引来优化排序语句。

  * 避免回表

    可以通过添加覆盖索引让 SQL 不需要回表，从而减少树的搜索次数，让查询更快地返回结果。

  * 关联查询

    通过在关联字段添加索引，让 BNL 变成 NLJ 或者 BKA。

* Insert Buffer

  对于非聚集索引的插入，先判断插入的非聚集索引页是否在缓冲池中。如果在，则直接插入，如果不在，则先放入 Insert Buffer 中，然后再以一定频率和情况进行 Insert Buffer 和辅助索引页子节点的 merge 操作。这时通常能将多个插入合并到一个操作中（因为在一个索引页中），就大大提高了非聚集索引的插入性能。

  增加 Insert Buffer 有两个好处：减少磁盘的离散读取，将多次插入合并为一次操作。

  使用 Insert Buffer 得满足两个条件：索引是辅助索引，索引不是唯一。

* Change Buffer

  可以算是对 Insert Buffer 的升级，InnoDB 存储引擎可以对 insert、delete、update 都进行缓存。

  innodb_change_buffering：确定哪些场景使用 Change Buffer，它的值包含 none、inserts、deletes、changes、purges、all（默认）。

  innodb_change_buffer_max_size：控制 Change Buffer 最大使用内存占总 buffer pool 的百分比。默认 25，表示最多可以使用 buffer pool 的 25%，最大值 50。

  跟 Insert Buffer 一样，Change Buffer 也得满足这两个条件：索引是辅助索引，索引不是唯一。

  * 为什么唯一索引的更新不使用 Change Buffer ?

    唯一索引必须要将数据页读入内存才能判断是否违反唯一性约束。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 Change Buffer 了。

* 普通索引和唯一索引的区别

  有普通索引的字段可以写入重复的值，而有唯一索引的字段不可以写入重复的值。

  如果对数据有修改操作，普通索引可以用 Change Buffer，而唯一索引不行。

  在数据修改时，唯一索引在 RR 隔离级别下，更容易出现死锁。

  在查询过程中，对于普通索引，查找到满足条件的第一个记录，还需要查找下一个记录，直到不满足条件。对于唯一索引，查找到第一个记录返回结果就结束了。但是 InnoDB 是按页从磁盘读取的，所以很大可能根据该普通索引查询的数据都在一个数据页里，因此如果通过普通索引查找到第一条满足条件所在的数据页，再查找后面的记录很大概率都在之前的数据页里，也就是多了几次内存扫描，实际这种消耗可以忽略不计。

* 普通索引和唯一索引如何选择

  如果业务要求某个字段唯一，则添加唯一索引。

  普通索引可以使用 Change Buffer，并且出现死锁的概率比唯一索引低。

* 