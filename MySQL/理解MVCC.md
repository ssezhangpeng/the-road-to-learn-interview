### 什么是 MVCC
- MVCC(Muti-Version Concurrency Control),中文翻译即为``多版本并发控制``。它指的是在使用``READ COMMITED``和``REPEATABLE READ``这两种隔离级别的事务在执行普通的 SELECT 操作时，访问数据记录的``版本链``的过程。这样可以使``不同事务``之间的读写并发执行，提升系统性能。

### MVCC机制使用的隔离级别
- ``READ UNCOMMITTED``: 由于可以读到未提交事务修改过的记录，所以直接读取记录的最新版本就好了，所以不需要使用MVCC
- ``SERIALIZABLE``: 访问记录必须要加锁，所以不需要使用MVCC
- ``READ COMMITTED``: 需要使用 MVCC
- ``REPEATABLE READ``: 需要使用 MVCC

### 快照读和当前读
- 快照读(```MVCC 都是使用在这种读场景下，所以后面我们只分析这种读场景```)
    - 简单的 SELECT 操作，属于快照读，使用 MVCC 机制，不加锁，比如
        ```sql
        SELECT * FROM table WHERE ？;
        ```
- 当前读
    - 特殊的读操作以及插入/更新/删除操作，需要加锁，比如
        ```sql
        -- 共享锁(S锁)
        SELECT * FROM table WHERE ？lock in share mode;
        -- 排他锁(X锁)
        SELECT * FROM table WHERE ？for update;
        -- 插入/更新/删除操作也都需要先把记录读取
        DELETE FROM table WHERE ?;
        ```
### InnoDB 引擎的隐藏列
>     在InnoDB中，每一行数据除了包括我们设计的字段之外，还包含了一些内部字段，也叫隐藏字段。比如在InnoDB的聚簇索引记录中会包含3个隐藏列row_id、trx_id 和 roll_pointer。
- row_id
    - 数据行id，用于标识一行数据。row_id并不是必要的，如果创建的表中有```主键或者非NULL唯一键```时都不会包含row_id列。
- trx_id
    - 事务id，每次对某数据记录进行修改时，都会把对应的事务id赋值给trx_id列。每次事务操作都会分配一个事务id，它是一个自增id。
- rolll_pointer
    - 当前数据记录的上一个版本的指针。每次对某条数据记录进行改动时，都会把旧版本数据记录按照一定格式写入到回滚日志(```undo log```) 中，而roll_pointer列则保存了该旧版本数据记录在回滚日志中的位置，相当于一个指针。

### 版本链
- 在每次更新记录后，都会将旧值存放到一条undo log 中(就算是该记录的一个旧版本)，随着更新次数多增多，所有的版本都会被 roll_pointer 属性连接成一个链表，这个链表称为```版本链```。

### ReadView(一致性视图)
- 