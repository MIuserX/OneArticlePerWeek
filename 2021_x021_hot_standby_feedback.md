# 原文

https://www.cybertec-postgresql.com/en/what-hot_standby_feedback-in-postgresql-really-does/



# 翻译

很多使用 PG 流复制的人都想知道 `hot_standby_feedback` 这个参数实际上做了什么。很多客户一直问这个问题，因此分享这个知识可能是非常有用的。



## PG 的 vacuum 都做了什么？

PG 中 vacuum 是个重要的命令，它的目标是清理不再被任何事务需要的死亡元组。在插入新数据的时候重新使用表中的空闲空间。重要的是：vacuum 的目标是重用表中的空闲空间 - 这不一定意味着表会缩小。同时也要记住：vacuum 只会清理死亡元组，如果这些元组不再被其他运行的事务所需要。

![img](https://www.cybertec-postgresql.com/wp-content/uploads/2018/08/hot_standby_feedback-01-VACUUM-NEU.jpg)



如你所见，这里有两个连接。左边的连接正在运行一个耗时很长的 SELECT。现在思考一下：一个 SQL 会 “冻结” 它的数据视图。在 SQL 内部，世界不会发生 “改变” - 这个查询将会一直看到相同的数据集，无视那些并发的修改。这对于理解真的非常重要。

让我们看一下右边的连接，它会删除一些数据并提交。随之而来的问题是：什么时候 PG 可以真正的从磁盘上删除这些数据？DELETE 并不会真正地从磁盘上删除这些行，因为可能后面会是 ROLLBACK 代替 COMMIT。换句话说，行一定不会在 DELETE 时删除。PG 仅会将它标记为死亡元组，对于当前的事务。如你所见，其他的事务可能还能看到这些死亡元组。

然而，甚至 COMMIT 也没有权力真正删除死亡元组。记住：左边的事务仍然可以看到死亡元组，因为 SELECT 不会在它运行的时候改变它的快照。因此，COMMIT 对于真正删除死亡元组来说还是太早。

这就该 VACUUM 上场了。VACUUM 来清理死亡元组，清理那些无法被任何事务看到的元组。在下图里，有两个 VACUUM 在执行。第一个 VACUUM 不能清理掉死亡元组，因为它仍然能被左边的事务看见。

然而，第二个 VACUUM 可以清理掉这个死亡元组，因为它不在被任何读取事务看见。

在单个服务器上，这个流程很清晰。VACUUM 可以清理那些无法被任何事务看到的死亡元组。



## PG 的复制冲突

Primary/Standby 场景下会发生什么问题呢？环境变的有点复杂，因为 primary 无法知道 standby 上正在运行的事务。

![img](https://www.cybertec-postgresql.com/wp-content/uploads/2018/08/vacuum_cleanup-01-939x1024.jpg)



在这个场景下，一个 SELECT 在 standby 上运行了好多分钟。同时，primary 上发生了一些变化（UPDATE、DELETE，或者其他的）。这仍然没问题。记住：DELETE 不会真正删除这些元组 - 它只是简单地将其标记为死亡元组，但对其他事务仍然是可见的。情况在 primary 上允许 VACUUM 清理死亡元组而变的糟糕。VACUUM 被允许这样做因为它不知道 standby 上仍有事务需要这个元组。结果是发生复制冲突。默认上，一个复制冲突会在 30s 内解决：

```
ERROR: canceling statement due to conflict with recovery
Detail: User query might have needed to see row versions that must be removed
```

如果你曾看到过这个信息 - 这就是我们讨论的问题。



## hot_standby_feedback 可以防止复制冲突

要解决此类的问题，我们可以让 standby 周期性地通知 primary 关于运行的最老的事务的信息。如果 primary 知道 standby 上运行的老事务的信息，它可以让 VACUUM 保留元组直到 standby 上的老事务完成。

这就是 hot_standby_feedback 实际上做的事。它防止 primary 过早地清理了 standby 上的事务还需要的元组。解决方法是通知 primary 关于 standby 上运行的最老的事务的 ID，因此 VACUUM 可以推迟它的清理工作。

好处是显而易见的：hot_standby_feedback 将会大量减少复制冲突的数量。然而，也有一些负作用：记住，VACUUM 会延迟它的清理操作。如果 standby 不结束它的查询，它会引起主库的表膨胀，在长时间的运行下这会导致危险。