## vacuum_freeze_min_age



## vacuum_freeze_table_age

正常的表的页里可能存在：

* 死亡的行 & 活着的行
* 老的行 & 新的行

冻结操作要找：还有活的行的页





## autovacuum_freeze_max_age

允许的最大值： 20亿

关联参数：vacuum_freeze_table_age





## 关键指标

* 行版本的年龄计算方法



## 一些问题

##### 1. 某个事务对过去、未来的可见的长度

一个事务可以看到的过去是 20亿，可以看到的未来也是 20亿。(distance=20亿)

但这个值可以配置嘛？：不可以，这是固定的算法。



##### 2. 并发的事务

在并发的系统中，同时存在多个 XID。最后启动的事务持有最新的 XID，最老的行版本持有最老的 XID。

他们俩的差，是最老的行版本的年龄。

vacuum_freeze_min_age 用来设置一个冻结年龄，一旦存在行版本的年龄大于冻结年龄，就应该用 VACUUM 来冻结这些行版本。



##### 3. VACUUM 的种类

VACUUM：正常VACUUM；侵入性VACUUM；

VACUUM FULL



##### 4. 困惑在哪里？

* 有些细节文档里好像没讲，不知道从哪里弄清楚，所以有困惑。
* 没有形成一个整体逻辑，所以感觉有些乱。



## 相关资料

Doc 19.4（讨论 vacuum 的 I/O 影响）；24.1（具体讨论 vacuum）；



## 整体逻辑

* vacuum 为什么存在？
* vacuum 的功能、机制、原理
* vacuum 相关的配置参数
* vacuum 参数的最佳实践（分场景）
* 实践中如何操作？（操作手册）



##### 1. vacuum 为什么存在？

存在的意义是满足某种需要，这就要看 vacuum 做了什么工作？做这些工作的意义是什么？不做行吗？

原因有四：

* 因为 PG 的 MVCC 机制，需要清理死亡元组，回收磁盘空间。
* 防止事务ID回卷失败
* 更新统计信息，让查询规划器能做出更好的查询计划，提高性能。
* 更新可见性地图

上面的工作必须做，需要一个机制，这个机制就是 vacuum，这就是 vacuum 存在的原因，它是整个系统的一部分，不是额外的东西。



##### 2. 详细说说 vacuum 的每个功能





##### 3. vacuum 相关的参数的意义

```
vacuum_cost_delay (float)：累计的cost达到这个阈值时，vacuum进程sleep的时间由这个参数指定。 
vacuum_cost_page_hit (int)：
vacuum_cost_page_miss (int)：
vacuum_cost_page_dirty (int)：
vacuum_cost_limit (int)：累计的cost达到这个阈值时，vacuum进程会 sleep 一段时间。

vacuum_freeze_min_age：
vacuum_freeze_table_age：
vacuum_multixact_freeze_min_age：
vacuum_multixact_freeze_table_age：

autovacuum_freeze_max_age：
autovacuum_multixact_freeze_max_age：
```





##### 4. 实践中如何调整 vacuum 的这些参数？

总体的思路是：

工具是为了满足需求，如果基本满足项目需求的话，其实进一步优化是不紧急事项。

只有出现了问题的迹象，才需要处理一下。



1. 只用 autovacuum（大多数实例这样就够用了）
2. autovacuum 与 人工vacuum 混用
3. 只用人工vacuum

感觉第 2 种在大部分场景下效果最好。 自动处理一些一般的表，人工处理一些特殊情况的表，总体达到最好效果。

1. 尽量使用 vacuum，不使用 vacuum full。
2. 一般使用 vacuum，特殊情况使用 vacuum full。
3. 只使用 vacuum full。

三种方法都有适用场景，大部分使用 2 应该是合适的。

