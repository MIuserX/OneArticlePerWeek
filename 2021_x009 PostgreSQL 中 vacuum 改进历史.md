## 原文链接

https://www.enterprisedb.com/postgres-tutorials/history-improvements-vacuum-postgresql



## 翻译



​		当我开始使用 PostgreSQL 的时候，`autovacuum` 还不存在，也没意识到需要手动VACUUM。几个月后，我想直到为什么我的数据库这么慢。在 cron 任务里加了 `vacuumdb` 命令，每 6 小时运行一次，这对我那时面对的问题很有效，但这个方法奏效，只是因为我的数据库很小并且只有很少的流量。在大部分环境中，`UPDATE` 和 `DELETE` 操作对某些表比其他表更频繁，这些表将会比其他表更跨地积累死亡元组，因此 `VACUUM` 之间地间隔也各种各样。如果一个碰到这种环境的用户执行 full-database 的 `VACUUM` 足够频繁来满足那些更新最重的表的需求，这也会对那些更新轻的表进行了超过必要的 `VACUUM`，浪费效力。如果减少了 full-database `VACUUM` 的频率来避免浪费效力，重更新的表将无法被足够的 `VACUUM` 并且占用的磁盘空间持续增长直到他们被死亡元组填满，或被称作 “溢出”。

​		PG 8.3 是第一个带有合理的现代化的 `autovacuum` 的发布。`autovacuum` 首次被默认允许。`autovacuum` 也具有了多进程架构，意味着可以同时对多个表进行 `VACUUM`。当然，`autovacuum` 负责计算出哪些表要进行 `vacuum`，大大地（虽然不完全）消除了手动 `vacuum` 的需要。这是一个巨大的进步；PG 用户首次无需在配置 PG 时考虑配置 `VACUUM`。运气好的时候，它可以开箱即用完美地工作。

​		PG 8.4 带来了两项巨大的改进。在老版本中，存在一个固定大小的 free space map，这个 free space map 可以通过改变配置参数并重启来改变大小。如果数据库中的具有空闲空间的页的数量达到配置的 free space map 的大小，服务器将失去对某些包含空闲空间的页的跟踪，典型问题是 runaway database bloat。PG 8.4 引入了一个新的、动态调整大小的 free space map，这将不会丢失对空闲空间的跟踪。PG 8.4 还增加了一个 visibility map，使新的 `VACUUM` 不用再扫描表的未修改的部分。然而，每次扫描整个索引仍然是必要的，周期性的扫描劝全表是必须的，为了防止事务ID回卷。`VACUUM` 