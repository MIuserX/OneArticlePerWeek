# 原文链接

https://www.interdb.jp/pg/pgsql01.html

# 翻译

这个章节和下个章节，总结了 PostgreSQL 的基础知识，帮助读者理解后续章节。这个章节中，将讨论如下主题：

* database cluster 的逻辑结构
* database cluster 的物理结构
* heap 表文件的内部结构
* 从一个表读取或者向一个表写入的方法

如果你早就对它们很熟悉了，你可以跳过这个章节。



## 1.1  database cluster 的逻辑结构

一个 **database cluster** 是 **database** 的集合。如果你是第一次听到这个定义，你可能会感到疑惑，但 **database cluster** 在 PostgreSQL 中不意味着 "一组 database server"。一个 PostgreSQL server 运行在一个单个的主机上并且管理着一个单个的 database cluster。

图 1.1 展示了 database cluster 的逻辑结构。

![img](https://www.interdb.jp/pg/img/fig-1-01.png)

一个 **database** 是一个 **database object** 的集合。在关系数据库理论中，一个 database object 是一个数据结构，用来存储或索引数据。一个 heap table 是 database object 的典型例子，还有很多其他的例子，像 索引、序列、视图、函数 等等。在 PostgreSQL 中，database 自身也是 database object 并且在逻辑上是互相分离的。所有的 database object 属于他们各自的 database。

PostgreSQL 中的所有 database object 在内部被各自的 **object identifier(OID)** 所标识，OID 是一个 4 字节长的整数。database object 之间的关系和各自的 OID 被存储在不同得 system catalog 中。例如，database 和 heap table 的 OID 被存在 `pg_database` 和 `pg_class` 中，所以你可以通过执行下面的 SQL 来获取你想知道的 object 的 OID：

```sql
sampledb=# SELECT datname, oid FROM pg_database WHERE datname = 'sampledb';
 datname  |  oid  
----------+-------
 sampledb | 16384
(1 row)

sampledb=# SELECT relname, oid FROM pg_class WHERE relname = 'sampletbl';
  relname  |  oid  
-----------+-------
 sampletbl | 18740 
(1 row)
```



## 1.2 database cluster 的物理结构

一个 database cluster 是一个目录叫做 **base directory**，它包含一些子目录和很多文件。如果你执行 initdb 命令来初始化一个新的 database cluster，一个 base directory 会被创建在执行的目录下。虽然这不是强制的，base directory 的路径通常被设置为环境变量 `PGDATA`。

图 1.2 展示了 PostgreSQL 的 database cluster 的例子。

![img](https://www.interdb.jp/pg/img/fig-1-02.png)

一个 database 是 base 子目录下的一个子目录，而每个表和索引是它所属的 database 目录中一个(或多个)文件。也有一些子目录包含特殊的数据，和配置文件。因为 PostgreSQL 支持 tablespaces ，这个术语的意义和其他的 RDBMS 不同。PostgreSQL 中的一个 tablespace 是一个子目录，包含一些在 base directory 之外的数据。



### 1.2.1 database cluster 的内部布局

database cluster 的内部布局在官方文档上有描述。官方文档上描述的主要文件和子目录如下图：

| files                             | description                                                  |
| --------------------------------- | ------------------------------------------------------------ |
| PG_VERSION                        | A file containing the major version number of PostgreSQL     |
| pg_hba.conf                       | A file to control PosgreSQL's client authentication          |
| pg_ident.conf                     | A file to control PostgreSQL's user name mapping             |
| postgresql.conf                   | A file to set configuration parameters                       |
| postgresql.auto.conf              | A file used for storing configuration parameters that are set in ALTER SYSTEM (version 9.4 or later) |
| postmaster.opts                   | A file recording the command line options the server was last started with |
| subdirectories                    | description                                                  |
| base/                             | Subdirectory containing per-database subdirectories.         |
| global/                           | Subdirectory containing cluster-wide tables, such as pg_database and pg_control. |
| pg_commit_ts/                     | Subdirectory containing transaction commit timestamp data. Version 9.5 or later. |
| pg_clog/ (Version 9.6 or earlier) | Subdirectory containing transaction commit state data. It is renamed to *pg_xact* in Version 10. CLOG will be described in [Section 5.4](https://www.interdb.jp/pg/pgsql05.html#_5.4.). |
| pg_dynshmem/                      | Subdirectory containing files used by the dynamic shared memory subsystem. Version 9.4 or later. |
| pg_logical/                       | Subdirectory containing status data for logical decoding. Version 9.4 or later. |
| pg_multixact/                     | Subdirectory containing multitransaction status data (used for shared row locks) |
| pg_notify/                        | Subdirectory containing LISTEN/NOTIFY status data            |
| pg_repslot/                       | Subdirectory containing [replication slot](http://www.postgresql.org/docs/current/static/warm-standby.html#STREAMING-REPLICATION-SLOTS)  data. Version 9.4 or later. |
| pg_serial/                        | Subdirectory containing information about committed serializable transactions (version 9.1 or later) |
| pg_snapshots/                     | Subdirectory containing exported snapshots (version 9.2 or later). The PostgreSQL's function pg_export_snapshot creates a snapshot information file in this subdirectory. |
| pg_stat/                          | Subdirectory containing permanent files for the statistics subsystem. |
| pg_stat_tmp/                      | Subdirectory containing temporary files for the statistics subsystem. |
| pg_subtrans/                      | Subdirectory containing subtransaction status data           |
| pg_tblspc/                        | Subdirectory containing symbolic links to tablespaces        |
| pg_twophase/                      | Subdirectory containing state files for prepared transactions |
| pg_wal/ (Version 10 or later)     | Subdirectory containing WAL (Write Ahead Logging) segment files. It is renamed from *pg_xlog* in Version 10. |
| pg_xact/ (Version 10 or later)    | Subdirectory containing transaction commit state data. It is renamed from *pg_clog* in Version 10. CLOG will be described in [Section 5.4](https://www.interdb.jp/pg/pgsql05.html#_5.4.). |
| pg_xlog/ (Version 9.6 or earlier) | Subdirectory containing WAL (Write Ahead Logging) segment files. It is renamed to *pg_wal* in Version 10. |



### 1.2.2 database 的内部布局

database 是一个在 base 目录下的子目录；数据库目录名字是唯一的，对于各自的 OID。例如，database `sampledb` 的 OID 是 16384，它的子目录名字是 16384。

```bash
$ cd $PGDATA
$ ls -ld base/16384
drwx------  213 postgres postgres  7242  8 26 16:33 16384
```



### 1.2.3 与表和索引关联的文件的内部布局

每个大小超过 1GB 的表或索引是一个单个的文件，存储在它所属的数据库目录中。表和索引作为数据库对象，在内部被 OID 唯一标识，同时，这些数据文件被变量：***relfilenode*** 标识。表和索引的 relfilenode 的值一般但不总是匹配各自的 OID，下面描述这些细节。

我们看一下表 `simpletbl` 的 OID 和 relfilenode ：

```sql
sampledb=# SELECT relname, oid, relfilenode FROM pg_class WHERE relname = 'sampletbl';
  relname  |  oid  | relfilenode
-----------+-------+-------------
 sampletbl | 18740 |       18740 
(1 row)
```

从上面的结果中可以看出，OID 和 relfilenode 是相等的。你也可以看到 `simpletbl` 表的数据文件路径是 `base/16384/18740`。

```sql
$ cd $PGDATA
$ ls -la base/16384/18740
-rw------- 1 postgres postgres 8192 Apr 21 10:21 base/16384/18740
```

表和索引的 relfilenode 的值可以被某些命令改变（像，`TRUNCATE`,`REINDEX`, `CLUSTER`）。例如，如果我们 truncate 表 `simpletbl`，PG 为表赋予一个新的 relfilenode，移除老的数据文件，并重建一个新的。

```sql
sampledb=# TRUNCATE sampletbl;
TRUNCATE TABLE

sampledb=# SELECT relname, oid, relfilenode FROM pg_class WHERE relname = 'sampletbl';
  relname  |  oid  | relfilenode
-----------+-------+-------------
 sampletbl | 18740 |       18812 
(1 row)
```

>在 PG 9.0 及以后的版本，内置的函数 `pg_relation_filepath` 对于使用 OID 或 表名 返回表的数据文件路径是非常有用的。
>
>```sql
>sampledb=# SELECT pg_relation_filepath('sampletbl');
> pg_relation_filepath 
>----------------------
> base/16384/18812
>(1 row)
>```
>
>

当表和索引的文件大小达到 1GB，PG 创建一个新文件，名字是 `{relfilenode}.1` 。如果新文件被数据占满，下一个文件 `{relfilenode}.2` 将会被创建，以此类推。

```sql
$ cd $PGDATA
$ ls -la -h base/16384/19427*
-rw------- 1 postgres postgres 1.0G  Apr  21 11:16 data/base/16384/19427
-rw------- 1 postgres postgres  45M  Apr  21 11:20 data/base/16384/19427.1
...
```

> 表和索引的最大文件大小可以在配置的时候改变，构建 PG 的时候使用 `--with-segsize` 即可。

仔细查看数据库子目录，你将会发现每个表拥有两个有关的文件，后缀分别是：`_fsm` 和 `_vm`。这些指的是 **free space map** 和 **visibility map**，存储了空闲空间和每个数据页的可见性的信息，

