### undo 日志

> ​    undo 存放在数据库内部的一个特殊段(segment)中，这个段称为 undo 段。undo 段位于**共享表空间**内。需要注意的是，undo 是逻辑日志，因此只是将数据库逻辑地恢复到原来的样子。但是数据结构和页本身在回滚之后可能不大相同。比如INSERT操作时，可能会分配新的 Segment，即表空间会增大，但用户执行 ROLLBACK 时，会将插入的事务进行回滚，但是表空间的大小并不会因此而收缩。

#### 作用

- ROLLBACK(事务回滚)
- MVCC(实现非锁定读)

#### 存储管理

> ​      InnoDB存储引擎对 undo 的管理同样采用段(Segment)的管理方式。InnoDB有一个**rollback segment**，每个回滚段记录了 1024 个 **undo log segment**，而在每个 undo log segment段中进行 **undo 页** 的申请。在 InnoDB1.1版本之前，只有一个 rollback segment，所以支持同时并发的事务限制为 1024，从 1.1 版本开始，最大支持 128个 rollback segment，故其支持的事务限制提高到 128 * 1024。但这些 rollback segment 都存储于共享表空间中(默认**ibdata1**)。从InnoDB1.2版本，支持 rollback segmet 设置单独的独立表空间。

- 当事务提交时，InnoDB 会做两件事情：
  - 将 undo log 放入列表中，以供之后的 purge 操作
  - 判断 undo log 所在的页是否可以重用，若可以分配给下个事务使用
- [⚠️] 事务提交之后并不能马上删除 undo log 及 undo log 所在的页。这是因为可能还有其它的事务需要通过 undo log 来得到行记录之前的版本。事务提交时将 undo log 放入一个链表(**history list**)中,是否可以最终删除有 purge 线程来判断。

#### undo 页重用

> ​    若为每个事务分配一个 单独的 undo 页会非常浪费存储空间，因为在事务提交的时候，可能并不能马上释放页。因此，在 InnoDB 存储引擎的设计中对 undo 页可以进行重用。具体来说，当事务提交时，首先将 undo log 放入链表中，然后判断 undo 页的使用空间是否小于 3/4，若是则表示该 undo 页可以被重用，之后新的 undo log 页记录在当前 undo log 页的后面，由于存放 undo log 的列表是以记录进行组织的，而 undo 页可能存放着不同事务的 undo log，因此 purge 操作需要设计磁盘的离散读取操作，是一个比较缓慢的过程(后续有具体的优化措施)。

#### undo log 格式

![undo log](https://gitee.com/ssezhangpeng/picture/raw/master/MySQL/mysql.png)

- insert undo log
  - 由于 insert  操作的记录，只对事务本身可见，对其它事务不可见(事务隔离性要求)，故该 undo log 可以在事务提交后直接删除，不需要 purge 操作。如上图所示。
  - 主要字段
    - next: 下一个 undo log 的位置(就是这个字段串起来了 MVCC 中的版本链)
    - undo no: 记录了事务的ID
    - table id: 记录了undo log 所对应的表对象

- update undo log
  - update undo log 记录的是对 delete 和 update 操作产生的 undo log。该 undo log 可能需要提供 MVCC 机制，因此不能在事务提交之后立即进行删除。提交时放入 undo log 链表，等待 purge 线程进行最后的删除。
  - 基本字段和 insert undo log 一致，但记录的内容更高，具体字段可参见上图。

#### purge 操作

> ​    前面多次说过，delete 和 update 操作可能并不直接删除原有的数据，而是放在 purge 线程中进行操作。purge 操作是清理之前的 delete 和 update 操作，将上述操作“最终”完成。而实际执行的操作为 delete 操作，若**该行记录已经不被任何事务引用**，则可以清理之前行记录的版本。

- 优化点

> ​       前面我们说过了，undo page 存放 undo log，并且 InnoDB 存储引擎设计为 undo page 可以重用，因此一个 undo page 中可能存放了多个 undo log，另外，我们还有一个 history list 列表，事务提交后，将 undo log 插入该链表中。

- 执行过程
  - 先从 history list 中找到第一个需要被清理掉记录 tx1，清理之后，InnoDB 会在 tx1 的 undo log 所在的页中寻找是否存在可以被清理的记录，找到之后，会继续清理，如果找不到，则回到 history list 继续找第二个需要被清理的记录。
  - 每清理完一个页，都判断该页是否可以被重用，如果可以，则进行标记为可重用，下次可继续分配新的 undo log。
- 总结: InnoDB这种先从 history list 中找 undo log，然后再从 undo page 中找 undo log 的设计模式是为了**避免大量的随机读取操作**，从而提高 purge 的效率。



### 参考资料

1. MySQL技术内幕-InnoDB存储引擎(第2版)-7.2.2节 姜承尧