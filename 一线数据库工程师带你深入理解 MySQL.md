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

  * 为什么唯一索引的更新不使用 Change Buffer？

    唯一索引必须要将数据页读入内存才能判断是否违反唯一性约束。如果都已经读入到内存了，那直接更新内存会更快，就没必要使用 Change Buffer 了。

* 普通索引和唯一索引的区别

  有普通索引的字段可以写入重复的值，而有唯一索引的字段不可以写入重复的值。

  如果对数据有修改操作，普通索引可以用 Change Buffer，而唯一索引不行。

  在数据修改时，唯一索引在 RR 隔离级别下，更容易出现死锁。

  在查询过程中，对于普通索引，查找到满足条件的第一个记录，还需要查找下一个记录，直到不满足条件。对于唯一索引，查找到第一个记录返回结果就结束了。但是 InnoDB 是按页从磁盘读取的，所以很大可能根据该普通索引查询的数据都在一个数据页里，因此如果通过普通索引查找到第一条满足条件所在的数据页，再查找后面的记录很大概率都在之前的数据页里，也就是多了几次内存扫描，实际这种消耗可以忽略不计。

* 普通索引和唯一索引如何选择

  如果业务要求某个字段唯一，则添加唯一索引。

  普通索引可以使用 Change Buffer，并且出现死锁的概率比唯一索引低。

* 认识联合索引

  idx_a_b(a, b)：对于 a，b 两个字段都做为条件时，查询是可以走索引的，对于单独 a 字段查询也是可以走索引的，但是对于 b 字段单独查询就走不了索引了。

  建议：where 条件中，经常同时出现的列放在联合索引中；把选择性最大的列放在联合索引的最左边。

* 联合索引使用分析

  * 可以完整用到联合索引的情况

    ```sql
    idx_a_b_c(a, b, c) > select * from t where a = 1 and b = 1 and c = 1;
    ```

    联合索引各字段都做为条件时，各字段的位置不会影响联合索引的使用。

    ```sql
    idx_a_b_c(a, b, c) > select * from t where c = 1 and b = 1 and a = 1;
    ```

    联合索引前面的字段使用了范围查询，后面的字段做为条件时仍然可以使用完整的联合索引。

    ```sql
    idx_a_b_c(a, b, c) > select * from t where a = 1 and b in (1, 2) and c = 1;
    ```

    联合索引前面的字段做为条件时，对后面的字段做排序可以使用完整的联合索引。

    ```sql
    idx_a_b_c(a, b, c) > select * from t where a = 1 and b = 1 order by c;
    idx_a_b_c(a, b, c) > select * from t where a = 1 order by b, c;
    ```

    对联合索引的字段同时做排序时（但是排序的三个字段顺序要跟联合索引中三个字段的顺序一致），可以完整用到联合索引。

    ```sql
    idx_a_b_c(a, b, c) > select a, b, c from t order by a, b, c;
    ```

  * 只能使用部分联合索引的情况

    ```sql
    idx_a_b_c(a, b, c) > select * from t where a = 1 and b = 1;
    idx_a_b_c(a, b, c) > select * from t where a = 1 and c = 1;
    ```

    联合索引 idx_a_b_c(a, b, c) 相当于 (a) 、(a, b) 、(a, b, c) 三种索引，称为联合索引的最左原则。

    ```sql
    idx_a_b_c(a, b, c) > select * from t where a = 1 and b in (1, 2) order by c;
    ```

    当联合索引前面的字段使用了范围查询，对后面的字段排序使用不了索引排序。

  * 可以用到覆盖索引的情况

    ```sql
    idx_a_b_c(a, b, c) > select b, c from t where a = 1;
    idx_a_b_c(a, b, c) > select c from t where a = 1 and b = 1;
    idx_a_b_c(a, b, c) > select id from t where a = 1 and b = 1 and c = 1;
    ```

    从辅助索引中就可以查询到结果，不需要回表查询聚集索引中的记录，因此可以减少 SQL 执行过程中的 IO 次数。

  * 不能使用联合索引的情况

    ```sql
    idx_a_b_c(a, b, c) > select * from t where b = 1;
    idx_a_b_c(a, b, c) > select * from t order by b;
    ```

* show index 的使用

  ```sql
  show index from t;
  ```

  [SHOW INDEX Statement](https://dev.mysql.com/doc/refman/5.7/en/show-index.html)

* Cardinality 取值

  Cardinality 表示该索引不重复记录数量的预估值。如果该值比较小，那就应该考虑是否还有必要创建这个索引。

  Cardinality 统计信息的更新发生在两个操作中，INSERT 和 UPDATE。当然也不是每次 INSERT 或 UPDATE 就更新的，其更新时机为，表中 1/16 的数据已经发生过变化；表中数据发生变化次数超过 2000000000。

  InnoDB 表取出 B+ 树索引中叶子节点的数量，记为 a，随机取出 B+ 树索引中的 8 个（innodb_stats_transient_sample_pages）叶子节点，统计每个页中不同记录的个数（假设为 b1，b2，b3，…，b8），则 Cardinality 的预估值为 (b1 + b2 + b3 + … b8) * a / 8，所以 Cardinality 的值是对 8 个叶子节点进行采样获取的，显然这个值并不准确，只供参考。

  | 参数                                | 解释                                                         |
  | ----------------------------------- | ------------------------------------------------------------ |
  | innodb_stats_transient_sample_pages | 设置统计 Cardinality 值时每次采样页的数量，默认值为 8。      |
  | innodb_stats_method                 | 用来判断如何对待索引中出现的 NULL 值记录，默认为 nulls_equal，表示将 NULL 值记录视为相等的记录。另外还有 nulls_unequal 和 nulls_ignored。nulls_unequal 表示将 NULL 视为不同的记录，nulls_ignored 表示忽略 NULL 值记录。 |
  | innodb_stats_persistent             | 是否将 Cardinality 持久化到磁盘。                            |
  | innodb_stats_on_metadata            | 当通过命令 show table status、show index 及访问 information_schema 库下的 tables 表和 statistics 表时，是否需要重新计算索引的 Cardinality。 |

* 统计信息不准确导致选错索引

  在 MySQL 中，优化器控制着索引的选择。一般情况下，优化器会考虑扫描行数、是否使用临时表、是否排序等因素，然后选择一个最优方案去执行 SQL 语句。

  而 MySQL 中扫描行数并不会每次执行语句都去计算一次，因为每次都去计算，数据库压力太大了。实际情况是通过统计信息来预估扫描行数，这个统计信息就可以看成 show index 中的 Cardinality。

  ```sql
  analyze table t;
  ```

* 单次选取的数据量过大导致选错索引

  单次选取的数据量过大也有可能导致优化器选错索引，这种时候，可以尝试使用 force index 让 sql 强制走某个索引。

### MySQL 锁

MySQL 中，锁就是协调多个用户或者客户端并发访问某一资源的机制，保证数据并发访问时的一致性和有效性。

根据加锁的范围，MySQL 中的锁可分为三类，全局锁，表级锁，行锁。

* 全局锁

  MySQL 全局锁会关闭所有打开的表，并使用全局读锁锁定所有表。

  ```sql
  FLUSH TABLES WITH READ LOCK;
  UNLOCK TABLES;
  ```

  当执行 FTWRL 后，所有的表都变成只读状态，数据更新或者字段更新将会被阻塞。

  全局锁一般用在整个库（包含非事务引擎表）做备份（mysqldump 或者 xtrabackup）时。也就是说，在整个备份过程中，整个库都是只读的。其实这样风险挺大的，如果是在主库备份，会导致业务不能修改数据，而如果是在从库备份，就会导致主从延迟。

  好在 mysqldump 包含一个参数 --single-transaction，可以在一个事务中创建一致性快照，然后进行所有表的备份。因此在增加这个参数的情况下，备份期间可以进行数据修改，但是需要所有表都是事务引擎表。所以这也是建议使用 InnoDB 存储引擎的原因之一。

* 表级锁

  * 表锁

    事务需要更新某张大表的大部分或全部数据。如果使用默认的行锁，不仅事务执行效率低，而且可能造成其它事务长时间锁等待和锁冲突，这种情况下可以考虑使用表锁来提高事务执行速度；

    事务涉及多个表，比较复杂，可能会引起死锁，导致大量事务回滚，可以考虑表锁避免死锁。

    对表执行 lock tables xxx read（表读锁）时，本线程和其它线程可以读，本线程写会报错，其它线程写会等待。

    对表执行 lock tables xxx write（表写锁）时，本线程可以读写，其它线程读写都会阻塞。

  * 元数据锁

    MDL 锁的出现解决了同一张表上事务和 DDL 并行执行时可能导致数据不一致的问题。

    对于开发来说，在工作中应该尽量避免慢查询，尽量保证事务及时提交，避免大事务等；对于 DBA 来说，也应该尽量避免在业务高峰执行 DDL 操作。

* InnoDB 替换 MyISAM

  InnoDB 支持事务，InnoDB 支持行锁

* 两阶段锁协议

  锁操作分为两个阶段，加锁阶段和解锁阶段，并且保证加锁阶段和解锁阶段不相交。

* InnoDB 行锁模式

  对于普通 select 语句，InnoDB 不会加任何锁，事务可以通过以下语句显式给记录集加共享锁或排他锁：

  共享锁（S）：A shared (S) lock permits the transaction that holds the lock to read a row. 允许一个事务去读一行，阻止其它事务获得相同数据集的排他锁，select * from xxx where ... lock in share mode;

  排他锁（X）：An exclusive (X) lock permits the transaction that holds the lock to update or delete a row. 允许获得排他锁的事务更新数据，阻止其它事务取得相同数据集的共享读锁和排他写锁，select * from xxx where ... for update;

* InnoDB 行锁算法

  Record Lock：单个记录上的索引加锁。

  Gap Lock：间隙锁，对索引项之间的间隙加锁。

  Next-Key Lock：Record Lock + Gap Lock。

  InnoDB 行锁实现特点意味着，如果不通过索引条件检索数据，那么 InnoDB 将对表中所有记录加锁，实际效果跟表锁一样。

* 事务隔离级别

  Read Uncommitted（读未提交），Read Committed（读已提交），Repeatable Read（可重复读），Serializable（串行）

* RC 隔离级别下的行锁

  for update：我们常使用的查询语句，比如 select * from t where a = 1 属于快照读，是不会看到别的事务插入的数据的。而在查询语句后面加了 for update 显式给记录集加了排他锁，也就让查询变成了当前读。插入、更新、删除操作，都属于当前读。其实也就可以理解 select … for update 是为了让普通查询获得插入、更新、删除操作时所获得的锁。

  * 通过非索引字段查询

    ![RC + 条件字段无索引](https://github.com/songor/mysql-learned/blob/master/picture/RC%20%2B%20%E6%9D%A1%E4%BB%B6%E5%AD%97%E6%AE%B5%E6%97%A0%E7%B4%A2%E5%BC%95.jpg)

    由于字段没有索引，因此只能走聚簇索引，进行全表扫描，聚簇索引上的所有记录都被加上了 X 锁。在 MySQL 中，如果一个条件无法通过索引快速过滤，那么存储引擎层面就会将所有记录加锁后返回，因此也就把所有记录都锁上了。

    没有索引的情况下，InnoDB 的当前读会对所有记录都加锁。所以在工作中应该特别注意 InnoDB 这一特性，否则可能会产生大量的锁冲突。

  * 通过唯一索引查询

    ![RC + 条件字段有唯一索引](https://github.com/songor/mysql-learned/blob/master/picture/RC%20%2B%20%E6%9D%A1%E4%BB%B6%E5%AD%97%E6%AE%B5%E6%9C%89%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95.jpg)

    如果查询的条件是唯一索引，那么 SQL 需要在满足条件的唯一索引上加 X 锁，并且会在对应的聚簇索引上加 X 锁。

  * 通过非唯一索引查询

    ![RC + 条件字段有非唯一索引](https://github.com/songor/mysql-learned/blob/master/picture/RC%20%2B%20%E6%9D%A1%E4%BB%B6%E5%AD%97%E6%AE%B5%E6%9C%89%E9%9D%9E%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95.jpg)

    如果查询的条件是非唯一索引，那么 SQL 需要在满足条件的非唯一索引上都加上锁，并且会在它们对应的聚簇索引上加锁。

* 产生幻读的原因

  行锁只能锁住当前行，但是新插入的记录，是在被锁住记录之间的间隙。因此，为了解决幻读问题，InnoDB 在 RR 隔离级别下配置了间隙锁（Gap Lock）。

* RR 隔离级别下的行锁

  * 通过非唯一索引查询

    ![RR + 条件字段有非唯一索引](https://github.com/songor/mysql-learned/blob/master/picture/RR%20%2B%20%E6%9D%A1%E4%BB%B6%E5%AD%97%E6%AE%B5%E6%9C%89%E9%9D%9E%E5%94%AF%E4%B8%80%E7%B4%A2%E5%BC%95.jpg)

    RR 隔离级别多了 GAP 锁

  * 通过非索引字段查询

    ![RR + 条件字段无索引](https://github.com/songor/mysql-learned/blob/master/picture/RR%20%2B%20%E6%9D%A1%E4%BB%B6%E5%AD%97%E6%AE%B5%E6%97%A0%E7%B4%A2%E5%BC%95.jpg)

    RR 隔离级别下，非索引字段做条件的当前读不但会把每条记录都加上 X 锁，还会把每个 GAP 加上 GAP 锁。

  * 通过唯一索引查询

    GAP 锁的目的是，为了防止同一事务两次当前读，出现幻读的情况。如果能确保索引字段唯一，那其实一个等值查询，最多就返回一条记录，而且相同索引记录的值，一定不会再新增，因此不会出现 GAP 锁。

    以唯一索引为条件的当前读不会有 GAP 锁，所以 RR 隔离级别下的唯一索引当前读加锁情况与 RC 隔离级别下的唯一索引当前读加锁情况一致。

* 认识死锁

  InnoDB 中解决死锁问题有两种方式：

  检测到死锁的循环依赖，立即返回一个错误（Deadlock found when trying to get lock; try restarting transaction）。将参数 innodb_deadlock_detect 设置为 on 表示开启这个逻辑；

  等查询的时间达到锁等待超时的设定后放弃锁请求。这个超时时间由 innodb_lock_wait_timeout 来控制（默认是 50 秒）。

* 为什么会产生死锁

  * 同一张表中

    不同线程并发访问同一张表的多行数据，未按顺序访问导致死锁。

    不同程序并发访问同一张表时，如果事先确保每个线程按固定顺序来处理记录，可以降低死锁的概率。

  * 不同表之间

    不同线程并发访问多个表时，未按顺序访问导致死锁。

    不同程序并发访问多个表时，应尽量约定以相同的顺序来访问表，可大大降低并发操作不同表时死锁发生的概率。

  * 事务隔离级别

    RR 隔离级别下，由于间隙锁导致死锁。

    可以考虑将隔离级别改成 RC，降低死锁的概率。

* 如何降低死锁概率

  更新 SQL 的 where 条件尽量用索引；

  基于 primary 或 unique key 更新数据；

  减少范围更新，尤其非主键、非唯一索引上的范围更新；

  加锁顺序一致，尽可能一次性锁定所有需要行；

  将 RR 隔离级别调整为 RC 隔离级别。

* 分析死锁的方法

  查看最后一个死锁的信息：

  ```sql
  show engine innodb status;
  ```

  另外设置 innodb_print_all_deadlocks = on 可以在 err log 中记录全部死锁信息。

### 事务

* 什么是事务

  事务就是一组原子性的 SQL 查询，或者说一个独立的工作单元。如果数据库引擎能够成功地对数据库应用该组查询的全部语句，那么就执行该组查询。如果其中有任何一条语句因为崩溃或其他原因无法执行，那么所有的语句都不会执行。也就是说，事务内的语句，要么全部执行成功，要么全部执行失败。

  一个良好的事务处理系统，必须具备 ACID 特性。

  InnoDB 采用 redo log 机制来保证事务更新的一致性和持久性。

* Redo log

  Redo log 称为重做日志，用于记录事务操作变化，记录的是数据被修改之后的值。

  Redo log 由两部分组成：内存中的重做日志缓冲（redo log buffer），重做日志文件（redo log file）。

  每次数据更新会先更新 redo log buffer，然后根据 innodb_flush_log_at_trx_commit 来控制 redo log buffer 更新到 redo log file 的时机。

  innodb_flush_log_at_trx_commit 有三个值可选：

  0，每秒触发一次 redo log buffer 写磁盘操作，并调用操作系统 fsync 刷新 IO 缓存；

  1，事务提交时，InnoDB 立即将缓存中的 redo 日志写到日志文件中，并调用操作系统 fsync 刷新 IO 缓存；

  2，事务提交时，InnoDB 立即将缓存中的 redo 日志写到日志文件中，但不是马上调用  fsync 刷新 IO 缓存，而是每秒只做一次磁盘 IO 缓存刷新操作。

  innodb_flush_log_at_trx_commit 参数的默认值是 1，也就是每个事务提交的时候都会从 log buffer 写更新记录到日志文件，而且会刷新磁盘缓存，这完全满足事务持久化的要求，是最安全的，但是这样会有比较大的性能损失。

  将参数设置为 0 时，如果数据库崩溃，最后 1 秒钟的 redo log 可能会由于未及时写入磁盘文件而丢失，这种方式尽管效率最高，但是最不安全。

  将参数设置为 2 时，如果数据库崩溃，由于已经执行了重做日志写入磁盘的操作，只是没有做磁盘 IO 刷新操作，因此，只要不发生操作系统奔溃，数据就不会丢失，这种方式是对性能和安全的一种折中处理。

* Binlog

  二进制日志（binlog）记录了所有的 DDL（数据定义语句）和 DML（数据操纵语句），但是不包括 select 和 show 这类操作。

  Binlog 有以下几个作用：

  恢复，数据恢复时可以使用二进制日志；

  复制，通过传输二进制日志到从库，然后进行恢复，以实现主从同步；

  审计，可以通过二进制日志进行审计数据的变更操作。

  可以通过参数 sync_binlog 来控制累积多少个事务后才将二进制日志 fsync 到磁盘：

  0，表示每次提交事务都只写入磁盘，不 fsync；

  1，表示每次提交事务都会执行 fsync；

  N，表示每次提交事务都只写入磁盘，累积 N 个事务后才 fsync。

* 怎样确保数据库突然断电不丢数据？

  只要 innodb_flush_log_at_trx_commit 和 sync_binlog 都为 1（通常称为”双一“），就能确保 MySQL 机器断电重启后数据不丢失。

* 什么是 MVCC

  MVCC， 即多版本并发控制。MVCC 的实现，是通过保存数据在某个时间点的快照来实现的，也就是说，不管需要执行多长时间，每个事务看到的数据都是一致的。根据事务开始的时间不同，每个事务对同一张表，同一时刻看到的数据可能是不一样的。

  MVCC 只在 RC 和 RR 两个隔离级别下工作。

* Undo log

  Undo log 是逻辑日志，将数据库逻辑地恢复到原来的样子，所有修改都被逻辑地取消了。也就是如果是 insert 操作，其对应的回滚操作就是 delete；如果是 delete，则对应的回滚操作是 insert；如果是 update，则对应的回滚操作是一个反向的 update 操作。

* MVCC 的实现原理

  InnoDB 每一行数据都有一个隐藏的回滚指针，用于指向该行修改前的最后一个历史版本（存放在 UNDO LOG 中）。

  当用户读取一行记录时，若该记录已经被其它事务占用，当前事务可以通过 undo log 读取之前的行版本信息，以此实现非锁定读取。

* MVCC 的优势

  MVCC 最大的好处是读不加锁，读写不冲突，极大地增加了 MySQL 的并发性。

  通过 MVCC 保证了事务 ACID 中的隔离性特性。

* 不好的事务习惯

  * 在循环中提交

    在大多数情况下，MySQL 都是开启自动提交的，如果遇到循环执行 SQL，则相当于每个循环中都会进行一次提交，实际这算一个不好的事务习惯了。

    每一次提交都要写一次重做日志。

    如果循环次数不是太多，建议在循环前开启一个事务，循环结束后统一提交。

  * 不关注同一个事务里语句顺序

    根据两阶段锁，整个事务里面涉及的锁，需要等到事务提交时才会释放。因此我们在同一个事务中，可以把没锁或者锁范围小的语句放在事务前面执行，而锁定范围大的语句放在后面执行。

  * 不关注不同事务访问资源的顺序

    如果不关注并发访问的不同事务中访问资源的顺序，就会增大出现死锁的概率。

  * 不关注事务隔离级别

    不同事务隔离级别加锁的情况是不同的，如果完全不关注自己业务使用的 MySQL 是什么隔离级别，可能会降低程序的并发能力或者导致死锁。

  * 在事务中混合使用存储引擎

    在事务中混合使用事务型（如 InnoDB）和非事务型（如 MyISAM）表，如果是正常提交，到没什么问题。但是，如果该事务回滚了，事务型的表可以正常回滚，而非事务型的表的变更就无法回滚了。这种情况就会导致数据不正常，并且事务最终的结果也难以确定。

    开启 GTID 的情况，可以避免同一个事务中混合使用存储引擎的情况。

* 认识分布式事务

  分布式事务是指一个大的事务由很多小操作组成，小操作分布在不同的服务器上或者不同的应用程序上。分布式事务需要保证这些小操作要么全部成功，要么全部失败。

  分布式事务使用两阶段提交协议：

  第一阶段，所有分支事务都开始准备，告诉事务管理器自己已经准备好了；

  第二阶段，确定是 rollback 还是 commit，如果有一个节点不能提交，则所有节点都要回滚。

  与本地事务不同点在于，分布式事务需要多一次 prepare 操作，等收到所有节点的确定信息后，再进行 commit 或者 rollback。

  MySQL 中分布式事务按实现方式可以分为两种，MySQL 自带的分布式事务和结合中间件实现分布式事务。

* MySQL 自带的分布式事务

* 结合中间件实现分布式事务

  ![MQ + 分布式事务](https://github.com/songor/mysql-learned/blob/master/picture/MQ%20%2B%20%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1.jpg)

  订单业务程序处理完增加订单的操作后，将减库存操作发送到消息队列中间件中（如 RocketMQ），订单业务程序完成提交。然后库存业务程序检查到消息队列有减对应商品库存的信息，就开始执行减库存操作。库存业务执行完减库存操作，再发送一条消息给消息队列中间件，内容为已经减掉库存。

### MySQL 其他经验

* SQL 注入

  SQL 注入产生的主要原因是程序对用户输入的数据没有进行严格的过滤，导致非法数据库查询语句的执行。

  使用 prepared statements（预编译）来预防 SQL 注入。

* 主键是否需要设置为自增？

  * 关于自增主键

    ```sql
    show create table t;
    ```

    表结构中包含 AUTO_INCREMENT，在下一次执行未指定 id 字段的 insert 语句时，主键 id 会自动获取到这个值。

  * 主键与聚集索引的关系

    在 InnoDB 中，聚集索引不一定是主键，但是主键一定是聚集索引，原因是如果没有定义主键，聚集索引可能是第一个不允许为 null 的唯一索引，如果也没有这样的唯一索引，InnoDB 会选择内置 6 字节长的 ROWID 作为隐含的聚集索引；

    InnoDB 的数据是按照主键顺序存放的，而聚集索引就是按照每张表的主键构造一颗 B+ 树，它的叶子节点存放的是整行数据；

    每张 InnoDB 表都有一个聚集索引，但是不一定有主键。

  聚集索引是按照每张表的主键构造一颗 B+ 树的，而 B+ 树中，所有记录节点都是按键值的大小顺序存放在同一层叶子节点上。如果每次插入的数据都是在聚集索引树的后面，聚集索引不需要分裂就可以存入数据。但是如果插入的数据值在聚集索引树的中间部分，由于要保证插入后叶子节点中的记录依然有序，就可能需要分裂聚集索引树来保证键值的有序性。

  （叶子节点数据页已经满的情况下，如果写入的值是聚集索引树的中间部分，则会进行一次页分裂，以保证叶子节点上的记录有序和 B+ 树的平衡，并且分裂后，有些数据页没被用满，导致页空间浪费）

  ![聚集索引树分裂](https://github.com/songor/mysql-learned/blob/master/picture/%E8%81%9A%E9%9B%86%E7%B4%A2%E5%BC%95%E6%A0%91%E5%88%86%E8%A3%82.jpg)

  因此如果业务输入的主键都是随机数字，那么写入数据时很可能会导致数据页频繁分裂，从而影响写入效率。而如果设置主键是自增，那么每一次都是在聚集索引的最后增加，当一页写满，就会自动开辟一个新页，不会有聚集索引树分裂这一步，效率会比随机主键高很多，这也是很多建表规范要求主键自增的原因。

  除了要求主键自增外，最好主键也要无业务意义，原因是防止主键更新而导致页分裂的情况。

  当然也不是所有的情况主键都需要设置为自增，比如可以用程序写入增长的主键值，保证了新写入数据的主键值比之前大，也可以避免聚集索引树频繁分裂。

* MySQL 是否需要开启查询缓存？

  它会存储 select 语句的文本以及发送到客户端的结果。如果下一次收到一个相同的查询，就会从查询缓存中获得查询结果。

  * 认识 QC

    ```sql
    show variables like "%query_cache%";
    ```

    ```sql
    show global status like "qcache%";
    ```

  * QC 的优劣

    优势：

    提高查询速度，使用查询缓存在单行数据的表中搜索要比不使用查询缓存快 238%。

    劣势：

    执行的 SQL 都很简单（比如从只有一行的表中查询数据），但每次查询都不一样的话，打开 QC 后，额外的开销为 13% 左右；

    如果表数据发生了修改，使用该表的所有缓存查询都将失效，并且从缓存中删除；

    QC 要求前后两次请求的 SQL 完全一样，不同数据库、不同协议版本或不同默认字符集的查询，都会被认为是不同的查询，甚至包括大小写；

    每次更新 QC 的内存块都需要进行锁定；

    可能会导致 SQL 查询时间不稳定。

  * 是否需要开启 QC

    如果线上环境中 99% 以上都是只读，很少更新，可以考虑全局开启 QC，也就是设置 query_cache_type 为 1。

    很多时候，我们希望缓存的是几张更新频率很低的表，其它表不考虑使用查询缓存，就可以考虑将 query_cache_type 设置成 2 或者 DEMAND，这样就只缓存下面这类 SQL：

    ```sql
    select sql_cache ...
    ```

  * 开启和关闭 QC

    在配置文件 my.conf 中设置 `query_cache_type`，`query_cache_size`。

  * 开启 QC 的注意事项

    如果要开启 QC，建议不要设置过大，通常几十兆就好。如果设置过大，会增加维护缓存所需要的开销。

    要注意一些即使开启 QC 也不能使用 QC 的场景：

    分区表不支持，如果涉及分区表的查询，将自动禁用查询缓存；

    子查询或者外层查询；

    存储过程、触发器中使用的 SQL；

    读取系统库；

    `select ... for update`，`select ... lock in share mode`；

    用到临时表；

    产生了 warning 的查询；

    显示增加了 SQL_NO_CACHE 关键字；

    如果没有全部库、表的 select 权限，则也不会使用 QC；

    使用了一些函数，如 now()，user()，password() 等。

* 使用读写分离需要注意哪些？

  通常我们说的 MySQL 读写分离是指，对于修改操作在主库上执行，而对于查询操作在从库上执行，主要目的是分担主库的压力。

  * 主从复制的原理

    * MySQL 异步复制

      在主库开启 binlog 的情况下，如果主库有增删改的语句，会记录到 binlog 中，主库通过 IO 线程把 binlog 里面的内容传给从库的中继日志（relay log）中，主库给客户端返回 commit 成功（这里不会管从库是否已经收到了事务的 binlog），从库的 SQL 线程负责读取它的 relay log 里的信息并应用到从库数据库中。

      ![MySQL 异步复制](https://github.com/songor/mysql-learned/blob/master/picture/MySQL%20%E5%BC%82%E6%AD%A5%E5%A4%8D%E5%88%B6.jpg)

      在主库上并行运行的更新 SQL，由于从库只有单个 SQL 线程去消化 relay log，因此更新的 SQL 在从库只能串行执行。这也是很多情况下，会出现主从延迟的原因。

      从 5.6 开始，MySQL 支持了每个库可以配置单独的 SQL 线程来消化 relay log，在 5.7 又增加了基于组提交的并行复制，大大改善了主从延迟的问题。

    * MySQL 半同步复制

      在主库开启 binlog 的情况下，如果主库有增删改的语句，会记录到 binlog 中，主库通过 IO 线程把 binlog 里面的内容传给从库的中继日志（relay log）中，从库收到 binlog 后，发送给主库一个 ACK 表示收到了，主库收到这个 ACK 以后，才能给客户端返回 commit 成功，从库的 SQL 线程负责读取它的 relay log 里的信息并应用到从库数据库中。

      ![MySQL 半同步复制](https://github.com/songor/mysql-learned/blob/master/picture/MySQL%20%E5%8D%8A%E5%90%8C%E6%AD%A5%E5%A4%8D%E5%88%B6.jpg)

  * 常见的读写分离方式

    * 通过程序

      开发通过配置程序来决定修改操作走主库，查询操作走从库。这种方式直连数据库，优点是性能会好点，缺点是配置麻烦。

      但是需要注意的是，从库需要设置为 read_only，防止配置错误在从库写入了数据。

    * 通过中间件

      MyCAT

  * 什么情况下会出现主从延迟

    大表 DDL、大事务、主库 DML 并发大、从库配置差、表上无主键等。

  * 读写分离怎样应对主从延迟

    * 判断主从是否延迟

      判断 Seconds_Behind_Master 是否等于 0：如果 Seconds_Behind_Master = 0，则查询从库，如果大于 0，则查询主库。

      对比位点：如果 Master_Log_File 跟 Relay_Master_Log_File 相等，并且 Read_Master_Log_Pos 跟 Exec_Master_Log_Pos 相等，则可以把读请求放到从库，否则读请求放到主库。

      GTID：对比 Retrieved_Gtid_Set 和 Executed_Gtid_Set 是否相等，相等则把读请求放到从库，有差异则把读请求放到主库。

    * 采用半同步复制

      半同步复制保证了所有给客户端发送过确认提交的事务，从库都已经收到这个日志了，因此出现延迟的概率会小很多。在实际生产应用中，建议结合位点或 GTID 判断。

    * 等待同步完成

      依然采用判断是否有延迟的方法，只是应对方式不一样，如果存在延迟，则将情况反馈给程序，在前端页面提醒用户数据未完全同步，如果没有延迟，则查询从库。

* 哪些情况需要考虑分库分表？

  MySQL 分库分表是指，把 MySQL 数据库物理地拆分到多个实例或者机器上去，从而降低单台 MySQL 实例的负载。

  * MySQL 分库分表拆分方法

    垂直拆分：

    有多个业务，每个业务单独分到一个实例里面；

    一个实例中有多个库，把这些库分别放到单独的实例中；

    一个库中存在过多的表，把这些表拆分到多个库中；

    把字段过多的表拆分成多个表，每张表包含一部分字段。

    水平拆分：

    如果通过垂直拆分，表数据量仍然很大（单表行数超过 500 万行或者单表容量超过 2GB），那就可以考虑使用水平拆分了。

    所谓水平拆分，就是把同一张表分为多张表结构相同的表，每张表里存储一部分数据。而拆分的算法也比较多，常见的就是取模、范围和全局表等。

  * 需要考虑分库分表的场景

    * 数据量过大，影响了运维操作

      备份，如果单张表或者单个实例数据量太大，那备份可能需要占用大量的 IO 和磁盘空间，并且持续时间还会比较久。

      大表 DDL：大表执行 DDL 不但会产生 MDL 写锁，并且还会导致主从延迟（主库执行完 DDL 后，才会写入到 binlog 里，然后传输到从库执行，而又因为从库 SQL 线程是单线程的，因此，需要等到这条 DDL 在从库执行完成，其他事务才能继续执行，而从库执行 DDL 这段时间，主从都是延迟的）。

    * 把修改频繁的字段拆分出来

    * 把大字段拆分出去（商品表中商品详情字段）

    * 表数据增长比较快（订单表）

    * 不同库表之间性能相互影响了

  * 分库分表的实现

    通过程序，通过数据库中间件（MyCAT）。

* 如何安全高效地删除大量无用数据？

  * 共享表空间和独立表空间

    InnoDB 数据是按照表空间进行存放的，其表空间分为共享表空间和独立表空间。

    共享表空间，文件名为 ibdata1，可以通过参数 `innodb_data_file_path` 进行设置。

    ```my.cnf
    [mysqld]
    innodb_data_file_path = ibdata1:1G;ibdata2:1G:autoextend
    ```

    用两个文件 ibdata1 和 ibdata2 组成表空间，文件 ibdata1 的大小为 1G，文件 ibdata2 的大小为 1G，autoextend 表示用完 1G 可以自动增长。

    独立表空间，每个 InnoDB 表数据存储在一个以 .idb 为后缀的文件中，由参数 `innodb_file_per_table` 控制，设置为 on 表示使用独立表空间，设置为 off 表示使用共享表空间。

    一般情况下建议设置为独立表空间，原因是，如果某张表被 drop 掉，会直接删除该表对应的文件，如果放在共享表空间中，即使执行了 drop table 操作，空间还是不能被回收。

  * 几种数据删除的方式

    * 删除表

      如果是某张表的数据和表结构都不需要使用了，那么可以考虑 drop 掉。

      ```sql
      alter table t rename t_bak_20200414;
      -- 等待半个月，观察是否有程序因为找不到表 t 而报错
      drop table t_bak_20200414;
      ```

    * 清空表

      如果是某张表的历史数据不需要使用了，要做一次清空，则可以考虑使用 truncate。

      ```sql
      create table t_bak_20200414 like t;
      insert into t_bak_20200414 select * from t;
      truncate table t;
      -- 观察半个月
      drop table t_bak_20200414;
      ```

      需要清空表而使用 delete from xxx，导致主从延迟和磁盘 IO 跑满。原因是 binlog 为 row 模式的情况下，执行全表 delete 会生成每一行对应的删除操作，因此可能导致单个删除事务非常大。而 truncate 可以理解为 drop + create，在 binlog 为 row 模式的情况下，只会产生一行 truncate 操作。所以，建议清空表时使用 truncate 而不使用 delete。

    * 非分区表删除部分记录

      在没有配置分区表的情况下，就只能用 delete 了。

      在 delete 很多数据后，实际表文件大小没变化，原因是，如果通过 delete 删除某条记录，InnoDB 引擎会把这条记录标记为删除，但是磁盘文件的大小并不会缩小。如果之后要在这中间插入一条数据，则可以复用这个位置，如果一直没有数据插入，就会形成一个“空洞”。因此 delete 命令是不能回收空间的，这也是 delete 后表文件大小没变化的原因。

      ```sql
      create table t_bak_20200414 like t;
      insert into t_bak_20200414 select * from t;
      -- 确保 date 字段有索引，如果没有索引，则需要添加索引,目的是避免执行删除命令时全表扫描
      delete from t where date < '2019-01-01';
      -- 如果要删除的数据比较多，建议写一个循环，每次删除满足条件记录的 1000 条，目的是避免大事务，删完为止
      delete from t where date < '2019-01-01' limit 1000;
      -- 最后重建表，目的是释放表空间，但是会锁表，建议在业务低峰执行
      alter table t engine=InnoDB;
      -- 或者
      optimize table t;
      ```

    * 分区表删除部分分区

      MySQL 分区是指将一张表按照某种规则（比如时间范围或者哈希等），划分为多个区块，各个区块所属的数据文件是相互独立的。

      ```sql
      CREATE TABLE t_log (
      	id INT,
      	log_info VARCHAR (100),
      	date datetime
      ) ENGINE = INNODB PARTITION BY RANGE (YEAR(date))(
      	PARTITION p2016
      	VALUES
      		less THAN (2017),
      		PARTITION p2017
      	VALUES
      		less THAN (2018),
      		PARTITION p2018
      	VALUES
      		less THAN (2019)
      );
      ```

      ```sql
      SELECT
      	TABLE_SCHEMA,
      	TABLE_NAME,
      	PARTITION_NAME,
      	TABLE_ROWS
      FROM
      	information_schema.PARTITIONS
      WHERE
      	table_schema = 'muke'
      AND table_name = 't_log';
      ```

      ```sql
      alter table t_log drop partition p2016;
      ```

      对于要经常删除历史数据的表，建议配置成分区表，以方便后续历史数据删除。

* 使用 MySQL 时，应用层可以这么优化

  * 使用连接池

    连接池可以理解为，创建一些持久连接的“池”，新的请求可以使用这些连接池，减少创建和断开连接的次数。

    当遇到连接池完全占满时，应该将连接请求进行排队，而不是扩展连接池。这样可以避免将压力都转到 MySQL 上而导致 MySQL 连接数过多。

  * 减少对 MySQL 的访问

    避免对同一行数据做重复检索。

  * 增加 Redis 缓存层

    计数器，缓解在 MySQL 中执行 update 的压力；

    K-V 数据缓存，某个字段会被频繁查询，而该字段内容变化的概率又不是很大；

    消息队列，生产者通过 lpush 将消息放在 list 中，消费者通过 rpop 取出该消息。

  * 单表过大及时归档

    在修改表结构时导致长时间主从延迟；备份时间过久；查询速度可能也会变慢。

    可以考虑对历史数据归档，控制单表的数据量。

  * 代码层读写分离

  * 提前规划表的索引

* MySQL 整体优化思路

  * 硬件相关优化

    * CPU 相关

      关闭 CPU 节能，设定为最大性能模式。原因是，考虑到在高并发之前没有任何连接的情况，机器可能会处于节电模式，高并发场景来临时可能导致处理不过来新的请求。

      配置合理的 CPU 核数和选择合适的 CPU 主频。原因是，CPU 核数越多，支持的并发也越高；CPU 主频越高，处理任务的速度越快。

    * 内存相关

      InnoDB 使用 buffer pool 缓存数据、索引等内容，从而加快访问速度。

      在应用数据库实例前，预估活跃的数据大小，合理配置数据库服务器内存的大小。

    * 磁盘相关

      使用 SSD（固态硬盘）或者 Pcle SSD 设备。传统的机械硬盘需要耗费长时间的磁头旋转和定位来查找数据，而 SSD 内部是由闪存组成的，闪存延迟低、功耗低，因此 SSD 比传统机械硬盘更快。

      尽可能选择 RAID 10。

      设置阵列写策略为 Write Back。

  * 系统层面优化

    * 调整 I/O 调度算法

      MySQL 运行的物理机上，I/O 调度算法建议使用 deadline/noop，尽量不使用 CFQ。原因是 CFQ 把 I/O 请求按照进程分别放入进程对应的队列中。CFQ 的公平是针对进程而言，每一个提交 I/O 请求的进程都会有自己的 I/O 队列，以时间算法为前提，轮转调动队列，默认从当前队列中取出 4 个请求处理，然后处理下一个队列的 4 个请求，确保每个进程享有的 I/O 资源是均衡的。因此高并发场景，CFQ 很可能会导致 I/O 的响应缓慢。

    * 文件系统选择

      优先选用 xfs 或 ext4，坚决不用 ext3。

    * 调整内核参数

      `vm.swappiness`，降低使用 swap 的概率，但是尽量不要设置为 0，可能引起 OOM。

      `vm.dirty_ratio`，`vm.dirty_background_ratio`。

  * MySQL 层优化

    * 参数优化

      `innodb_buffer_pool_size`，该参数控制 InnoDB 缓存表和索引数据的内存区域大小，对性能影响非常大，建议设置为机器内存的 50-80%

      `innodb_flush_log_at_trx_commit`

      `sync_binlog`

      `innodb_file_per_table`

      `max_connection`，最大连接数

      `long_query_time`

      `query_cache_type`，`query_cache_size`

    * 设计优化

      使用 InnoDB 存储引擎，不建议使用 MyISAM 存储引擎；

      预估表数据量和访问量，如果数据量或者访问量比较大，则需要提前考虑分库分表；

      指定合适的数据库规范，在设计表、执行 SQL 语句时按照数据库规范来进行。

* 

