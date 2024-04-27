### NULL 和 '' 的区别是什么？

- `NULL` 代表一个不确定的值,就算是两个 `NULL`,它俩也不一定相等。例如，`SELECT NULL=NULL`的结果为 false，但是在我们使用`DISTINCT`,`GROUP BY`,`ORDER BY`时,`NULL`又被认为是相等的。
- `''`的长度是 0，是不占用空间的，而`NULL` 是需要占用空间的。
- `NULL` 会影响聚合函数的结果。例如，`SUM`、`AVG`、`MIN`、`MAX` 等聚合函数会忽略 `NULL` 值。 `COUNT` 的处理方式取决于参数的类型。如果参数是 `*`(`COUNT(*)`)，则会统计所有的记录数，包括 `NULL` 值；如果参数是某个字段名(`COUNT(列名)`)，则会忽略 `NULL` 值，只统计非空值的个数。
- 查询 `NULL` 值时，必须使用 `IS NULL` 或 `IS NOT NULLl` 来判断，而不能使用 =、!=、 <、> 之类的比较运算符。而`''`是可以使用这些比较运算符的。

### [Boolean 类型如何表示？](#boolean-类型如何表示)

MySQL 中没有专门的布尔类型，而是用 TINYINT(1) 类型来表示布尔值。TINYINT(1) 类型可以存储 0 或 1，分别对应 false 或 true。

MySQL基础架构

![img](https://oss.javaguide.cn/javaguide/13526879-3037b144ed09eb88.png)

缓存器在8.0后移除了

MySQL 当前默认的存储引擎是 InnoDB

MySQL 存储引擎采用的是 **插件式架构** ，支持多种存储引擎，我们甚至可以为不同的数据库表设置不同的存储引擎以适应不同场景的需要。**存储引擎是基于表的，而不是数据库。**

### [MyISAM 和 InnoDB 有什么区别？](https://javaguide.cn/database/mysql/mysql-questions-01.html#myisam-和-innodb-有什么区别)

MySQL 5.5 之前，MyISAM 引擎是 MySQL 的默认存储引擎，但是，MyISAM 不支持事务和行级锁，而且最大的缺陷就是崩溃后无法安全恢复。

**1.是否支持行级锁**

MyISAM 只有表级锁(table-level locking)，而 InnoDB 支持行级锁(row-level locking)和表级锁,默认为行级锁。

**2.是否支持事务**

MyISAM 不提供事务支持。

InnoDB 提供事务支持，实现了 SQL 标准定义了四个隔离级别，具有提交(commit)和回滚(rollback)事务的能力。并且，InnoDB 默认使用的 REPEATABLE-READ（可重读）隔离级别是可以解决幻读问题发生的（**基于 MVCC 和 Next-Key Lock**）。

**3.是否支持外键**

MyISAM 不支持，而 InnoDB 支持。

外键对于维护数据一致性非常有帮助，但是对性能有一定的损耗。因此，通常情况下，我们是不建议在实际生产项目中使用外键的，在业务代码中进行约束即可！

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

在 InnoDB 默认的隔离级别 REPEATABLE-READ 下，行锁默认使用的是 Next-Key Lock。**但是，如果操作的索引是唯一索引或主键，InnoDB 会对 Next-Key Lock 进行优化，将其降级为 Record Lock，即仅锁住索引本身，而不是范围。**

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