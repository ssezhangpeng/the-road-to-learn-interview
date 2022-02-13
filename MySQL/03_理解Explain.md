### 简介

> EXPLAIN作为MySQL的性能分析神器，我们应该对其结果进行充分的了解。

#### 结果输出字段

- id(唯一标识)

  - 该语句的唯一标识。如果explain的结果包括多个id值，则数字越大越先执行；而对于相同id的行，则表示从上往下依次执行。

- select_type(查询类型)

  - SIMPLE：简单查询(未使用UNION或子查询)
  - UNION： 在UNION中的第二个和随后的SELECT被标记为UNION。如果UNION被FROM子句中的子查询包含，那么它的第一个SELECT会被标记为DERIVED。
  - SUBQUERY：子查询中的第一个 SELECT。

- table(表名)

  - 表示当前这一行正在访问哪张表，如果SQL定义了别名，则展示表的别名。

- partitions(匹配的分区)

  - 当前查询匹配记录的分区。对于未分区的表，返回null。

- type(联接类型，***性能从好到坏排序***)

  - system：该表只有一行（相当于系统表），system是const类型的特例。
  - const：针对主键或唯一索引的等值查询扫描, 最多只返回一行数据. const 查询速度非常快, 因为它仅仅读取一次即可。
  - eq_ref：当使用了***索引的全部组成部分***，并且索引是PRIMARY KEY或UNIQUE NOT NULL 才会使用该类型，性能仅次于system及const。
  - ref：当满足索引的最左前缀规则，或者索引不是主键也不是唯一索引时才会发生。如果使用的索引只会匹配到少量的行，性能也是不错的。
  - range：范围扫描，表示检索了指定范围的行，主要用于有限制的***索引扫描***。比较常见的范围扫描是带有BETWEEN子句或WHERE子句里有>、>=、<、<=、IS NULL、<=>、BETWEEN、LIKE、IN()等操作符。
  - index：全索引扫描，和ALL类似，只不过index是全盘扫描了索引的数据。当查询仅使用索引中的一部分列时，可使用此类型。
  - ALL：全表扫描，性能最差。

- possible_keys(可能的索引选择)

  - 展示当前查询可以使用哪些索引，这一列的数据是在优化过程的早期创建的，因此有些索引可能对于后续优化过程是没用的。

- key(实际选择的索引)

  - 表示MySQL实际选择的索引。

- key_len(索引的长度)

  - 索引使用的字节数，参考[计算方式](https://www.cnblogs.com/gomysql/p/4004244.html)

- ref(索引的哪一列被引用)

  - 表示将哪个字段或常量和key列所使用的字段进行比较。

- rows(估计要扫描的行)

  - MySQL估算会扫描的行数，数值越小越好。

- filtered(符合查询条件的数据百分比)

  - 表示符合查询条件的数据百分比，最大100。用rows × filtered可获得和下一张表连接的行数。例如rows = 1000，filtered = 50%，则和下一张表连接的行数是500。

- Extra(附加信息)

  -  Using index：仅使用索引树中的信息从表中检索列信息，而不必进行其他查找以读取实际行。
  - Using temporary：为了解决该查询，MySQL需要创建一个临时表来保存结果。如果查询包含不同列的GROUP BY和 ORDER BY子句，通常会发生这种情况。
  - Using where：如果我们不是读取表的所有数据，或者不是仅仅通过索引就可以获取所有需要的数据，则会出现using where信息。
  - Using index condition：表示先按条件过滤索引，过滤完索引后找到所有符合索引条件的数据行，随后用 WHERE 子句中的其他条件去过滤这些数据行。通过这种方式，除非有必要，否则索引信息将可以延迟“下推”读取整个行的数据。
  - Using filesort：当Query 中包含 ORDER BY 操作，而且无法利用索引完成排序操作的时候，MySQL Query Optimizer 不得不选择相应的排序算法来实现。数据较少时从内存排序，否则从磁盘排序。

  ### 参考资料

  [Explain 详解](https://cloud.tencent.com/developer/article/1663095)