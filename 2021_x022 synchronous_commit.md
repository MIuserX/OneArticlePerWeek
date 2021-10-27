# 原文

https://www.percona.com/blog/2020/08/21/postgresql-synchronous_commit-options-and-synchronous-standby-replication/



# 翻译

## synchronous_commit 是关于什么?

这个参数可以控制事务提交的哪个阶段向客户端回复事务执行成功的消息。

所以这个参数不仅仅与同步从库有关，它对于独立运行的数据库实例也有更广法的意义和牵连。为了更好的理解这个参数，~~我们要看看 WAL 的整个传播过程和确认事务提交的信号可以接受的各个阶段~~。这允许我们对每个事务选择各种级别的时间。选择的持续时间越短，越快获得事务提交确认信息，这会提高系统的整个吞吐量和性能。



## WAL 传播过程

PG WAL 是在主库上的 修改/活动 的记录，可以被视为是数据库发生的变化的日志/总账。下面的图展示了 WAL 在主节点和从节点的传播流程。 

![img](https://www.percona.com/blog/wp-content/uploads/2020/08/sychronous_commit-1.png)

> PG 使用内部函数 `pg_pwrite()` 向 WAL segments 中写入，`pg_pwrite()` 内部使用的是 `write()` 系统调用，这个系统调用不保证数据被刷入磁盘。要完成刷盘操作，需要调用另一个函数 `issue_xlog_fsync()` ，这个函数会根据 (GUC)`wal_sync_method` 函数使用合适的 fsync 方法。

上面的图展示了 5 个主要阶段：

1. **WAL Record Inserts (local)**：WAL 产生在 WAL buffer 中。因为多个后台进程会同时创建 WAL，WAL buffer 被锁很好的保护。WAL buffers 中的 WAL 被几个不同的后台进程持续的写入 WAL segments。如果 `synchronous_commit` 被完全关闭，刷盘操作不会立刻执行，而是依赖于 `wal_write_delay` 配置(这个配置后面讨论)。
2. **WAL Writes and WAL Flush(local)**：这个阶段把 WAL 刷入本地磁盘 "segment files" ，这被认为是一种重操作。PG 在这方面做了很多优化，为了避免频繁的刷盘。
3. **Remote Write**：WAL 被写入从库（但还没有刷盘）。数据可能还保留在页缓存上一段时间。除非我们想 ~~address~~ 主从库同时宕机的情况，否则这个级别可以作为选项。
4. **Remote Flush**：在这个阶段，数据被真正的写入并刷到从库的磁盘上。我们能够保证数据在从库上可以获取到，即使从库也宕机。
5. **Remote Apply**：在这个阶段，WAL 在从库上已经被 replay ，对于正在运行的会话是可见的。



相应的对于 `synchronous_commit` 可接受的值有：

1. **off**：可使用的值是 `off`，`0(zero)`，`false`，`no` ，这会关闭 `synchronous_commit` 。提交成功信号会在刷盘之前返回。一般这被叫做 “异步提交”。如果 PG 实例宕机，最后若干异步提交的事务会丢失。
2. **local**：WAL 被写入并刷入本地磁盘。在这个场景中，提交成功信号会在刷盘之后返回。
3. **remote_write**：WAL 被成功地传递给了远程实例，并被远程实例写入（但还未刷盘）。
4. **on**：这是默认值，你可以使用 `on`，`true`，`yes`，`1` 。但它的意义会因是否存在同步从库而发生改变。如果存在同步从库，设置这个级别，会导致等待从库的 "刷盘" 完成。
5. **remote_apply**：这会导致主库等待从库 replay 完 WAL（对运行在从库的会话可见）。



通过选择合适的级别，我们可以决定 WAL 传播的哪个阶段返回事务提交成功的信号。如果不存在同步备库(`synchronous_standby_names` 为空)，`synchronous_commit` 设置为 `on, remote_apply, remote_write, local` 时，它们表示同一种级别：本地刷盘后返回。



这个领域经常被问到的问题是：

***如果使用异步流复制(synchronous_commit=off)我们会丢失多少数据？***

答案有些复杂，而且依赖于 `wal_writer_delay` 的设置。默认是 200ms。这意味着 **WAL 会在每 `wal_writer_delay` 毫秒后刷入磁盘**。WAL Writer 周期性的醒来并调用 `XLogBackgroundFlush()`。这个操作检测被写满的 WAL 页。如果存在，它把 buffers 刷入磁盘。所以在良好的负载下，WAL writer 写整个 buffers。在低负载下，找不到写满的页，上一次异步提交之后的所有 WAL 会被刷盘。

如果超过 `wal_writer_delay` 毫秒的时间过去，或者从上次刷盘之后写入的 blocks 超过 `wal_writer_flush_after` 个。计划会保证事务完成后**最多两个 `wal_writer_delay`** 后会被刷盘。然而，PG 写入/刷入 写满的 buffer 时，以一种更复杂的方式，这是为了减少因负载过重，每个 WAL Writer 循环中多个 WAL page 被满时的写入问题。概念上，这造成 **最坏的情况延迟到3个 `wal_writer_delay` 循环**。

所以，简单来讲，答案是：

***大部分情况下，丢失将会少于两个 `wal_writer_delay` 时间内的事务。但最坏情况会丢失 3 倍 `wal_writer_delay` 时间内的事务。***



## 配置的作用域

当我们讨论参数和值时，很多用户考虑的时全局设置 `synchronous_commit` 在实例级别。但是它真正的威力和使用是在不同的级别上设置。在实例级别上修改它可能不是想要的。

PG 允许我们在不同作用域中设置该值。

1. **在事务级别**

    在一个调整良好的应用设计中，特定的事务可以选择特定的提交级别，对于每个事务，例如：

    ```sql
    SET LOCAL synchronous_commit = 'remote_write';
    ```

    记得写 `LOCAL` 关键字。一旦事务块完成(提交或回滚)，这个设置将会恢复到会话级别的设置值。这允许设计师选择额外的日常支出对于特定的事务。

2. **在会话级别**

    这个设置可以在会话级别上指定，它会在整个会话声明周期内生效，除非被事务级别的设置覆写。

    ```sql
    SET synchronous_commit = 'remote_write';
    ```

    可能应用会把这个值作为连接串选项传递给 PG 。例如：`"host=hostname user=postgres ... options='-c synchronous_commit=off'"` 。这样可以减少代码的修改。

3. **在用户级别**

    在一个理想的系统中，具有良好的用户管理，每个用户只会处理特定的功能。它从重要的事务管理系统到报告用户账户。例如：

    ```sql
    ALTER USER trans_user SET synchronous_commit= ON;
    ALTER USER report_user SET synchronous_commit=OFF;
    ```

    这些用户创建的会话将会默认拥有这些设置。这个用户级别的设置可以被会话级别的设置或事务级别的设置覆写。

4. **在 database 级别**

    如果我们拥有专用系统用来报告或临时阶段信息，指定 database 级别是非常有用的。

    ```sql
     ALTER DATABASE reporting SET synchronous_commit=OFF;
    ```

    

5. **在实例级别**

    在实例级别设置如下：

    ```sql
    ALTER SYSTEM SET synchronous_commit=OFF;
    SHOW synchronous_commit;
    ```



## 常见使用场景

**迁移**：当迁移发生时，跨系统的大数据移动是非常常见的，合适的设置 `synchronous_commit` 或干脆关闭 将是一个合适的选择，来减少整体的迁移时间。

**数据加载**：在数据仓库系统/报表系统，会发生加载大数据的情况，关闭 `synchronous_commit` 将会很大提速，因为减少了重复刷盘的额外消耗。

**审计与日志**：甚至在关键的系统中，仅仅有一部分事务是非常关键的 - 一个 business 想变的可获得 - 在从库上，在提交成功信号返回之前。但存在相关的日志和审计信息记录。非常有选择性选择，对于同步流复制从库提交，可以获得非常高的收益。





## 尾声

使用 pgbench 进行一个快速的测试，可以帮助确认在特定的环境中不同同步提交级别下的消耗。总而言之，当我们提高同步提交级别时，性能是下降的。环境因素，像 fsync 到本地磁盘的延迟、网络延迟、备库上的负载、备库的竞争、整个复制量、备库的磁盘性能，等等，都会影响消耗和性能。

甚至完全关闭 `synchronous_commit` 不会导致数据库打断，不像 fsync。理解整个 WAL 传播过程可以帮助里理解复制延迟，和 pg_stat_replicaiton 视图中的信息。

一个系统的性能是关于 消除不想要的、可以避免的消耗，而不牺牲重要的东西。我看到过一些强大的 PG 用户，拥有调优良好的系统对于 PG ，通过让 `synchronous_commit` 的使用非常高效和有选择性。我希望这个文章将会帮助那些仍没有使用它的并且在寻找更高性能的调优方法。

