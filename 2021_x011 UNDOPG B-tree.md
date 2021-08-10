## 原文

https://ieftimov.com/post/postgresql-indexes-btree/



## 翻译

​		关系型数据库中，索引是一个非常重要的特性，它缩短了我们查询数据的时间。在上一篇文章中：[the basics of indexes](https://ieftimov.com/postgresql-indexes-first-principles) ，我们讲了基本原理，并且知道我们如何在表上创建一个索引，并且测试它对我们查询的影响。在这篇文章中，我们仍然会看一些实现的细节，对于 PG 中最常用的索引：B-Tree。



### What is B-Tree?

​		从维基百科上看到的：

> 在计算机科学中，一个 B-Tree 是一个自平衡的树形数据结构，保持数据排序状态，并且允许查找、顺序访问、插入、删除操作，以 LOGn 时间复杂度。

​		真棒，这是真的嘛？基本上，它是个自排序的数据结构。这是它为什么自平衡的原因 - 它选择了它自己的形态。

> B-tree 是　二叉查找树，一个节点可以拥有超过两个的子节点。不像自平衡二叉查找树，B-tree 对于读写大块数据做了优化。

​		不像一般的二叉树，B-tree 可以拥有多个叶子节点，它基于自己平衡。同时，它的实现对于 I/O 进行了优化，这使得它适合做数据库的索引。

> B-tree 对于外存是一个好的例子。被广泛用于数据库和文件系统。

​		现在，对 B-tree 有了大量的理解。了解它们怎么工作是非常有趣的，所以让我们开始吧。



### Functionality

​		我们开始讲自平衡的 B-tree 之前，先说一个免责声明：有很多计算机科学的理论是关于 B-tree 的，但我们并不会涉及。例如，你可能首先想看一下二叉树，23-树，234树，在你深入了解 B-tree 之前。虽然如此，我们这里要讲到的内容依然对你理解B-tree 是如何工作的非常有帮助。

​		嘻嘻嘻，思考一下二叉树。在二叉树中，每个节点最多可以拥有两个子节点，因此叫做 二叉树。

​		GRAPH

​		好了，一个B-tree 是一个每个节点可以拥有多个子节点的，或者说，B-tree 可以拥有 N 个子节点。在二叉查找树中每个节点有一个值，B-tree 有个概念：**关键字**。关键字像一个值的列表，每个几点都有。

​		B-tree 也拥有概念：**阶**，一个阶为3的 B-tree 意味着每个非叶子节点可以拥有最多3个子节点。记住这个，这意味着每个节点最多拥有两个 (3-1) 关键字。

​		困惑吗？思考一下这个：一个非叶子节点，关键字是 `5,10`，你可以增加3个节点：

* 一个节点，值都小于 5
* 一个节点，值都在5和10之间
* 一个节点，值都大于10



​		我们将这个树画出来：

![img](https://ieftimov.com/b-tree-example-1.jpg)

​		关于 B-tree ，最重要的事是它们的平衡方面。这个概念基于每个节点都有关键字这个事实，像上面的例子。B-tree 平衡它们自己的方法非常有趣，关键字是这个功能的最重要的方面。

​		基本上，无论何时要添加一个新项目(或者，在我们的例子中，是个数字)，B-tree 为这个新项目寻找合适的位置(或节点)。例如，如果我们想添加数字 6，B-tree 将会 “寻找根节点”，在根节点上它应该将数字放置的位置。“寻找” 就是个比较操作，比较新的数字与节点的关键字。因为 6 大于 5，但是比 10 小（这是根节点的关键字），它将会创建一个新节点，在根节点之下：

![img](https://ieftimov.com/b-tree-example-2.jpg)

​		使用这个算法，B-tree 始终保持有序状态，在其中查找一个值时相当便宜。B-tree 有多种实现。这篇文章中，PG 使用了 B-tree 实现：“Lehman and Yao 的高并发 B-tree 管理算法”。你可以在 [这里](http://www.csd.uoc.gr/~hy460/pdf/p650-lehman.pdf) 读到实际的论文。

​		但是，这与 PG 的 B-tree 索引有什么联系呢？



### B-tree and PG

​		PG 中的索引，简单的讲，是被索引的列的数据的复制。唯一的不同是：数据的排序 - 索引的数据是被排了序的，这使得 PG 可以快速地查找和获取数据。例如，当你在表里查找一个记录，你查找的列被索引了，索引会缩短查询的耗时，因为PG 在索引中查找并且可以简单的找到数据在磁盘上的位置。

​		当你回想起索引是有序的时候，B-tree 数据结构刚好符合。在覆盖之下，索引是 B-tree，但是一个非常大的树。由于 B-tree 数据结构的本质，无论何时一个新记录被添加到被索引的表上，B-tree 知道如何再平衡并且保持有序。



### Limitations

​		几乎在所有的情况中，数据量比较大时索引的力量时非常显著的。这意味着索引将不得不与实际数据表一样大。

​		想象一下，如果我们处理几十亿的数据。这意味着索引表将拥有几十亿记录，存储在一个有序格式。OK，PG 可以处理这个情况。但是，你可以想象一个 `INSERT` 命令将要耗时多久嘛？将新记录添加到数据表，将会耗费很长时间，因为索引必须将记录添加到合适的位置，来保持索引的有序。因为这个限制，B-tree 索引的实现保留了页文件，这简单的呈现，是节点在一个大的 B-tree 数据结构。

​		虽然每个索引是一个整体，这个分页算法增加了一个索引数据的分离，但仍旧保持有序。在这种情况，代替将整个索引 dump 到内存中，仅需要添加一个记录，PG 找到新记录应该被添加的页并将索引的值写入这个页。

​		我的解释非常抽象，但这篇文章的主题是介绍 B-tree 数据结构，和它如何是个 PG 索引。如果你了解实现的更多细节，看 [Discovering the Computer Science Behind Postgres Indexes](http://patshaughnessy.net/2014/11/11/discovering-the-computer-science-behind-postgres-indexes) 从 Pat Shaughnessy。实际上，如果我知道它的文章的存在在我开始写这篇文章之前，我可能不会写这个。



### Using a B-tree index

​		先把 B-tree 索引的工作放一边，我们看看如何使用 B-tree 索引。给一个列添加一个 B-tree 索引的命令是：

```sql
CREATE INDEX name ON table USING btree (column);
```

​		或者，因为 `btree` 索引是默认的，我们可以省略 `using` 部分：

```sql
CREATE INDEX name ON table;
```

​		这将在 `table` 的 `name` 列上创建一个 B-tree 索引。



### Outro

​		在这篇文章中，我们大概了解了 PG 中常用的所用的索引之一：B-tree。我们看到了二叉查找树与 B-tree 的不同，和它们的行为如何融合入 PG。记住，由于文章的篇幅所限和目标读者，我忽略了大量的细节。



### Links

​		如果你想要更深入的了解 B-tree，无论是索引还是数据结构，这有一些有用的文章：

* [Efficient Locking for Concurrent Operations on B-tree](http://www.csd.uoc.gr/~hy460/pdf/p650-lehman.pdf)
* [B-tree](https://www.cs.utexas.edu/users/djimenez/utsa/cs3343/lecture16.html)
* [Anatomy of an SQL index](http://use-the-index-luke.com/sql/anatomy)
* [B-tree on Wikipedia](https://en.wikipedia.org/wiki/B-tree)