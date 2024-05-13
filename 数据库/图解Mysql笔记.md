## Mysql执行流程

![查询语句执行流程](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/sql%E6%89%A7%E8%A1%8C%E8%BF%87%E7%A8%8B/mysql%E6%9F%A5%E8%AF%A2%E6%B5%81%E7%A8%8B.png)

**Server 层负责建立连接、分析和执行 SQL**，**存储引擎层负责数据的存储和提取**。

查询过程

客户端发送一条查询请求，服务层进行分析，解析，预处理器，然后由优化器选择执行计划

执行器在调用存储引擎的 API 来执行查询，将结果返回给客户端，同时缓存查询结果

### 执行计划的内容

1. **查询的执行顺序**：执行计划会告诉 MySQL 按照何种顺序执行查询，包括连接表的顺序、子查询的执行顺序等

   就是先where，后select种

2. **使用的索引**：执行计划会指出哪些索引被用到了，以及如何使用这些索引来加速查询。

3. **扫描方法**：执行计划会指出 MySQL 使用的是全表扫描还是索引扫描，以及扫描的方向（顺序扫描或逆序扫描）。

4. **连接方法**：如果查询涉及多个表，执行计划会指出 MySQL 使用的是哪种连接方法（例如，嵌套循环连接、哈希连接、排序合并连接等）。

5. **是否需要临时表**：如果查询需要排序或者分组，执行计划会指出是否需要创建临时表。

6. **预计的行数**：执行计划会估算每个步骤中涉及的行数，以便评估执行计划的成本。

### explain的列

![image-20240513094416896](C:\Users\86147\AppData\Roaming\Typora\typora-user-images\image-20240513094416896.png)

id

- 查询执行顺序
- id 值相同时表示从上向下执行
- id 值相同被视为一组
- 如果是子查询，id 值会递增，id 值越高，优先级越高
- id为NULL最后执行

select_type

- simple：表示查询中不包含子查询或者 union
- primary：当查询中包含任何复杂的子部分，最外层的查询被标记成 primary
- derived：在 from 的列表中包含的子查询被标记成 derived
- subquery：在 select 或 where 列表中包含了子查询，则子查询被标记成 subquery

type

显示连接使用了何种类型。从最好到最差的连接类型为 system > const > eq_reg > ref > range > index > ALL。
一般来说，得保证查询达到range级别，最好达到ref

- system：表中只有一行数据。属于 const 的特例。如果物理表中就一行数据为 ALL

- const ：查询结果最多有一个匹配行

  ~~~sql
  explain select * from (select * from film where id = 1) tmp;
  ~~~

  ref：比 eq_ref，不使用唯一索引，而是使用普通索引或者唯一性索引的部分前缀，索引要和某个值相比较，可能会找到多个符合条件的行

  range：把这个列当作条件只检索其中一个范围。常见 where 从句中出现 between、<、>、>=、in 等。主要应用在具有索引的列中

  index：扫描全索引就能拿到结果，一般是扫描某个二级索引，这种扫描不会从索引树根节点开始快速查找，而是直接对二级索引的叶子节点遍历和扫描，速度还是比较慢的，这种查询一般为使用覆盖索引，二级索引一般比较小，所以这种通常比ALL快一些

  ALL：即全表扫描，扫描你的聚簇索引的所有叶子节点。通常情况下这需要增加索引来进行优化了

key实际使用的索引。如果为 NULL，则没有使用索引

### Mysql的存储

存储的目录文件有 db.opt，用来存储当前数据库的默认字符集和字符校验规则。t_order.frm **表结构**和t_order.ibd，**表数据**

#### 表空间的文件结构

**表空间由段（segment）、区（extent）、页（page）、行（row）组成**

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E8%A1%A8%E7%A9%BA%E9%97%B4%E7%BB%93%E6%9E%84.drawio.png)

**InnoDB 的数据是按「页」为单位来读写的**，默认页大小为16kb。

**在表中数据量大的时候，为某个索引分配空间的时候就不再按照页为单位分配了，而是按照区（extent）为单位分配。每个区的大小为 1MB，对于 16KB 的页来说，连续的 64 个页会被划为一个区，这样就使得链表中相邻的页的物理位置也相邻，就能使用顺序 I/O 了**。

段一般分为数据段、索引段和回滚段等。

- 索引段：存放 B + 树的非叶子节点的区的集合；
- 数据段：存放 B + 树的叶子节点的区的集合；
- 回滚段：存放的是回滚数据的区的集合，之前讲[事务隔离 (opens new window)](https://xiaolincoding.com/mysql/transaction/mvcc.html)的时候就介绍到了 MVCC 利用了回滚段实现了多版本查询数据。

## InnoDB 的主要行格式

Compact 行格式

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/COMPACT.drawio.png)

##### 变长字段长度列表

变长字段的真实数据占用的字节数占用的内存大小要存到「变长字段长度列表」里面，并且字节数按列的顺序逆序存放。逆序存放的原因是可以提高cpuCahce的命中率。并且变长字段列表不是必有的。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/%E5%8F%98%E9%95%BF%E5%AD%97%E6%AE%B5%E9%95%BF%E5%BA%A6%E5%88%97%E8%A1%A81.png)

#####  NULL 值列表

值为 NULL 的列存储到 NULL值列表中，NULL 值列表也不是必须的。

如果存在允许 NULL 值的列，则每个列对应一个二进制位（bit），二进制位按照列的顺序逆序排列。

- 二进制位的值为`1`时，代表该列的值为NULL。
- 二进制位的值为`0`时，代表该列的值不为NULL。

NULL 值列表必须用整数个字节的位表示（1字节8位

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/row_format/null%E5%80%BC%E5%88%97%E8%A1%A82.png)

#####  记录头信息

delete_mask ：标识此条数据是否被删除。next_record：下一条记录的位置。record_type：表示当前记录的类型

##### 三个隐藏字段

- row_id

如果我们建表的时候指定了主键或者唯一约束列，那么就没有 row_id 隐藏字段了。如果既没有指定主键，又没有唯一约束，那么 InnoDB 就会为记录添加 row_id 隐藏字段。row_id不是必需的，占用 6 个字节。

- trx_id

事务id，表示这个数据是由哪个事务生成的。 trx_id是必需的，占用 6 个字节。

- roll_pointer

这条记录上一个版本的指针。roll_pointer 是必需的，占用 7 个字节。

**MySQL 规定除了 TEXT、BLOBs 这种大对象类型之外，其他所有的列（不包括隐藏列和记录头信息）占用的字节长度加起来不能超过 65535 个字节**（2的16次方-1）。

Varchar（n）中的n表示这个字段能有多少个字符，不同规则下字符代表的字节数不一致，如utf8就是三个字节

## 索引

分类--由4个角度来分类

![image-20240327184831178](C:\Users\86147\AppData\Roaming\Typora\typora-user-images\image-20240327184831178.png)

创建表时默认主键作为索引键，其次为非空的唯一键，两种都没有就自动生成一个隐式自增 id 。

**创建的主键索引和二级索引默认使用的是 B+Tree 索引**。每个节点里的数据是**按主键顺序存放**的

这张表以id为主键

![主键索引 B+Tree](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E7%B4%A2%E5%BC%95/btree.drawio.png)

![image-20240327185308472](C:\Users\86147\AppData\Roaming\Typora\typora-user-images\image-20240327185308472.png)

二级索引只存储主键值，不存储实际数据，如果要查实际数据还要再通过主键索引查值

**这个过程叫「回表」，也就是说要查两个 B+Tree 才能查到数据**。**在二级索引的 B+Tree 就能查询到结果的过程就叫作「覆盖索引」，也就是只需要查一个 B+Tree 就能找到数据**。

![二级索引 B+Tree](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E7%B4%A2%E5%BC%95/%E4%BA%8C%E7%BA%A7%E7%B4%A2%E5%BC%95btree.drawio.png)

### 为什么选择B+树作为索引的数据结构

***1、B+Tree vs B Tree***

B+Tree 只在叶子节点存储数据，而 B 树 的非叶子节点也要存储数据，所以 B+Tree 的单个节点的数据量更小，在相同的磁盘 I/O 次数下，就能查询更多的节点。

B+Tree 叶子节点采用的是双链表连接，适合 MySQL 中常见的基于范围的顺序查找，而 B 树无法做到这一点。

***2、B+Tree vs 二叉树***

对于有 N 个叶子节点的 B+Tree，其搜索复杂度为`O(logdN)`，其中 d 表示节点允许的最大子节点个数为 d 个。即使数据达到千万级别，B+Tree的高度依然为3-4层左右

而二叉树的每个父节点的儿子节点个数只能是 2 个，意味着其搜索复杂度为 `O(logN)`

***3、B+Tree vs Hash***

Hash 在做等值查询的时候效率贼快，搜索复杂度为 O(1)。但是 Hash 表不适合做范围查询，它更适合做等值的查询，这也是 B+Tree 索引要比 Hash 表索引有着更广泛的适用场景的原因。

### 索引的分类

按字段个数分类--单列索引和联合索引

联合索引

```sql
CREATE INDEX index_product_no_name ON product(product_no, name);
```

使用联合索引时，存在**最左匹配原则**。利用索引的前提是索引里的 key 是有序的**。**

**合索引的最左匹配原则，在遇到范围查询（如 >、<）的时候，就会停止匹配，也就是范围查询的字段可以用到联合索引，但是在范围查询字段的后面的字段无法用到联合索引。注意，对于 >=、<=、BETWEEN、like 前缀匹配的范围查询，并不会停止匹配，**

![联合索引](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/%E7%B4%A2%E5%BC%95/%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95.drawio.png)

### 索引的缺点

占据物理空间，创建和维护时间耗费，增删改效率降低，因为要动态维护索引

数据扫描方式**执行效率从低到高的顺序为**：

- All（全表扫描）；
- index（全索引扫描）；
- range（索引范围扫描）；
- ref（非唯一索引扫描）；
- eq_ref（唯一索引扫描）；
- const（结果只有一条的主键或唯一索引扫描）。

### InnoDB数据

默认数据页大小为16k，所以数据库每次读写都是以16KB为单位的

数据页结构

<img src="https://cdn.xiaolincoding.com//mysql/other/243b1466779a9e107ae3ef0155604a17.png" alt="图片" style="zoom: 50%;" />

![图片](https://cdn.xiaolincoding.com//mysql/other/557d17e05ce90f18591c2305871af665.png)

采用链表的结构是让数据页之间不需要是物理上的连续的，而是逻辑上的连续。**数据页中的记录按照「主键」顺序组成单向链表**。

页目录的创建

<img src="https://cdn.xiaolincoding.com//mysql/other/261011d237bec993821aa198b97ae8ce.png" alt="图片" style="zoom:50%;" />

**每个槽相当于指针指向了不同组的最后一个记录**。**页目录就是由多个槽组成的，槽相当于分组记录的索引**。可以通过二分法快速定位，因为记录是按主键大小排序的。

**InnoDB 采用了 B+ 树作为索引**来定位我们的数据页。**每个节点都是一个数据页**，但只有叶子节点是存放数据的，非叶子节点仅存放目录项作为索引。

![图片](https://cdn.xiaolincoding.com//mysql/other/7c635d682bd3cdc421bb9eea33a5a413.png)

### 聚簇索引和二级索引

因为表的数据都是存放在聚簇索引的叶子节点里，所以一定有一个聚簇（主键）索引，且由于数据在物理上只会保存一份，所以聚簇索引只能有一个。

**索引覆盖**指的是用二级索引搜索b+树时，找到主键值即可返回，而无需再回表。

为什么采用B+树作为索引

二叉查找树弊端：**当每次插入的元素都是二叉查找树中最大的元素，二叉查找树就会退化成了一条链表，查找数据的时间复杂度变成了 O(n**

自平衡二叉树弊端：**每个节点的左子树和右子树的高度差不能超过 1**。红黑树也是其中一种。

插入元素变多时导致树的高度变大，**意味着磁盘 I/O 操作次数多，会影响整体数据查询的效率**

B树弊端：每个节点都包含数据（索引+记录），用户的记录数据的大小很有可能远远超过了索引数据，**这就需要花费更多的磁盘 I/O 操作次数来读到「有用的索引数据」。就是要加载更多的数据到内存里面**



**B+ 树对比B树的插入和删除效率更高**，因为只涉及到叶子节点的插入和删除，比较少树的变形，但B树的变形极为复杂。

 **B+ 树所有叶子节点间还有一个链表进行连接，这种设计对范围查找非常有帮助**比如说我们想知道 12 月 1 日和 12 月 12 日之间的订单，这个时候可以先查找到 12 月 1 日所在的叶子节点，然后利用链表向右遍历，直到找到 12 月12 日的节点，这样就不需要从根节点查询了，进一步节省查询需要的时间。

**索引失效**

1.当我们使用左或者左右模糊匹配的时候，也就是 `like %xx` 或者 `like %xx%`，都会失效，会走全表扫描。**因为索引 B+ 树是按照「索引值」有序排列存储的，只能根据前缀进行比较。**

但是如果数据库表中的字段只有主键+二级索引，那么即使使用了左模糊匹配，也不会走全表扫描（type=all），而是走全扫描二级索引树(type=index)。

2.对索引使用函数

```mysql
// name 为二级索引
select * from t_user where length(name)=6;
```

3.对索引进行表达式计算

```mysql
explain select * from t_user where id + 1 = 10
```

4.对索引隐式类型转换

phone 为varchar类型

```text
select * from t_user where phone = 1300000001;
```

- MySQL 在遇到字符串和数字比较的时候，会自动把字符串转为数字，然后再进行比较。如果字符串是索引列，而条件语句中的输入参数是数字的话，那么索引列会发生隐式类型转换，由于隐式类型转换是通过 CAST 函数实现的，等同于对索引列使用了函数，所以就会导致索引失效。

5.联合索引非最左匹配

**多个普通字段组合在一起创建的索引就叫做联合索引**，多个主键索引也就是联合索引

如果创建了一个 `(a, b, c)` 联合索引，匹配上联合索引：

- where a=1；
- where a=1 and b=2 and c=3；
- where a=1 and b=2；

6.在 WHERE 子句中，如果在 OR 前的条件列是索引列，而在 OR 后的条件列不是索引列，那么索引会失效。

```text
select * from t_user where id = 1 or age = 18;
```

### count性能

<img src="https://cdn.xiaolincoding.com//mysql/other/af711033aa3423330d3a4bc6baeb9532.png" alt="图片" style="zoom:50%;" />

```sql
select count(name) from t_order;
```

这条语句是统计「 t_order 表中，name 字段不为 NULL 的记录」有多少个

```sql
select count(1) from t_order;
```

如果表里只有主键索引，没有二级索引时。InnoDB 循环遍历聚簇索引（主键索引）,**但是不会读取记录中的任何字段的值**。如果表里有二级索引时，InnoDB 循环遍历的对象就二级索引了。

**count(\*) 执行过程跟 count(1) 执行过程基本一样的**

count(\*)和count(1)都一样是查询表中所有的行数

如果你非要统计表中该字段不为 NULL 的记录个数，建议给这个字段建立一个二级索引。

## 事务篇（Transaction）

​	4个特性：原子性，一致性，隔离性，持久性

严重性从高到低

### 脏读

**如果一个事务「读到」了另一个「未提交事务修改过的数据」，就意味着发生了「脏读」现象。**

### 不可重复读

**在一个事务内多次读取同一个数据，如果出现前后两次读到的数据不一样的情况，就意味着发生了「不可重复读」现象。**

### 幻读

**在一个事务内多次查询某个符合查询条件的「记录数量」，如果出现前后两次查询到的记录数量不一样的情况，就意味着发生了「幻读」现象。**

有四个隔离级别来规避这些现象

隔离水平从低到高：

- **读未提交（\*read uncommitted\*）**，指一个事务还没提交时，它做的变更就能被其他事务看到；
- **读提交（\*read committed\*）**，指一个事务提交之后，它做的变更才能被其他事务看到；
- **可重复读（\*repeatable read\*）**，指一个事务执行过程中看到的数据，一直跟这个事务启动时看到的数据是一致的，**MySQL InnoDB 引擎的默认隔离级别**；
- **串行化（\*serializable\* ）**；会对记录加上读写锁，在多个事务对这条记录进行读写操作时，如果发生了读写冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行

<img src="https://cdn.xiaolincoding.com//mysql/other/4e98ea2e60923b969790898565b4d643.png" alt="图片" style="zoom: 50%;" />

**MySQL InnoDB 引擎的默认隔离级别虽然是「可重复读」，但是它很大程度上避免幻读现象**

这种隔离级别通过Read View来实现

解决方案有两种

针对**快照读**（普通 select 语句），是**通过 MVCC 方式解决了幻读**

针对**当前读**（select ... for update 等语句），是**通过 next-key lock（记录锁+间隙锁）方式解决了幻读**

## Read View 在 MVCC 里如何工作的？

Read View字段

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/readview%E7%BB%93%E6%9E%84.drawio.png" alt="img" style="zoom:50%;" />

聚簇索引记录中的两个隐藏列

<img src="https://cdn.xiaolincoding.com//mysql/other/f595d13450878acd04affa82731f76c5.png" alt="图片" style="zoom:50%;" />

- trx_id，当一个事务对某条聚簇索引记录进行改动时，就会**把该事务的事务 id 记录在 trx_id 隐藏列里**；
- roll_pointer，每次对某条聚簇索引记录进行改动时，都会把旧版本的记录写入到 undo 日志中，然后**这个隐藏列是个指针，指向每一个旧版本记录**，于是就可以通过它找到修改前的记录。

在创建 Read View 后，我们可以将记录中的 trx_id 划分这三种情况：

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/ReadView.drawio.png" alt="img" style="zoom:50%;" />

- 如果记录的 trx_id 值小于 Read View 中的 `min_trx_id` 值，表示这个版本的记录是在创建 **Read View 前已经提交的事务生成的，所以该版本的记录对当前事务可见**。
- 如果记录的 trx_id 值大于等于 Read View 中的 `max_trx_id` 值，表示这个版本的记录是在创建 Read View **后**才启动的事务生成的，所以该版本的记录对当前事务**不可见**。
- 如果记录的 trx_id 值在 Read View 的min_trx_id和max_trx_id之间，需要判断 trx_id 是否在 m_ids 列表中：
  - 如果记录的 trx_id **在** `m_ids` 列表中，表示生成该版本记录的活跃事务依然活跃着（还没提交事务），所以该版本的记录对当前事务**不可见**。
  - 如果记录的 trx_id **不在** `m_ids`列表中，表示生成该版本记录的活跃事务已经被提交，所以该版本的记录对当前事务**可见**。

**这种通过「版本链」来控制并发事务访问同一个记录时的行为就叫 MVCC（多版本并发控制）。**

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB/%E4%BA%8B%E5%8A%A1ab%E7%9A%84%E8%A7%86%E5%9B%BE2.png" alt="img" style="zoom:33%;" />

如果事务的记录不可见，则会通过undo来回溯到可见的事务id并读取

**可重复读隔离级别是启动事务时生成一个 Read View，然后整个事务期间都在用这个 Read View**。就是无论在m_ids中的事务是否提交了，都是会根据启动事务时的Read View去判断，所以即使m_ids提交了事务，Read View还是会判断不可见

**读提交隔离级别是在每次读取数据时，都会生成一个新的 Read View**。与重复读则相反。

### 可重复读隔离级别解决幻读的两种方法

针对**快照读**（==普通 select 语句==），是**通过 MVCC 方式解决了幻读**

针对**当前读**（select ... for update 等语句），是**通过 next-key lock（记录锁+间隙锁）方式解决了幻读**，别的事务无法插入

## 锁篇

根据加锁的范围，可以分为**全局锁、表级锁和行锁**三类。

全局锁主要应用于做**全库逻辑备份**，这样在备份数据库期间，不会因为数据或表结构的更新，而出现备份文件的数据与预期的不一样。

### 表级锁

MySQL 里面表级别的锁有这几种：

- 表锁；

  表锁除了会限制别的线程的读写外，也会限制本线程接下来的读写操作。

- 元数据锁（MDL）;

  我们不需要显示的使用 MDL，因为当我们对数据库表进行操作时，会自动给这个表加上 MDL：

  - 对一张表进行 CRUD 操作时，加的是 **MDL 读锁**；
  - 对一张表做结构变更操作的时候，加的是 **MDL 写锁**；

- 意向锁；

  - 在使用 InnoDB 引擎的表里对某些记录加上「共享锁」之前，需要先在表级别加上一个「意向共享锁」；
  - 在使用 InnoDB 引擎的表里对某些纪录加上「独占锁」之前，需要先在表级别加上一个「意向独占锁」；
  - 如果没有「意向锁」，那么加「独占表锁」时，就需要遍历表里所有记录，查看是否有记录存在独占锁，这样效率会很慢。
  - **意向锁的目的是为了快速判断表里是否有记录被加锁**。

- AUTO-INC 锁；

  AUTO-INC 锁是特殊的表锁机制，锁**不是再一个事务提交后才释放，而是再执行完插入语句后就会立即释放**。

### 行级锁

有三种类别

- Record Lock，记录锁，也就是仅仅把一条记录锁上；

  有s锁（共享锁）和x锁（独占锁）之分

- Gap Lock，间隙锁，锁定一个范围，但是不包含记录本身；

  **间隙锁之间是兼容的，即两个事务可以同时持有包含共同间隙范围的间隙锁**

- Next-Key Lock：Record Lock + Gap Lock 的组合，锁定一个范围，并且锁定记录本身。

  **next-key lock 是包含间隙锁+记录锁的，如果一个事务获取了 X 型的 next-key lock，那么另外一个事务在获取相同范围的 X 型的 next-key lock 时，是会被阻塞的**。

###  插入意向锁

一个事务在插入一条记录的时候，如果插入记录有间隙锁阻塞，会生成插入意向锁，表明有事务想在某个区间插入新记录，但是现在处于等待状态。它并**不是意向锁，它是一种特殊的间隙锁，属于行级别锁**。

## MySql 加锁

### 添加行级锁

**加锁的对象是索引，加锁的基本单位是 next-key lock**。但是，next-key lock 在一些场景下会退化成记录锁或间隙锁。

总结就是**在能使用记录锁或者间隙锁就能避免幻读现象的场景下， next-key lock 就会退化成记录锁或间隙锁**。

唯一索引等值查询时

- 当查询的记录是「存在」的，在索引树上定位到这一条记录后，将该记录的索引中的 next-key lock 会**退化成「记录锁」**。
- 当查询的记录是「不存在」的，在索引树找到第一条大于该查询记录的记录后，将该记录的索引中的 next-key lock 会**退化成「间隙锁」**。

唯一索引范围查询时

- 情况一：针对「大于等于」的范围查询，因为存在等值查询的条件，那么如果等值查询的记录是存在于表中，那么该记录的索引中的 next-key 锁会**退化成记录锁**。如果是大于则是间隙锁
- 情况二：针对「小于或者小于等于」的范围查询，要看条件值的记录是否存在于表中：
  - 当条件值的记录不在表中，那么不管是「小于」还是「小于等于」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。
  - 当条件值的记录在表中，如果是「小于」条件的范围查询，**扫描到终止范围查询的记录时，该记录的索引的 next-key 锁会退化成间隙锁**，其他扫描到的记录，都是在这些记录的索引上加 next-key 锁；如果「小于等于」条件的范围查询，扫描到终止范围查询的记录时，该记录的索引 next-key 锁不会退化成间隙锁。其他扫描到的记录，都是在这些记录的索引上加 next-key 锁。

## MySQL日志

更新语句的流程会涉及到 undo log（回滚日志）、redo log（重做日志） 、binlog （归档日志）

- **undo log（回滚日志）**：是 Innodb 存储引擎层生成的日志，实现了事务中的**原子性**，主要**用于事务回滚和 MVCC**（版本控制）。（undolog的两大作用）
- **redo log（重做日志）**：是 Innodb 存储引擎层生成的日志，实现了事务中的**持久性**，主要**用于掉电等故障恢复**；
- **binlog （归档日志）**：是 Server 层生成的日志，主要**用于数据备份和主从复制**；

我们在执行执行一条“增删改”语句的时候，虽然没有输入 begin 开启事务和 commit 提交事务，但是 MySQL 会**隐式开启事务**来执行“增删改”语句的。

undo log 是一种用于撤销回退的日志。在事务没提交之前，MySQL 会先记录更新前的数据到 undo log 日志文件里面，当事务回滚时，可以利用 undo log 来进行回滚

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E5%9B%9E%E6%BB%9A%E4%BA%8B%E5%8A%A1.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="回滚事务" style="zoom:67%;" />

在发生回滚时，就读取 undo log 里的数据，然后做原先相反操作。

一条记录的每一次更新操作产生的 undo log 格式都有一个 roll_pointer 指针和一个 trx_id 事务id：

- 通过 trx_id 可以知道该记录是被哪个事务修改的；
- 通过 roll_pointer 指针可以将这些 undo log 串成一个链表，这个链表就被称为版本链；

版本链如图

![版本链](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%89%88%E6%9C%AC%E9%93%BE.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

**undo log 还有一个作用，通过 ReadView + undo log 实现 MVCC（多版本并发控制）**

对于「读提交」和「可重复读」隔离级别的事务来说，它们的快照读（普通 select 语句）是通过 Read View + undo log 来实现的，它们的区别在于创建 Read View 的时机不同：

读提交是每次select都会拍一张ReadView，而可重复读是启动一个事务后的ReadView，整个事务期间都用ReadView，如果当前记录trx_id不满足，就通过roll_pointer来回溯

## 数据库的Buffer Pool

Innodb 存储引擎设计了一个**缓冲池（Buffer Pool）**，来提高数据库的读写性能。

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/%E7%BC%93%E5%86%B2%E6%B1%A0.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="Buffer Poo" style="zoom:67%;" />

- 当读取数据时，如果数据存在于 Buffer Pool 中，客户端就会直接读取 Buffer Pool 中的数据，否则再去磁盘中读取。
- 当修改数据时，如果数据存在于 Buffer Pool 中，那直接修改 Buffer Pool 中数据所在的页，然后将其页设置为脏页（该页的内存数据和磁盘上的数据已经不一致），为了减少磁盘I/O，不会立即将脏页写入磁盘，后续由后台线程选择一个合适的时机将脏页写入到磁盘。

### Buffer Pool 缓存什么

在 MySQL 启动的时候，**InnoDB 会为 Buffer Pool 申请一片连续的内存空间，然后按照默认的`16KB`的大小划分出一个个的页， Buffer Pool 中的页就叫做缓存页**。

MySQL 刚启动的时候，你会观察到使用的虚拟内存空间很大，而使用到的物理内存空间却很小，这是因为只有这些虚拟内存被访问后，操作系统才会触发缺页中断，申请物理内存，接着将虚拟地址和物理地址建立映射关系。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/bufferpool%E5%86%85%E5%AE%B9.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

Undo 页是记录的是undolog

查询一条记录，会把整个数据页加载到Buffer Pool中，再通过页里的「页目录」去定位到某条具体的记录

## redo log

当有一条记录需要更新的时候，InnoDB 引擎就会先更新内存（同时标记为脏页），然后将本次对这个页的修改以 redo log 的形式记录下来，**这个时候更新就算完成了**。

后续，InnoDB 引擎会在适当的时候，由后台线程将缓存在 Buffer Pool 的脏页刷新到磁盘里，这就是 **WAL （Write-Ahead Logging）技术**。

**WAL 技术指的是， MySQL 的写操作并不是立刻写到磁盘上，而是先写日志，然后在合适的时间再写到磁盘上**。这个技术使**MySQL 的写操作从磁盘的「随机写」变成了「顺序写」**

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/wal.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="img" style="zoom: 50%;" />

redo log 是物理日志，记录了某个数据页做了什么修改，比如**对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新**，每当执行一个事务就会产生这样的一条或者多条物理日志。

在事务提交时，只要先将 redo log 持久化到磁盘即可。

### redo log 和 undo log 区别

- redo log 记录了此次事务「**完成后**」的数据状态，记录的是更新**之后**的值；
- undo log 记录了此次事务「**开始前**」的数据状态，记录的是更新**之前**的值；

 **redo log 保证了事务四大特性中的持久性**。

#### redo log 要写到磁盘，数据也要写磁盘，为什么要多此一举？

写入 redo log 的方式使用了追加操作， 所以磁盘操作是**顺序写**，而写入数据需要先找到写入位置，然后才写到磁盘，所以磁盘操作是**随机写**。

**顺序写的性能比随机写的性能高**

#### redo log的两个好处

- **实现事务的持久性，让 MySQL 有 crash-safe 的能力**，能够保证 MySQL 在任何时间段突然崩溃，重启后之前已提交的记录都不会丢失；
- **将写操作从「随机写」变成了「顺序写」**，提升 MySQL 写入磁盘的性能。

redo log 也有自己的缓存—— **redo log buffer**，每当产生一条 redo log 时，会先写入到 redo log buffer，后续在持久化到磁盘

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/redologbuf.webp" alt="事务恢复" style="zoom: 50%;" />

### redo log 什么时候刷盘？

- MySQL 正常关闭时；
- 当 redo log buffer 中记录的写入量大于 redo log buffer 内存空间（默认大小为16MB）的一半时，会触发落盘；
- InnoDB 的后台线程每隔 1 秒，将 redo log buffer 持久化到磁盘。
- 每次事务提交时都将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘（这个策略可由 innodb_flush_log_at_trx_commit 参数控制）。

#### innodb_flush_log_at_trx_commit 参数控制的是什么？

默认值为1

- 当设置该**参数为 0 时**，表示每次事务提交时 ，**将 redo log 留在 redo log buffer 中** 
- 当设置该**参数为 1 时**，表示每次事务提交时，**将缓存在 redo log buffer 里的 redo log 直接持久化到磁盘**
- 当设置该**参数为 2 时**，表示每次事务提交时，都只是缓存在 redo log buffer 里的 redo log **写到 redo log 文件，注意写入到「 redo log 文件」并不意味着写入到了磁盘**，因为操作系统的文件系统中有个 Page Cache（Page Cache 是专门用来缓存文件数据的，所以写入「 redo log文件」意味着写入到了操作系统的文件缓存。）

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/innodb_flush_log_at_trx_commit.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="img" style="zoom: 67%;" />

#### innodb_flush_log_at_trx_commit 为 0 和 2 的时候，什么时候才将 redo log 写入磁盘？

InnoDB 的后台线程每隔 1 秒：

- 针对参数 0 ：会把缓存在 redo log buffer 中的 redo log ，通过调用 `write()` 写到操作系统的 Page Cache，然后调用 `fsync()` 持久化到磁盘。**所以参数为 0 的策略，MySQL 进程的崩溃会导致上一秒钟所有事务数据的丢失**;
- 针对参数 2 ：调用 fsync，将缓存在操作系统中 Page Cache 里的 redo log 持久化到磁盘。**所以参数为 2 的策略，较取值为 0 情况下更安全，因为 MySQL 进程的崩溃并不会丢失数据，只有在操作系统崩溃或者系统断电的情况下，上一秒钟所有事务数据才可能丢失**。

### redo log 文件写满了怎么办？

默认情况下， InnoDB 存储引擎有 1 个重做日志文件组( redo log Group），「重做日志文件组」由有 2 个 redo log 文件组成，这两个 redo 日志的文件名叫 ：`ib_logfile0` 和 `ib_logfile1` 。

![重做日志文件组](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E9%87%8D%E5%81%9A%E6%97%A5%E5%BF%97%E6%96%87%E4%BB%B6%E7%BB%84.drawio.png)

重做日志文件组是以**循环写**的方式工作的，从头开始写，写到末尾就又回到开头，相当于一个环形。

所以 InnoDB 存储引擎会先写 ib_logfile0 文件，当 ib_logfile0 文件被写满的时候，会切换至 ib_logfile1 文件，当 ib_logfile1 文件也被写满时，会切换回 ib_logfile0 文件。

![重做日志文件组写入过程](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E9%87%8D%E5%81%9A%E6%97%A5%E5%BF%97%E6%96%87%E4%BB%B6%E7%BB%84%E5%86%99%E5%85%A5%E8%BF%87%E7%A8%8B.drawio.png)

InnoDB 用 write pos 表示 redo log 当前记录写到的位置，用 checkpoint 表示当前要擦除的位置

指针顺时针移动

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/checkpoint.png" alt="img" style="zoom: 33%;" />

如果 write pos 追上了 checkpoint，就意味着 **redo log 文件满了，这时 MySQL 不能再执行新的更新操作，也就是说 MySQL 会被阻塞**（*因此所以针对并发量大的系统，适当设置 redo log 的文件大小非常重要*），此时**会停下来将 Buffer Pool 中的脏页刷新到磁盘中，然后标记 redo log 哪些记录可以被擦除，接着对旧的 redo log 记录进行擦除，等擦除完旧记录腾出了空间，checkpoint 就会往后移动（图中顺时针）**，然后 MySQL 恢复正常运行，继续执行新的更新操作。

## binlog

undo log 和 redo log 这两个日志都是 Innodb 存储引擎生成的。binlog是由server层生成的。

事务提交的时候，会将该事物执行过程中产生的所有 binlog 统一写 入 binlog 文件。

binlog 文件是记录了所有数据库表结构变更和表数据修改的日志，不会记录查询类的操作，比如 SELECT 和 SHOW 操作。

### redo log 和 binlog的区别？

*1、适用对象不同：*

- binlog 是 MySQL 的 Server 层实现的日志，所有存储引擎都可以使用；
- redo log 是 Innodb 存储引擎实现的日志；

*2、文件格式不同：*

- binlog 有 3 种格式类型，分别是 STATEMENT（默认格式）、ROW、 MIXED，区别如下：
  - STATEMENT：每一条修改数据的 SQL 都会被记录到 binlog 中（相当于记录了逻辑操作，所以针对这种格式， binlog 可以称为逻辑日志），主从复制中 slave 端再根据 SQL 语句重现。但 STATEMENT 有动态函数的问题，比如你用了 uuid 或者 now 这些函数，你在主库上执行的结果并不是你在从库执行的结果，这种随时在变的函数会导致复制的数据不一致；
  - ROW：记录行数据最终被修改成什么样了，不会出现 STATEMENT 下动态函数的问题。但 ROW 的缺点是每行数据的变化结果都会被记录，如执行批量 update 语句，更新多少行数据就会产生多少条记录，使 binlog 文件过大。
  - MIXED：包含了 STATEMENT 和 ROW 模式，它会根据不同的情况自动使用 ROW 模式和 STATEMENT 模式；
- redo log 是物理日志，记录的是在某个数据页做了什么修改，比如对 XXX 表空间中的 YYY 数据页 ZZZ 偏移量的地方做了AAA 更新

*3、写入方式不同：*

- binlog 是追加写，写满一个文件，就创建一个新的文件继续写，不会覆盖以前的日志，保存的是全量的日志。
- redo log 是循环写，日志空间大小是固定，全部写满就从头开始，保存未被刷入磁盘的脏页日志。

*4、用途不同：*

- binlog 用于备份恢复、主从复制；
- redo log 用于掉电等故障恢复

### 主从复制的实现

这个过程一般是**异步**的，也就是主库上执行事务操作的线程不会等待复制 binlog 的线程同步完成。

![MySQL 主从复制过程](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6%E8%BF%87%E7%A8%8B.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

具体过程

- MySQL 主库在收到客户端提交事务的请求之后，会先写入 binlog，再提交事务，更新存储引擎中的数据，事务提交完成后，返回给客户端“操作成功”的响应。
- 从库会创建一个专门的 I/O 线程，连接主库的 log dump 线程，来接收主库的 binlog 日志，再把 binlog 信息写入 relay log 的中继日志里，再返回给主库“复制成功”的响应。
- 从库会创建一个用于回放 binlog 的线程，去读 relay log 中继日志，然后回放 binlog 更新存储引擎中的数据，最终实现主从的数据一致性。

![MySQL 主从架构](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%BB%E4%BB%8E%E6%9E%B6%E6%9E%84.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0)

### binlog 什么时候刷盘？

事务执行过程中，先把日志写到 binlog cache（Server 层的 cache），事务提交的时候，再把 binlog cache 写到 binlog 文件中。

一个事务的 binlog 是不能被拆开的，因此无论这个事务有多大（比如有很多条语句），也要保证一次性写入。这是因为有一个线程只能同时有一个事务在执行的设定。

MySQL 给每个线程分配了一片内存用于缓冲 binlog ，该内存叫 binlog cache，参数 binlog_cache_size 用于控制单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。

#### cache什么时候写到binlog文件当中

事务提交时

![binlog cach](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/binlogcache.drawio.png)

虽然每个线程有自己 binlog cache，但是最终都写到同一个 binlog 文件：

- 图中的 write，指的就是指把日志写入到 binlog 文件，但是并没有把数据持久化到磁盘，因为数据还缓存在文件系统的 page cache 里，write 的写入速度还是比较快的，因为不涉及磁盘 I/O。
- 图中的 fsync，才是将数据持久化到磁盘的操作，这里就会涉及磁盘 I/O，所以频繁的 fsync 会导致磁盘的 I/O 升高。

## 为什么需要两阶段提交（redolog和undolog的提交）？

举个例子，假设 id = 1 这行数据的字段 name 的值原本是 'jay'，然后执行 `UPDATE t_user SET name = 'xiaolin' WHERE id = 1;` 如果在持久化 redo log 和 binlog 两个日志的过程中，出现了半成功状态，那么就有两种情况：

- **如果在将 redo log 刷入到磁盘之后， MySQL 突然宕机了，而 binlog 还没有来得及写入**。MySQL 重启后，通过 redo log 能将 Buffer Pool 中 id = 1 这行数据的 name 字段恢复到新值 xiaolin，但是 binlog 里面没有记录这条更新语句，在主从架构中，binlog 会被复制到从库，由于 binlog 丢失了这条更新语句，从库的这一行 name 字段是旧值 jay，与主库的值不一致性；
- **如果在将 binlog 刷入到磁盘之后， MySQL 突然宕机了，而 redo log 还没有来得及写入**。由于 redo log 还没写，崩溃恢复以后这个事务无效，所以 id = 1 这行数据的 name 字段还是旧值 jay，而 binlog 里面记录了这条更新语句，在主从架构中，binlog 会被复制到从库，从库执行了这条更新语句，那么这一行 name 字段是新值 xiaolin，与主库的值不一致性；

**MySQL 为了避免出现两份日志之间的逻辑不一致的问题，使用了「两阶段提交」来解决**，

### 两阶段提交过程

为了保证这两个日志的一致性，MySQL 使用了**内部 XA 事务**

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="两阶段提交" style="zoom: 50%;" />

- **prepare 阶段**：将 XID（内部 XA 事务的 ID） 写入到 redo log，同时将 redo log 对应的事务状态设置为 prepare，然后将 redo log 持久化到磁盘（innodb_flush_log_at_trx_commit = 1 的作用）；
- **commit 阶段**：把 XID 写入到 binlog，然后将 binlog 持久化到磁盘（sync_binlog = 1 的作用），接着调用引擎的提交事务接口，将 redo log 状态设置为 commit。

### 异常重启会出现什么现象

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E4%B8%A4%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4%E5%B4%A9%E6%BA%83%E7%82%B9.drawio.png?image_process=watermark,text_5YWs5LyX5Y-377ya5bCP5p6XY29kaW5n,type_ZnpsdHpoaw,x_10,y_10,g_se,size_20,color_0000CD,t_70,fill_0" alt="时刻 A 与时刻 B" style="zoom:50%;" />

时刻A和时刻B， **redo log 都处于 prepare 状态**

在 MySQL 重启后会按顺序扫描 redo log 文件，碰到处于 prepare 状态的 redo log，就拿着 redo log 中的 XID 去 binlog 查看是否存在此 XID：

- **如果 binlog 中没有当前内部 XA 事务的 XID，说明 redolog 完成刷盘，但是 binlog 还没有刷盘，则回滚事务**。对应时刻 A 崩溃恢复的情况。
- **如果 binlog 中有当前内部 XA 事务的 XID，说明 redolog 和 binlog 都已经完成了刷盘，则提交事务**。对应时刻 B 崩溃恢复的情况。

**对于处于 prepare 阶段的 redo log，即可以提交事务，也可以回滚事务，这取决于是否能在 binlog 中查找到与 redo log 相同的 XID**，**两阶段提交是以 binlog 写成功为事务提交成功的标识**

### 两阶段提交的问题

- **磁盘 I/O 次数高**：对于“双1”配置，每个事务提交都会进行两次 fsync（刷盘），一次是 redo log 刷盘，另一次是 binlog 刷盘。
- **锁竞争激烈**：两阶段提交虽然能够保证「单事务」两个日志的内容一致，但在「多事务」的情况下，却不能保证两者的提交顺序一致，因此，在两阶段提交的流程基础上，还需要加一个锁来保证提交的原子性，从而保证多事务的情况下，两个日志的提交顺序一致。

#### 组提交

**MySQL 引入了 binlog 组提交（group commit）机制，当有多个事务提交的时候，会将多个 binlog 刷盘操作合并成一个，从而减少磁盘 I/O 的次数**

引入了组提交机制后，prepare 阶段不变，只针对 commit 阶段，将 commit 阶段拆分为三个过程：

- **flush 阶段**：多个事务按进入的顺序将 binlog 从 cache 写入文件（wrtie操作）（不刷盘）；
- **sync 阶段**：对 binlog 文件做 fsync 操作（多个事务的 binlog 合并一次刷盘）；
- **commit 阶段**：各个事务按顺序做 InnoDB commit 操作；

**每个阶段都有一个队列**，每个阶段有锁进行保护，因此保证了事务写入的顺序

<img src="http://keithlan.github.io/image/mysql_innodb_arch/commit_4.png" alt="每个阶段都有一个队列" style="zoom:67%;" />

**锁粒度减小了，这样就使得多个阶段可以并发执行，从而提升效率**

redo log组提交就是在 prepare 阶段不再让事务各自执行 redo log 刷盘操作，而是推迟到组提交的 flush 阶段，也就是说 prepare 阶段融合在了 flush 阶段。sync 阶段之前。通过延迟写 redo log 的方式，为 redolog 做了一次组写入

#### flush 阶段

第一个事务会成为 flush 阶段的 Leader，此时后面到来的事务都是 Follower ：

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%BB%84%E6%8F%90%E4%BA%A41.png)

接着，获取队列中的事务组，由绿色事务组的 Leader 对 redo log 做一次 write + fsync，即一次将同组事务的 redolog 刷盘：

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%BB%84%E6%8F%90%E4%BA%A42.png)

完成了 prepare 阶段后，将绿色这一组事务执行过程中产生的 binlog 写入 binlog 文件（调用 write，不会调用 fsync，所以不会刷盘，binlog 缓存在操作系统的文件系统中）。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/write_binlog.png)

 flush 阶段队列的作用是**用于支撑 redo log 的组提交**

#### sync 阶段

绿色这一组事务的 binlog 写入到 binlog 文件后，并不会马上执行刷盘的操作，而是**会等待一段时间**，这个等待的时长由 `Binlog_group_commit_sync_delay` 参数控制，**目的是为了组合更多事务的 binlog，然后再一起刷盘**，

sync 阶段队列的作用是**用于支持 binlog 的组提交**。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%BB%84%E6%8F%90%E4%BA%A44.png)

binlog组提交的限制因素有时间限制和容量限制

#### commit 阶段

最后进入 commit 阶段，调用引擎的提交事务接口，将 redo log 状态设置为 commit。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/mysql/how_update/%E7%BB%84%E6%8F%90%E4%BA%A46.png)

commit 阶段队列的作用是承接 sync 阶段的事务，完成最后的引擎提交，使得 sync 可以尽早的处理下一组事务，最大化组提交的效率。

## MySQL 磁盘 I/O 很高，有什么优化的方法？

现在我们知道事务在提交的时候，需要将 binlog 和 redo log 持久化到磁盘，那么如果出现 MySQL 磁盘 I/O 很高的现象，我们可以通过控制以下参数，来 “延迟” binlog 和 redo log 刷盘的时机，从而降低磁盘 I/O 的频率：

- 设置组提交的两个参数： binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，延迟 binlog 刷盘的时机，从而减少 binlog 的刷盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但即使 MySQL 进程中途挂了，也没有丢失数据的风险，因为 binlog 早被写入到 page cache 了，只要系统没有宕机，缓存在 page cache 里的 binlog 就会被持久化到磁盘。
- 将 sync_binlog 设置为大于 1 的值（比较常见是 100~1000），表示每次提交事务都 write，但累积 N 个事务后才 fsync，相当于延迟了 binlog 刷盘的时机。但是这样做的风险是，主机掉电时会丢 N 个事务的 binlog 日志。
- 将 innodb_flush_log_at_trx_commit 设置为 2。表示每次事务提交时，都只是缓存在 redo log buffer 里的 redo log 写到 redo log 文件，注意写入到「 redo log 文件」并不意味着写入到了磁盘，因为操作系统的文件系统中有个 Page Cache，专门用来缓存文件数据的，所以写入「 redo log文件」意味着写入到了操作系统的文件缓存，然后交由操作系统控制持久化到磁盘的时机。但是这样做的风险是，主机掉电的时候会丢数据

## 总结

 `UPDATE t_user SET name = 'xiaolin' WHERE id = 1;` 的流程如下:

1. 执行器负责具体执行，会调用存储引擎的接口，通过主键索引树搜索获取 id = 1 这一行记录：
   - 如果 id=1 这一行所在的数据页本来就在 buffer pool 中，就直接返回给执行器更新；
   - 如果记录不在 buffer pool，将数据页从磁盘读入到 buffer pool，返回记录给执行器。
2. 执行器得到聚簇索引记录后，会看一下更新前的记录和更新后的记录是否一样：
   - 如果一样的话就不进行后续更新流程；
   - 如果不一样的话就把更新前的记录和更新后的记录都当作参数传给 InnoDB 层，让 InnoDB 真正的执行更新记录的操作；
3. 开启事务， InnoDB 层更新记录前，首先要记录相应的 undo log，因为这是更新操作，需要把被更新的列的旧值记下来，也就是要生成一条 undo log，undo log 会写入 Buffer Pool 中的 Undo 页面，不过在内存修改该 Undo 页面后，需要记录对应的 redo log。
4. InnoDB 层开始更新记录，会先更新内存（同时标记为脏页），然后将记录写到 redo log 里面，这个时候更新就算完成了。为了减少磁盘I/O，不会立即将脏页写入磁盘，后续由后台线程选择一个合适的时机将脏页写入到磁盘。这就是 **WAL 技术**，MySQL 的写操作并不是立刻写到磁盘上，而是先写 redo 日志，然后在合适的时间再将修改的行数据写到磁盘上。
5. 至此，一条记录更新完了。
6. 在一条更新语句执行完成后，然后开始记录该语句对应的 binlog，此时记录的 binlog 会被保存到 binlog cache，并没有刷新到硬盘上的 binlog 文件，在事务提交时才会统一将该事务运行过程中的所有 binlog 刷新到硬盘。
7. 事务提交（为了方便说明，这里不说组提交的过程，只说两阶段提交）：
   - **prepare 阶段**：将 redo log 对应的事务状态设置为 prepare，然后将 redo log 刷新到硬盘；
   - **commit 阶段**：将 binlog 刷新到磁盘，接着调用引擎的提交事务接口，将 redo log 状态设置为 commit（将事务设置为 commit 状态后，刷入到磁盘 redo log 文件）；
8. 至此，一条更新语句执行完成。

## 内存篇

## Buffer Pool

默认大小为128MB

Buffer Pool缓存单位为缓存页，16KB大小

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/bufferpool%E5%86%85%E5%AE%B9.drawio.png)

为了更好的管理这些在 Buffer Pool 中的缓存页，InnoDB 为每一个缓存页都创建了一个**控制块**，控制块信息包括「缓存页的表空间、页号、缓存页地址、链表节点」等等。

控制块也是占有内存空间的，它是放在 Buffer Pool 的最前面，接着才是缓存页，如下图：

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/%E7%BC%93%E5%AD%98%E9%A1%B5.drawio.png)

### 管理Buffer Pool

为了能够快速找到空闲的缓存页，可以使用链表结构，将空闲缓存页的「控制块」作为链表的节点，这个链表称为 **Free 链表**（空闲链表）

Buffer Pool 是一片连续的内存空间

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/freelist.drawio.png)

该头节点包含链表的头节点地址，尾节点地址，以及当前链表中节点的数量

### 管理脏页

 **Flush 链表**，它跟 Free 链表类似的，链表的节点也是控制块，区别在于 Flush 链表的元素都是脏页。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/Flush.drawio.png)

#### 提高缓存命中率

LRU（Least recently used）算法

用一个LRU链表串联起热点页面

该算法的思路是，链表头部的节点是最近使用的，而链表末尾的节点是最久没被使用的。那么，当空间不够了，就淘汰最久没被使用的节点，从而腾出空间。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/lru.png)

### Buffer Pool 里有三种页和链表来管理数据。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/bufferpoll_page.png)

- Free Page（空闲页），表示此页未被使用，位于 Free 链表；
- Clean Page（干净页），表示此页已被使用，但是页面未发生修改，位于LRU 链表。
- Dirty Page（脏页），表示此页「已被使用」且「已经被修改」，其数据和磁盘上的数据已经不一致。当脏页上的数据写入磁盘后，内存数据和磁盘数据一致，那么该页就变成了干净页。脏页同时存在于 LRU 链表和 Flush 链表。

#### 但是简单的LRU算法有一些缺点

- 预读失效；

  MySQL 的预读机制。程序是有空间局部性的，靠近当前被访问数据的数据，在未来很大概率会被访问到。所以，MySQL 在加载数据页时，会提前把它相邻的数据页一并加载进来，目的是为了减少磁盘 IO

  但是如果这些**被提前加载进来的数据页，并没有被访问**，，这个就是**预读失效**。

  怎么解决预读失效

  最好就是**让预读的页停留在 Buffer Pool 里的时间要尽可能的短，让真正被访问的页才移动到 LRU 链表的头部，从而保证真正被读取的热数据留在 Buffer Pool 里的时间尽可能长**。

  将LRU区域分为old 和 new 区域

  **划分这两个区域后，预读的页就只需要加入到 old 区域的头部，当页被真正访问的时候，才将页插入 young 区域的头部**

  并且old区域占比比较小  37开

  ![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost4@main/mysql/innodb/young%2Bold.png)

- Buffer Pool 污染；

​	当某一个 SQL 语句**扫描了大量的数据**时，在 Buffer Pool 空间比较有限的情况下，可能会将 **Buffer Pool 里的所有页都替换出去，导致大量热数据被淘汰了**，等这些热数据又被再次访问的时候，由于缓存未命中，就会产生大量的磁盘 IO，MySQL 性能就会急剧下降，这个过程被称为 **Buffer Pool 污染**。

注意， Buffer Pool 污染并不只是查询语句查询出了大量的数据才出现的问题，即使查询出来的结果集很小，也会造成 Buffer Pool 污染。

比如，在一个数据量非常大的表，执行了这条语句：

```sql
select * from t_user where name like "%xiaolin%";
```

可能这个查询出来的结果就几条记录，但是由于这条语句会发生索引失效，所以这个查询过程是全表扫描的，接着会发生如下的过程：

- 从磁盘读到的页加入到 LRU 链表的 old 区域头部；
- 当从页里读取行记录时，也就是页被访问的时候，就要将该页放到 young 区域头部；
- 接下来拿行记录的 name 字段和字符串 xiaolin 进行模糊匹配，如果符合条件，就加入到结果集里；
- 如此往复，直到扫描完表中的所有记录。

经过这一番折腾，原本 young 区域的热点数据都会被替换掉。

#### 怎么解决

进入到 young 区域条件增加了一个**停留在 old 区域的时间判断**。

在对某个处在 old 区域的缓存页进行第一次访问时，就在它对应的控制块中记录下来这个访问时间：

- 如果后续的访问时间与第一次访问的时间**在某个时间间隔内**，那么**该缓存页就不会被从 old 区域移动到 young 区域的头部**；
- 如果后续的访问时间与第一次访问的时间**不在某个时间间隔内**，那么**该缓存页移动到 young 区域的头部**；

脏页的落盘时机

- 当 redo log 日志满了的情况下，会主动触发脏页刷新到磁盘；
- Buffer Pool 空间不足时，需要将一部分数据页淘汰掉，如果淘汰的是脏页，需要先将脏页同步到磁盘；
- MySQL 认为空闲时，后台线程会定期将适量的脏页刷入到磁盘；
- MySQL 正常关闭之前，会把所有的脏页刷入到磁盘；
