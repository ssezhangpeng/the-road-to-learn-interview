### redo log

> ​    重做日志是用来实现事务的持久性, 其中有两部分组成: 一是内存中的重做日志缓冲(redo log buffer), 其是易失性的；二是重做日志文件(redo log file), 其是持久性的。

#### 作用

- 实现事务的持久性，即ACID中的 D
- 可以通过 redo log 进行数据同步(基于 redo log 进行同步)
- 可以通过 redo log 进行 crash 恢复

#### 存储管理

- Log block

  > ​    在 InnoDB存储引擎中，redo log 都是以 512字节进行存储的。这意味着 redo log buffer 和 redo log file 都是以 块(block) 的方式进行保存的，称之为重做日志块。另外，因为重做日志块的大小和磁盘扇区大小一样，都是 512字节，因此重做日志的写入可以保证原子性，不需要***doublewrite技术***。

- redo log buffer

  > ​    redo log buffer 是由 log block 组成，好似一个数组，由 LOG_BLOCK_HDR_NO 用来标记这个数组中的位置，其是递增并且循环使用的，所以逻辑上可以把这个 buffer 看作一个环形 buffer。该 buffer 中的脏页根据一定的规则刷新的磁盘，规则如下:
  >
  > - 事务提交时
  > - 当log buffer 中有一半的内存空间已经被使用时
  > - log checkpoint 时
  >
  > ​    log block 总是写入追加(append)到redo log file 的最后部分，当一个 redo log file被写满时，会接着写下一个 redo log file，其使用方式为 round-robin。

- log group

  > ​    log group 为重做日志组，目前InnoDB只有一个，一个日志组中可以有多个重做日志文件(默认为2个)。每一个组中的第一个 redo log file 前 2KB 空间中，不存放 Log block，而是存放了一些和数据恢复相关的重要信息，比如 checkpoint(检查点)。

- LSN

  > ​    日志序列号(Log Sequence Number)，占用 8个字节，并且单调递增，标识的含义是
  >
  > - 重做日志写入的总量(单位为字节)
  > - checkpoint 的位置
  > - 页的版本
  >
  > ​    LSN 不仅记录在重做日志中，还存在于每个页中。在每个页的头部，又一个值为 FIL_PAGE_LSN，记录了该页的LSN。标识该页最后刷新时候LSN的大小。因为重做日志记录的是每个页的修改，因此**页中的LSN用来判断该页是否需要进行恢复操作**。
  >
  > ​    可以使用如下命令进行查看 LSN 的当前情况:
  >
  > ```mysql
  > mysql> show engine innodb status\G;
  > --- Log sequence number: 标识当前的 LSN
  > --- Log flushed up to:   标识刷新到重做日志文件的 LSN
  > --- Last checkpoint at:  标识刷新到磁盘的 LSN
  > ```

#### 参考资料

1. MySQL技术内幕-InnoDB存储引擎(第2版)-7.2.1节 姜承尧

2. http://catkang.github.io/2020/02/27/mysql-redo.html