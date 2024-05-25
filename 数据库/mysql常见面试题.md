### NULL 和 '' 的区别是什么？

- `NULL` 代表一个不确定的值,就算是两个 `NULL`,它俩也不一定相等。例如，`SELECT NULL=NULL`的结果为 false，但是在我们使用`DISTINCT`,`GROUP BY`,`ORDER BY`时,`NULL`又被认为是相等的。

- `''`的长度是 0，是不占用空间的，而`NULL` 是需要占用空间的。

- `NULL` 会影响聚合函数的结果。

  例如，`SUM`、`AVG`、`MIN`、`MAX` 等聚合函数会忽略 `NULL` 值。

   COUNT 的处理方式取决于参数的类型。如果参数是 COUNT(*)，则会统计所有的记录数，包括 NULL 值；

  如果参数是某个字段名(`COUNT(列名)`)，则会忽略 `NULL 值，只统计非空值的个数。

- 查询 `NULL` 值时，必须使用 `IS NULL` 或 `IS NOT NULLl` 来判断，而不能使用 =、!=、 <、> 之类的比较运算符。而`''`是可以使用这些比较运算符的。

### [Boolean 类型如何表示？](#boolean-类型如何表示)

MySQL 中没有专门的布尔类型，而是用 TINYINT(1) 类型来表示布尔值。TINYINT(1) 类型可以存储 0 或 1，分别对应 false 或 true。

**MySQL基础架构**

![img](https://oss.javaguide.cn/javaguide/13526879-3037b144ed09eb88.png)

缓存器在8.0后移除了

MySQL 当前默认的存储引擎是 InnoDB

**binlog在server层**

MySQL 存储引擎采用的是 **插件式架构** ，支持多种存储引擎，我们甚至可以为不同的数据库表设置不同的存储引擎以适应不同场景的需要。**存储引擎是基于表的，而不是数据库。**

### 更新语句的执行过程

~~~sql
update tb_student A set A.age='19' where A.name=' 张三 ';
~~~

- 先查询到张三这一条数据

- 然后拿到查询的语句，把 age 改为 19，然后调用引擎 API 接口，写入这一行数据，

  InnoDB 引擎把数据保存在内存中（数据页），

  同时记录 redo log，此时 redo log 进入 prepare 状态，然后告诉执行器，执行完成了，随时可以提交

- 执行器收到通知后记录 binlog，然后调用引擎接口，提交 redo log 为提交状态。
- 更新完成

### [MyISAM 和 InnoDB 有什么区别？](https://javaguide.cn/database/mysql/mysql-questions-01.html#myisam-和-innodb-有什么区别)

MySQL 5.5 之前，MyISAM 引擎是 MySQL 的默认存储引擎，但是，MyISAM 不支持事务和行级锁，而且最大的缺陷就是崩溃后无法安全恢复。

**1.是否支持行级锁**

MyISAM 只有表级锁(table-level locking)，而 InnoDB 支持行级锁(row-level locking)和表级锁,默认为行级锁。

**2.是否支持事务**

MyISAM 不提供事务支持。

InnoDB 提供事务支持，实现了 SQL 标准定义了四个隔离级别，具有提交(commit)和回滚(rollback)事务的能力。并且，InnoDB 默认使用的 REPEATABLE-READ（可重读）隔离级别是可以解决幻读问题发生的（**基于 MVCC 和 Next-Key Lock**）。

**3.是否支持外键**

MyISAM 不支持，而 InnoDB 支持。

外键对于维护数据一致性非常有帮助，但是对性能有一定的损耗。因此，**通常情况下，我们是不建议在实际生产项目中使用外键**的，在业务代码中进行约束即可！

**4.是否支持数据库异常崩溃后的安全恢复**

MyISAM 不支持，而 InnoDB 支持。

使用 InnoDB 的数据库在异常崩溃后，数据库重新启动的时候会保证数据库恢复到崩溃前的状态。这个恢复的过程依赖于 `redo log` 。

**5.是否支持 MVCC**

MyISAM 不支持，而 InnoDB 支持。

 MyISAM 连行级锁都不支持。MVCC 可以看作是行级锁的一个升级，可以有效减少加锁操作，提高性能。

**6.索引实现不一样。**

虽然 MyISAM 引擎和 InnoDB 引擎都是使用 B+Tree 作为索引结构，但是两者的实现方式不太一样。

InnoDB 引擎中，其数据文件本身就是索引文件。

相比 MyISAM，索引文件和数据文件是分离的，索引文件包含了 B+ 树结构，而数据文件则包含了实际的数据记录。

**总结**：

- InnoDB 支持行级别的锁粒度，MyISAM 不支持，只支持表级别的锁粒度。
- MyISAM 不提供事务支持。InnoDB 提供事务支持，实现了 SQL 标准定义了四个隔离级别。
- MyISAM 不支持外键，而 InnoDB 支持。
- MyISAM 不支持 MVCC，而 InnoDB 支持。
- 虽然 MyISAM 引擎和 InnoDB 引擎都是使用 B+Tree 作为索引结构，但是两者的实现方式不太一样。
- MyISAM 不支持数据库异常崩溃后的安全恢复，而 InnoDB 支持。
- InnoDB 的性能比 MyISAM 更强大。

### 表级锁和行级锁对比

- **表级锁：** MySQL 中锁定粒度最大的一种锁（全局锁除外），**是针对非索引字段加的锁，**对当前操作的整张表加锁，实现简单，资源消耗也比较少，加锁快，不会出现死锁。不过，触发锁冲突的概率最高，高并发下效率极低。

- **行级锁：** MySQL 中锁定粒度最小的一种锁，是 **针对索引字段加的锁** ，只针对当前操作的行记录进行加锁。 行级锁能大大减少数据库操作的冲突。其加锁粒度最小，并发度高，但加锁的开销也最大，加锁慢，会出现死锁。行级锁和存储引擎有关，是在存储引擎层面实现的。

  当我们执行 `UPDATE`、`DELETE` 语句时，如果 `WHERE`条件中字段没有命中唯一索引或者索引失效的话，就会导致扫描全表对表中的所有行记录进行加锁

### InnoDB 有哪几类行锁？

**记录锁（Record Lock）**：也被称为记录锁，属于单个行记录上的锁。

**间隙锁（Gap Lock）**：锁定一个范围，不包括记录本身。

**临键锁（Next-Key Lock）**：Record Lock+Gap Lock，锁定一个范围，包含记录本身，主要目的是为了解决幻读问题（MySQL 事务部分提到过）。

在 InnoDB 默认的隔离级别 REPEATABLE-READ 下，行锁默认使用的是 Next-Key Lock。**但是，如果操作的索引是唯一索引或主键，InnoDB 会对 Next-Key Lock 进行优化，将其降级为 Record Lock，即仅锁住索引本身，而不是范围。**因为唯一索引和主键都有确定的值

### MySQL可以直接存储文件本身吗

一般不建议，可以使用阿里云oss网络文件存储服务

**数据库只存储文件地址信息，文件由文件存储服务负责存储。**

### 如何存储IP地址

可以将 IP 地址转换成整形数据存储，性能更好，占用空间也更小。

### [对于频繁的查询优先考虑使用覆盖索引](https://javaguide.cn/database/mysql/mysql-high-performance-optimization-specification-recommendations.html#对于频繁的查询优先考虑使用覆盖索引)

只需要在一棵索引树上就能获取SQL所需的所有列数据，无需回表，速度更快

- **避免 InnoDB 表进行索引的二次查询，也就是回表操作**
- **可以把随机 IO 变成顺序 IO 加快查询效率**

### [避免使用子查询，可以把子查询优化为 join 操作](https://javaguide.cn/database/mysql/mysql-high-performance-optimization-specification-recommendations.html#避免使用子查询-可以把子查询优化为-join-操作)

**子查询性能差的原因：** 子查询的结果集无法使用索引，通常子查询的结果集会被存储到临时表中，不论是内存临时表还是磁盘临时表都不会存在索引，所以查询性能会受到一定的影响

### [对应同一列进行 or 判断时，使用 in 代替 or](https://javaguide.cn/database/mysql/mysql-high-performance-optimization-specification-recommendations.html#对应同一列进行-or-判断时-使用-in-代替-or)

### [在明显不会有重复值时使用 UNION ALL 而不是 UNION](#在明显不会有重复值时使用-union-all-而不是-union)

- UNION 会把两个结果集的所有数据放到临时表中后再进行去重操作
- UNION ALL 不会再对结果集进行去重操作

### [超 100 万行的批量写 (UPDATE,DELETE,INSERT) 操作,要分批多次进行操作](https://javaguide.cn/database/mysql/mysql-high-performance-optimization-specification-recommendations.html#超-100-万行的批量写-update-delete-insert-操作-要分批多次进行操作)

**大批量操作可能会造成严重的主从延迟**

主从环境中,大批量操作可能会造成严重的主从延迟，大批量的写操作一般都需要执行一定长的时间，而只有当主库上执行完成后，才会在其他从库上执行，所以会造成主库与从库长时间的延迟情况

**binlog 日志为 row 格式时会产生大量的日志**

大批量写操作会产生大量日志，特别是对于 row 格式二进制数据而言，由于在 row 格式中会记录每一行数据的修改，我们一次修改的数据越多，产生的日志量也就会越多，日志的传输和恢复所需要的时间也就越长，这也是造成主从延迟的一个原因

**避免产生大事务操作**

大批量修改数据，一定是在一个事务中进行的，这就会造成表中大批量数据进行锁定，从而导致大量的阻塞，阻塞会对 MySQL 的性能产生非常大的影响。

**特别是长时间的阻塞会占满所有数据库的可用连接**，这会使生产环境中的其他应用无法连接到数据库，因此一定要注意大批量写操作要进行分批



### 索引详解

按照底层存储方式角度划分

聚簇索引（聚集索引）：索引结构和数据一起存放的索引，InnoDB 中的主键索引就属于聚簇索引

**找到了索引就找到了需要的数据，那么这个索引就是聚簇索引**

非聚簇索引（非聚集索引）：索引结构和数据分开存放的索引，二级索引(辅助索引)就属于非聚簇索引。MySQL 的 MyISAM 引擎，不管主键还是非主键，使用的都是非聚簇索引。也就是需要回表

#### 聚簇索引的优缺点

**优点**：

- **查询速度非常快**：相比于非聚簇索引， 聚簇索引少了一次读取数据的 IO 操作。
- **对排序查找和范围查找优化**：聚簇索引对于主键的排序查找和范围查找速度非常快。

**缺点**：

- **依赖于有序的数据**：因为 B+树是多路平衡树，如果索引的数据不是有序的，那么就需要在插入时排序，如果数据是整型还好，否则类似于字符串或 UUID 这种又长又难比较的数据，插入或查找的速度肯定比较慢。
- **更新代价大**：如果对索引列的数据被修改时，那么对应的索引也将会被修改，而且聚簇索引的叶子节点还存放着数据，修改代价肯定是较大的，所以对于主键索引来说，主键一般都是不可被修改的。

### 

如果一个索引包含（或者说覆盖）所有需要查询的字段的值，我们就称之为 **覆盖索引**

### 联合索引

使用表中的多个字段创建索引，就是 **联合索引**，也叫 **组合索引** 或 **复合索引**

最左前缀匹配原则指的是在使用联合索引时，MySQL 会根据索引中的字段顺序，从左到右依次匹配查询条件中的字段。

最左匹配原则会一直向右匹配，直到遇到范围查询（如 >、<）为止。

**对于 >=、<=、BETWEEN 以及前缀匹配 LIKE 的范围查询，不会停止匹配**

### 索引下推

它允许存储引擎在索引遍历过程中，执行部分 `WHERE`字句的判断条件，直接过滤掉不满足条件的记录，从而减少回表次数，提高查询效率

如下面先查到431200的zipcode，再根据birthday = 12的过滤掉后再回表查询

联合索引`(zipcode, birthdate)`

![img](https://oss.javaguide.cn/github/javaguide/database/mysql/index-condition-pushdown-graphic-illustration.png)

### [尽可能的考虑建立联合索引而不是单列索引](https://javaguide.cn/database/mysql/mysql-index.html#尽可能的考虑建立联合索引而不是单列索引)

因为索引是需要占用磁盘空间的，可以简单理解为每个索引都对应着一颗 B+树

### [注意避免冗余索引](https://javaguide.cn/database/mysql/mysql-index.html#注意避免冗余索引)

冗余索引指的是索引的功能相同，能够命中索引(a, b)就肯定能命中索引(a) ，那么索引(a)就是冗余索引。

### 索引失效情况

- 在索引列上进行计算、函数、类型转换等操作;

- 以 % 开头的 LIKE 查询比如 `LIKE '%abc';`;

- 创建了组合索引，但查询条件未遵守最左匹配原则;

- 查询条件中使用 OR，且 OR 的前后条件中有一个列没有索引，涉及的索引都不会被使用到;
- IN 的取值范围较大时会导致索引失效，走全表扫描(NOT IN 和 IN 的失效场景相同

- 发生[隐式转换l);当操作符**左右两边的数据类型不一致**时，会发生**隐式转换**

#### 可以使用 `EXPLAIN` 命令来分析 SQL 的 **执行计划** ，这样就知道语句是否命中索引了



## 三大日记

### redo log

是`InnoDB`存储引擎独有的，它让`MySQL`拥有了崩溃恢复能力

`MySQL` 中数据是以页为单位，你查询一条记录，会从硬盘把一页的数据加载出来，加载出来的数据叫数据页，会放入到 `Buffer Pool` 中。

后续的查询都是先从 `Buffer Pool` 中找，没有命中再去硬盘加载，减少硬盘 `IO` 开销，提升性能。

更新表数据的时候，也是如此，发现 `Buffer Pool` 里存在要更新的数据，就直接在 `Buffer Pool` 里更新。

然后会把“在某个数据页上做了什么修改”记录到重做日志缓存（`redo log buffer`）里，接着刷盘到 `redo log` 文件里

**每条 redo 记录由“表空间号+数据页号+偏移量+修改数据长度+具体修改的数据”组成**

#### 刷盘时机

- 事务提交（可以通过`innodb_flush_log_at_trx_commit`参数控制）
- log buffer 空间不足时：log buffer 中缓存用了一半
- 事务日志缓冲区满：InnoDB 使用一个事务日志缓冲区（transaction log buffer）来暂时存储事务的重做日志条目。当缓冲区满时，会触发日志的刷新，将日志写入磁盘
- Checkpoint（检查点）：InnoDB 定期会执行检查点操作，将内存中的脏数据（已修改但尚未写入磁盘的数据）刷新到磁盘，并且会将相应的重做日志一同刷新，以确保数据的一致性
- 后台刷新线程：InnoDB 启动了一个后台线程，负责周期性（每隔 1 秒）地将脏页（已修改但尚未写入磁盘的数据页）刷新到磁盘，并将相关的重做日志一同刷新
- 正常关闭服务器：MySQL 关闭的时候，redo log 都会刷入到磁盘里去

##### `innodb_flush_log_at_trx_commit` 的值有 3 种，也就是共有 3 种刷盘策略

- 0：每次事务提交时不进行刷盘操作。性能高但不安全
- 1：次事务提交时都将进行刷盘操作。 性能低但安全
- 2：**每次事务提交时都只把 log buffer 里的 redo log 内容写入 page cache（**文件系统缓存）。page cache 是专门用来缓存文件的，这里被缓存的文件就是 redo log 文件。这种方式的性能和安全性都介于前两者中间

![img](https://oss.javaguide.cn/github/javaguide/04.png)

#### 不同刷盘策略的流程图

![img](https://oss.javaguide.cn/github/javaguide/06.png)

![img](https://oss.javaguide.cn/github/javaguide/07.png)

 只要事务提交成功，`redo log`记录就一定在硬盘里，不会有任何数据丢失

![img](https://oss.javaguide.cn/github/javaguide/09.png)

如果仅仅只是`MySQL`挂了不会有任何数据丢失，但是宕机可能会有`1`秒数据的丢失

### 日志文件组

硬盘上存储的 `redo log` 日志文件不只一个，而是以一个**日志文件组**的形式出现的

采用的是环形数组形式，从头开始写，写到末尾又回到头循环写

![img](https://oss.javaguide.cn/github/javaguide/10.png)

**日志文件组**中还有两个重要的属性，分别是 `write pos、checkpoint`

- **write pos** 是当前记录的位置，一边写一边后移
- **checkpoint** 是当前要擦除的位置，也是往后推移

每次刷盘 `redo log` 记录到**日志文件组**中，`write pos` 位置就会后移更新。

每次 `MySQL` 加载**日志文件组**恢复数据时，会清空加载过的 `redo log` 记录，并把 `checkpoint` 后移更新

![img](https://oss.javaguide.cn/github/javaguide/11.png)

#### 为什么要有redolog，直接数据页刷新存回硬盘不就行了吗

数据页大小是`16KB`，刷盘比较耗时，可能就修改了数据页里的几 `Byte` 数据，数据页刷盘是随机写，因为一个数据页对应的位置可能在硬盘文件的随机位置

如果是写 `redo log`，一行记录可能就占几十 `Byte`，只包含表空间号、数据页号、磁盘文件偏移
量、更新值，再加上是顺序写，所以刷盘速度很快

### binlog

`redo log` 它是物理日志，记录内容是“在某个数据页上做了什么修改”，属于 `InnoDB` 存储引擎

`binlog` 是逻辑日志，记录内容是语句的原始逻辑，类似于“给 ID=2 这一行的 c 字段加 1”，属于`MySQL Server` 层。只要发生了表数据更新，都会产生 `binlog` 日志

数据库的**数据备份、主备、主主、主从**都离不开`binlog`

![img](https://oss.javaguide.cn/github/javaguide/01-20220305234724956.png)

### [记录格式](https://javaguide.cn/database/mysql/mysql-logs.html#记录格式)

`binlog` 日志有三种格式，可以通过`binlog_format`参数指定。

- **statement**
- **row**
- **mixed**

指定`statement`，记录的内容是`SQL`语句原文，比如执行一条

`update T set update_time=now() where id=1`，记录的内容如下

![img](https://oss.javaguide.cn/github/javaguide/02-20220305234738688.png)

但是有个问题，`update_time=now()`这里会获取当前系统时间，直接执行会导致与原库的数据不一致

为了解决这种问题，我们需要指定为`row`，记录的内容不再是简单的`SQL`语句了，还包含操作的具体数据，记录内容如下

![img](https://oss.javaguide.cn/github/javaguide/03-20220305234742460.png)

`row`格式记录的内容看不到详细信息，要通过`mysqlbinlog`工具解析出来

通常情况下都是指定为`row`

但是这种格式，需要更大的容量来记录，比较占用空间，恢复与同步时会更消耗`IO`资源，影响执行速度

有了一种折中的方案，**指定为`mixed`**，记录的内容是前两者的混合。

`MySQL`会判断这条`SQL`语句是否可能引起数据不一致，如果是，就用`row`格式，否则就用`statement`格式

#### 刷盘时机

事务执行过程中，先把日志写到`binlog cache`，事务提交的时候，再把`binlog cache`写到`binlog`文件中

因为一个事务的`binlog`不能被拆开，无论这个事务多大，也要确保一次性写入，所以系统会给每个线程分配一个块内存作为`binlog cache`

![img](https://oss.javaguide.cn/github/javaguide/04-20220305234747840.png)

write`和`fsync`的时机，可以由参数`sync_binlog`控制，默认是`1，每次提交事务都会执行`fsync`

为`0`的时候，表示每次提交事务都只`write`，由系统自行判断什么时候执行`fsync`

最后还有一种折中方式，可以设置为`N(N>1)`，表示每次提交事务都`write`，但累积`N`个事务后才`fsync`

### 两阶段提交

`binlog`只有在提交事务时才写入，因为是一个完整的事务

而且binlog是serve层，redolog是ionodb层

<img src="https://oss.javaguide.cn/github/javaguide/01-20220305234816065.png" alt="img" style="zoom: 67%;" />

假设执行过程中写完`redo log`日志后，`binlog`日志写期间发生了异常，会发生数据库的主从数据不一致

为了解决两份日志之间的逻辑一致问题，`InnoDB`存储引擎使用**两阶段提交**方案

将`redo log`的写入拆成了两个步骤`prepare`和`commit`，这就是**两阶段提交**

<img src="https://oss.javaguide.cn/github/javaguide/04-20220305234956774.png" alt="img" style="zoom:67%;" />

此时发生异常没有binlog，就会发生回滚

![img](https://oss.javaguide.cn/github/javaguide/05-20220305234937243.png)

### undolog

在 MySQL 中，恢复机制是通过 **回滚日志（undo log）** 实现的，并且，回滚日志会先于数据持久化到磁盘上

并且`MVCC` 的实现依赖于：**隐藏字段、Read View、undo log**。

### 总结

MySQL InnoDB 引擎使用 **redo log(重做日志)** 保证事务的**持久性**，使用 **undo log(回滚日志)** 来保证事务的**原子性**。

`MySQL`数据库的**数据备份、主备、主主、主从**都离不开`binlog`，需要依靠`binlog`来同步数据，保证数据一致性。

### MVCC实现

**隐藏字段、Read View、undo log**

<img src="https://cdn.xiaolincoding.com//mysql/other/f595d13450878acd04affa82731f76c5.png" alt="图片" style="zoom:50%;" />

当读取记录时，若该记录被其他事务占用或当前版本对该事务不可见，则可以通过 `undo log` 读取之前的版本数据，以此实现非锁定读

readview

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/readview%E7%BB%93%E6%9E%84.drawio.png" alt="img" style="zoom:50%;" />

- 如果记录的 trx_id **在** `m_ids` 列表中，表示生成该版本记录的活跃事务依然活跃着（还没提交事务），所以该版本的记录对当前事务**不可见**。
- 如果记录的 trx_id **不在** `m_ids`列表中，表示生成该版本记录的活跃事务已经被提交，所以该版本的记录对当前事务**可见**。

## [RC 和 RR 隔离级别下 MVCC 的差异](#rc-和-rr-隔离级别下-mvcc-的差异)

在事务隔离级别 `RC` 和 `RR` （InnoDB 存储引擎的默认事务隔离级别）下，`InnoDB` 存储引擎使用 `MVCC`（非锁定一致性读），但它们生成 `Read View` 的时机却不同

- 在 RC 隔离级别下的 **`每次select`** 查询前都生成一个`Read View` (m_ids 列表)

- 在 RR 隔离级别下只在事务开始后 **`第一次select`** 数据前生成一个`Read View`（m_ids 列表

  可重复读的情况下m_ids是不会变的，只有提交后，再次开启事务才会。

## [MVCC➕Next-key-Lock 防止幻读](https://javaguide.cn/database/mysql/innodb-implementation-of-mvcc.html#mvcc➕next-key-lock-防止幻读)

MVCC就是版本控制，快照读的意思。

1. **执行普通 `select`，此时会以 `MVCC` 快照读的方式读取数据**
2. **执行 select...for update/lock in share mode、insert、update、delete 等当前读**

### 数据库的三范式

- 第一范式：每个字段的原子性，每个字段中的值是不可再分的
- 第二范式：每个非主键字段完全依赖于主键字段，而不是依赖主键的一部分
- 第三范式：每个非主键字段不能依赖于其他非主键字段

### SQL优化技巧

1.减少select*的使用，这个语句不会走覆盖索引

2.用union all 代替 union 因为这样不会执行去重的操作，提高效率

3.小表驱动大表（没了解过）

4.一批数据的插入，可以用批量操作，而不是逐条插入

5.多用limit，返回我们只需要的数据

6.连接查询代替子查询，因为子查询会导致创建临时表，损耗性能

7.sql语句在进行一些范围查询的时候，可以先缩减数据的查找量

8.索引优化，expalin来判断sql语句是否有走索引查询

### 百万数据查询优化

1.使用索引

2.分页查询，每次返回部分数据而不是一次性查询全部数据，减少数据库的负载和网络的开销

3.使用缓存，对于一些热点数据的查询可以缓存到内存当中，减少对数据库的查询次数

4.使用合适的数据结构类型，避免过大的数据类型，减少对存储空间和索引的占用