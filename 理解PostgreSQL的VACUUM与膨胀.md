S

原文地址：https://www.percona.com/blog/2018/08/06/basic-understanding-bloat-vacuum-postgresql-mvcc/



### PostgreSQL 中的版本是什么？（start: 2300）

​		让我们先来考虑下 MySQL 和 Oracle 中的情况。当你执行 UPDATE 或 DELETE 一个行时会发生什么呢？你会在全局 UNDO 段中看见一个 UNDO 记录。这个全局 UNDO 段包含了行的过去的镜像，帮助数据库完成一致性。(A.C.I.D 中的 C)。例如，有一个老事务依赖于一个删除的行，这个行对它仍旧是可见的，因为过去的镜像仍旧被维护在 UNDO 中。如果你是一个 Oracle DBA，读到了这个文章，你可能立刻会想到错误 `ORA-01555 snapshot too old`。这个错误的意思时 - 你可能拥有一个更小的 UNDO_retention 或 不是一个能保留所有的过去的镜像(版本)的大 UNDO 段，而这是已存在的事务或老事务所需要的。

​		在 PostgreSQL 中你不用担心这个问题。



### PostgreSQL 是如何管理 UNDO 的？

​		简单来讲，PostgreSQL 在表中同时维护了行的老版本和最新版本。**这意味着，UNDO 被维护在每个表里。**这是通过版本来做到的。现在，我们得到了提示：PostgreSQL 的表中的每个行都有一个版本号。这绝对是正确的。为了理解这些版本号如何维护在每个表里，你需要理解 PostgreSQL 表的隐藏列（特别是 xmin）。



### 理解表中的隐藏列

​		当你描述一个表时，你可能只看见你添加的列，就像你在下面的日志中看到的那样：

```sql
percona=# \d scott.employee
                                          Table "scott.employee"
  Column  |          Type          | Collation | Nullable |                    Default                     
----------+------------------------+-----------+----------+------------------------------------------------
 emp_id   | integer                |           | not null | nextval('scott.employee_emp_id_seq'::regclass)
 emp_name | character varying(100) |           |          | 
 dept_id  | integer                |           |          |
```



​		然而，如果你在 pg_attribute 表中查看表的所有列时，你可以看到几个隐藏的列，就像下面这样：

```sql
percona=# SELECT attname, format_type (atttypid, atttypmod)
FROM pg_attribute
WHERE attrelid::regclass::text='scott.employee'
ORDER BY attnum;
 attname  |      format_type       
----------+------------------------
 tableoid | oid
 cmax     | cid
 xmax     | xid
 cmin     | cid
 xmin     | xid
 ctid     | tid
 emp_id   | integer
 emp_name | character varying(100)
 dept_id  | integer
(9 rows)
```



​		让我们来了解一下这些隐藏列的细节：

##### tableoid

是所属表的 OID。被从继承结构中查询的SQL使用。更多关于表继承的细节看：https://www.postgresql.org/docs/10/static/ddl-inherit.html

##### xmin

创建这个行版本的事务 ID（XID）。对于 UPDATE，一个新的行版本被创建。让我们看下面的例子，对 xmin 有更深的理解：

```sql
percona=# select txid_current();
 txid_current 
--------------
          646
(1 row)

percona=# INSERT into scott.employee VALUES (9,'avi',9);
INSERT 0 1
percona=# select xmin,xmax,cmin,cmax,* from scott.employee where emp_id = 9;
 xmin | xmax | cmin | cmax | emp_id | emp_name | dept_id 
------+------+------+------+--------+----------+---------
  647 |    0 |    0 |    0 |      9 | avi      |       9
(1 row)
```



​		就像在上面的例子中看到的那样，命令 `select txid_current();` 的事务ID 是 646。因此，紧跟着的事务的事务ID是 647。这意味着，在事务 647 之前未启动新的事务，可以看到这个行。换句话说，早已运行的 XID 小于 647 的事务不能看到被事务 647 插入的行。

​		通过上面的例子，你现在应该理解了每个行都有一个 xmin，是创建它的事务的事务ID。

​		注意：这个行为依赖于你选的事务隔离级别，不同的隔离级别这个行为可能不同，后续的文章会讨论这个问题。

##### xmax

如果这不是一个被删除的行版本，xmax 是 0。在 DELETE 被提交前，行的 xmax 被修改为执行 DELETE 的事务的事务ID。让我用一个例子来更好的理解：

**终端 1**：开启一个事务，删除一个行，但不提交

```sql
percona=# BEGIN;
BEGIN
percona=# select txid_current();
 txid_current 
--------------
          655
(1 row)

percona=# DELETE from scott.employee where emp_id = 10;
DELETE 1
```

**终端 2**：在 DELETE 前后，观察 xmax 值（DELETE没有被提交）

```sql
-- Before the Delete
------------------
percona=# select xmin,xmax,cmin,cmax,* from scott.employee where emp_id = 10;
 xmin | xmax | cmin | cmax | emp_id | emp_name | dept_id 
------+------+------+------+--------+----------+---------
  649 |    0 |    0 |    0 |     10 | avi      |      10


-- After the Delete
------------------
percona=# select xmin,xmax,cmin,cmax,* from scott.employee where emp_id = 10;
 xmin | xmax | cmin | cmax | emp_id | emp_name | dept_id 
------+------+------+------+--------+----------+---------
  649 |  655 |    0 |    0 |     10 | avi      |      10
(1 row)
```

​		如上所示，xmax 值变成了执行 DELETE 的事务ID。如果你执行了 ROLLBACK ，或者事务被打断了，xmax 停留在试图删除它的事务ID上(这里是655)，在这个例子中。

​		现在我们理解了隐藏列 xmin 和 xmax，让我们观察一下，PostgreSQL 中执行了 DELETE 或 UPDATE 会发生什么。如我们早先讨论的那样，通过每个表的隐藏列，每个表中都维护了行的多个版本。让我们看下面的例子来更好的理解。

​		我们将插入10条记录到：scott.employee

```sql
percona=# INSERT into scott.employee VALUES (generate_series(1,10),'avi',1);
INSERT 0 10
```

​		然后，我们删除其中的5条：

```sql
percona=# DELETE from scott.employee where emp_id > 5;
DELETE 5
percona=# select count(*) from scott.employee;
 count 
-------
     5
(1 row)
```

​		现在，执行 DELETE 之后再看 COUNT，你将看不到被删除的记录。要看到表中存在但又不可见的行版本，我们需要一个叫做 pageinspect 的插件。pageinspect 模块提供了函数，允许你在一个低的层次查看数据库页的内容，这对于 DEBUG 来说是有用的。让我们来创建这个扩展看看那些被删除的更旧的行版本。

```sql
percona=# CREATE EXTENSION pageinspect;
CREATE EXTENSION

percona=# SELECT t_xmin, t_xmax, tuple_data_split('scott.employee'::regclass, t_data, t_infomask, t_infomask2, t_bits) FROM heap_page_items(get_raw_page('scott.employee', 0));
 t_xmin | t_xmax |              tuple_data_split               
--------+--------+---------------------------------------------
    668 |      0 | {"\\x01000000","\\x09617669","\\x01000000"}
    668 |      0 | {"\\x02000000","\\x09617669","\\x01000000"}
    668 |      0 | {"\\x03000000","\\x09617669","\\x01000000"}
    668 |      0 | {"\\x04000000","\\x09617669","\\x01000000"}
    668 |      0 | {"\\x05000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x06000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x07000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x08000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x09000000","\\x09617669","\\x01000000"}
    668 |    669 | {"\\x0a000000","\\x09617669","\\x01000000"}
(10 rows)
```

​		现在，我们仍然能看到 10 条记录，在已经删除了其中的 5 条之后。同时，你可以观察 t_max 被设置为删除该记录的事务ID 。这些被删除的记录被维护在相同的表里，为了那些老的仍旧需要访问这些记录的事务。

​		我们将会看一下一个 UPDATE 将会做什么。

```sql
percona=# DROP TABLE scott.employee ;
DROP TABLE
percona=# CREATE TABLE scott.employee (emp_id INT, emp_name VARCHAR(100), dept_id INT);
CREATE TABLE
percona=# INSERT into scott.employee VALUES (generate_series(1,10),'avi',1);
INSERT 0 10
percona=# UPDATE scott.employee SET emp_name = 'avii';
UPDATE 10
percona=# SELECT t_xmin, t_xmax, tuple_data_split('scott.employee'::regclass, t_data, t_infomask, t_infomask2, t_bits) FROM heap_page_items(get_raw_page('scott.employee', 0));
 t_xmin | t_xmax |               tuple_data_split                
--------+--------+-----------------------------------------------
    672 |    673 | {"\\x01000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x02000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x03000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x04000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x05000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x06000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x07000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x08000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x09000000","\\x09617669","\\x01000000"}
    672 |    673 | {"\\x0a000000","\\x09617669","\\x01000000"}
    673 |      0 | {"\\x01000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x02000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x03000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x04000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x05000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x06000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x07000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x08000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x09000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x0a000000","\\x0b61766969","\\x01000000"}
(20 rows)
```

​		PostgreSQL 中的一个 UPDATE 将会执行一个 INSERT 和一个 DELETE。因此，所有被 UPDATE 的记录都被删除了，并且插入了包含新的值的记录。被删除的记录的 t_max 值不为 0 。

​		基于合适的隔离级别，一些以前的事务因为一致性的原因还需要看到那些 t_max 值不为 0 的记录。

​		我们讨论了 xmin 和 xmax。那么隐藏列 cmin 和 cmax 又是什么呢？

##### cmin

​		插入事务中的命令的标识符。你应该可以看到 cmin 起始于 0，在下面的例子中。

##### cmax

​		删除事务中的命令的标识符，或 0 。然而，cmin 和 cmax 一直都是一样的，每一份 PostgreSQL 源码。



​		看下面的例子，理解 cmin 和 cmax 值随着 INSERT 和 DELETE 事务的改变。

```sql
On Terminal A
---------------
percona=# BEGIN;
BEGIN
percona=# INSERT into scott.employee VALUES (1,'avi',2);
INSERT 0 1
percona=# INSERT into scott.employee VALUES (2,'avi',2);
INSERT 0 1
percona=# INSERT into scott.employee VALUES (3,'avi',2);
INSERT 0 1
percona=# INSERT into scott.employee VALUES (4,'avi',2);
INSERT 0 1
percona=# INSERT into scott.employee VALUES (5,'avi',2);
INSERT 0 1
percona=# INSERT into scott.employee VALUES (6,'avi',2);
INSERT 0 1
percona=# INSERT into scott.employee VALUES (7,'avi',2);
INSERT 0 1
percona=# INSERT into scott.employee VALUES (8,'avi',2);
INSERT 0 1
percona=# COMMIT;
COMMIT
percona=# select xmin,xmax,cmin,cmax,* from scott.employee;
 xmin | xmax | cmin | cmax | emp_id | emp_name | dept_id 
------+------+------+------+--------+----------+---------
  644 |    0 |    0 |    0 |      1 | avi      |       2
  644 |    0 |    1 |    1 |      2 | avi      |       2
  644 |    0 |    2 |    2 |      3 | avi      |       2
  644 |    0 |    3 |    3 |      4 | avi      |       2
  644 |    0 |    4 |    4 |      5 | avi      |       2
  644 |    0 |    5 |    5 |      6 | avi      |       2
  644 |    0 |    6 |    6 |      7 | avi      |       2
  644 |    0 |    7 |    7 |      8 | avi      |       2
(8 rows)
```

​		观察上面的例子，你看见 cmin 和 cmax 值随着 INSERT 而增长。

​		现在，让我们在终端 A 中删除 3 条记录但不提交，然后我们在终端 B 中看这个值怎么呈现。

```sql
-- On Terminal A
---------------

percona=# BEGIN;
BEGIN
percona=# DELETE from scott.employee where emp_id = 4;
DELETE 1
percona=# DELETE from scott.employee where emp_id = 5;
DELETE 1
percona=# DELETE from scott.employee where emp_id = 6;
DELETE 1

-- On Terminal B, before issuing COMMIT on Terminal A
----------------------------------------------------
percona=# select xmin,xmax,cmin,cmax,* from scott.employee;
 xmin | xmax | cmin | cmax | emp_id | emp_name | dept_id 
------+------+------+------+--------+----------+---------
  644 |    0 |    0 |    0 |      1 | avi      |       2
  644 |    0 |    1 |    1 |      2 | avi      |       2
  644 |    0 |    2 |    2 |      3 | avi      |       2
  644 |  645 |    0 |    0 |      4 | avi      |       2
  644 |  645 |    1 |    1 |      5 | avi      |       2
  644 |  645 |    2 |    2 |      6 | avi      |       2
  644 |    0 |    6 |    6 |      7 | avi      |       2
  644 |    0 |    7 |    7 |      8 | avi      |       2
(8 rows)
```

​		现在，你可以在上面的例子中看出被删除的列中 cmin 和 cmax 从 0 开始增加。他们的值与删除之前不同，但很相似。即使你执行了 ROLLBACK，值还是保持这样。

​		在理解了隐藏列和 PostgreSQL 如何用多个行版本来维护 UNDO 之后，下一个问题是：什么会从这个表里清理这个 UNDO ？这会持续增加表的大小吗？为了更好的理解，我们需要知道 PostgreSQL 中的 VACUUM。



### PostgreSQL 中的 VACUUM

​		如上面的例子所示，每个已被删除但仍然占有空间的元组叫做 ***死亡元组***。如果不存在依赖于这些元组的已运行的事务，死亡元组就不再被需要了。因此，PostgreSQL 在这种表上运行 VACUUM 。VACUUM 回收被死亡元组占用的空间。这些空间可能会被引用为 ***Bloat***。VACUUM 扫描数据页寻找死亡元组，并把他们记录到 freespace map(FSM)。除 hash 索引之外的所有 relation 都有一个单独的文件来存储 FSM ，名为 <relation_oid>_fsm。

​		这里，relation_oid 是 relation 在 pg_class 表中的 oid。

```sql
percona=# select oid from pg_class where relname = 'employee';
  oid  
-------
 24613
(1 row)
```

​		VACUUM 之后，这个空间不会还给磁盘，但可以被这个表未来的 insert 复用。VACUUM 把空闲空间信息存储到 FSM 文件。

​		运行 VACUUM 是一个非锁定操作。它不会在表上造成排他锁。这意味着 VACUUM 可以运行在一个生产环境中事务繁忙的表上，当存在几个事务正在向它写入的时候。

​		像我们早先讨论的那样，一个更新 10 个元组的 UPDATE 生成了 10 个死亡元组。让我们看看下面的例子，来理解 VACUUM 之后这些死亡元组发生了什么。

```sql
percona=# VACUUM scott.employee ;
VACUUM
percona=# SELECT t_xmin, t_xmax, tuple_data_split('scott.employee'::regclass, t_data, t_infomask, t_infomask2, t_bits) FROM heap_page_items(get_raw_page('scott.employee', 0));
 t_xmin | t_xmax |               tuple_data_split                
--------+--------+-----------------------------------------------
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
    673 |      0 | {"\\x01000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x02000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x03000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x04000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x05000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x06000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x07000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x08000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x09000000","\\x0b61766969","\\x01000000"}
    673 |      0 | {"\\x0a000000","\\x0b61766969","\\x01000000"}
(20 rows)
```

​		在上面的例子中，你可能注意到死亡元组被删除了并且空间可以再利用。然而，这个空间没有被归还给文件系统。只有后续的 INSERT 可以重用这些空间。

​		VACUUM 做了一个额外的工作。所有在过去被插入并被提交的行被标记为冻结，这意味着它们对于所有现在和未来的事务是可见的。我们将要讨论这个的细节，在未来的博客 “PostgreSQL 的事务ID 回卷” 。

​		VACUUM 通常不会将空间归还给文件系统，除非死亡元组比例很高。

​		让我们考虑下面的例子，看看 VACUUM 将空间归还为文件系统。

​		创建一个表然后插入一些记录。这些记录在磁盘上被物理地排序基于主键。

```sql
percona=# CREATE TABLE scott.employee (emp_id int PRIMARY KEY, name varchar(20), dept_id int);
CREATE TABLE
percona=# INSERT INTO scott.employee VALUES (generate_series(1,1000), 'avi', 1);
INSERT 0 1000
```

​		现在，在表上运行 ANALYZE 来更新它的统计信息，并看看经过刚才的记录插入，为表申请了多少个页。

```sql
percona=# ANALYZE scott.employee ;
ANALYZE
percona=# select relpages, relpages*8192 as total_bytes, pg_relation_size('scott.employee') as relsize 
FROM pg_class 
WHERE relname = 'employee';
relpages | total_bytes | relsize 
---------+-------------+---------
6        | 49152       | 49152
(1 row)
```

​		让我们看一下，当你删除 emp_id > 500 的记录之后，VACUUM 会怎么做。

```sql
percona=# DELETE from scott.employee where emp_id > 500;
DELETE 500
percona=# VACUUM ANALYZE scott.employee ;
VACUUM
percona=# select relpages, relpages*8192 as total_bytes, pg_relation_size('scott.employee') as relsize 
FROM pg_class 
WHERE relname = 'employee';
relpages | total_bytes | relsize 
---------+-------------+---------
3        | 24576       | 24576
(1 row)
```

​		在上面的例子中，你可以看到 VACUUM 归还了表的一半空间给文件系统。之前，它使用了 6 个页(每个页 8KB，或者用 block_size 来设置)。在 VACUUM 之后，它释放了 3 个页给文件系统。

​		现在，让我们重复同样的动作，删除 emp_id < 500 的记录：

```sql
percona=# DELETE from scott.employee ;
DELETE 500
percona=# INSERT INTO scott.employee VALUES (generate_series(1,1000), 'avi', 1);
INSERT 0 1000
percona=# DELETE from scott.employee where emp_id < 500;
DELETE 499
percona=# VACUUM ANALYZE scott.employee ;
VACUUM
percona=# select relpages, relpages*8192 as total_bytes, pg_relation_size('scott.employee') as relsize 
FROM pg_class 
WHERE relname = 'employee';
 relpages | total_bytes | relsize 
----------+-------------+---------
        6 |       49152 |   49152
(1 row)
```

​		