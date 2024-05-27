Redis除了做缓存，还能做什么

- **分布式锁**：通过 Redis 来做分布式锁是一种比较常见的方式。通常情况下，我们都是基于 Redisson 来实现分布式锁。关于 Redis 实现分布式锁的详细介绍，可以看我写的这篇文章：[分布式锁详解open in new window](https://javaguide.cn/distributed-system/distributed-lock.html) 。
- **限流**：一般是通过 Redis + Lua 脚本的方式来实现限流。如果不想自己写 Lua 脚本的话，也可以直接利用 Redisson 中的 `RRateLimiter` 来实现分布式限流，其底层实现就是基于 Lua 代码+令牌桶算法。
- **消息队列**：Redis 自带的 List 数据结构可以作为一个简单的队列使用。Redis 5.0 中增加的 Stream 类型的数据结构更加适合用来做消息队列。它比较类似于 Kafka，有主题和消费组的概念，支持消息持久化以及 ACK 机制。
- **延时队列**：Redisson 内置了延时队列（基于 Sorted Set 实现的）。
- **分布式 Session** ：利用 String 或者 Hash 数据类型保存 Session 数据，所有的服务器都可以访问。
- **复杂业务场景**：通过 Redis 以及 Redis 扩展（比如 Redisson）提供的数据结构，我们可以很方便地完成很多复杂的业务场景比如通过 Bitmap 统计活跃用户、通过 Sorted Set 维护排行榜。

### [如何基于 Redis 实现延时任务？](https://javaguide.cn/database/redis/redis-questions-01.html#如何基于-redis-实现延时任务)

两种方案

1. Redis 过期事件监听（不好用）
2. Redisson 内置的延时队列

Redisson 内置的延时队列具备下面这些优势：

1. **减少了丢消息的可能**：DelayedQueue 中的消息会被持久化，即使 Redis 宕机了，根据持久化机制，也只可能丢失一点消息，影响不大。当然了，你也可以使用扫描数据库的方法作为补偿机制。
2. **消息不存在重复消费问题**：每个客户端都是从同一个目标队列中获取任务的，不存在重复消费的问题

### redis的缓存淘汰策略

- **LRU（Least Recently Used，最近最少使用）**：LRU 策略根据键最近被访问的时间来淘汰键。当内存不足时，Redis 会淘汰最近最少被访问的键。
- **LFU（Least Frequently Used，最不经常使用）**：LFU 策略根据键被访问的频率来淘汰键。当内存不足时，Redis 会淘汰访问频率最低的键
- **TTL（Time-To-Live，生存时间）**
- **随机淘汰**

## Redis常见面试题

是一个基于内存的数据库，可用于缓存，分布式锁，消息队列

Redis具有高性能和**高并发**：Redis 单机的 QPS 能轻松破 10w，而 MySQL 单机的 QPS 很难破 1w

### 数据结构类型

**String（字符串），Hash（哈希），List（列表），Set（集合）、Zset（有序集合）**

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/key.png)

后续版本还支持

BitMap，Stream等数据结构

### 数据结构底层实现

#### String 类型内部实现

实现主要是 SDS（简单动态字符串），相比于C字符串有以下功能

- 不仅可以存储文本数据，还可以存储二进制数据（因为C语言的字符串是以'/0'结尾的，而redis是是以len属性值结尾）
- **SDS 获取字符串长度的时间复杂度是 O(1**），SDS 结构里用 len 属性记录了字符串长度，所以复杂度为 O(1)
- **Redis 的 SDS API 是安全的，拼接字符串不会造成缓冲区溢出**

List 类型内部实现

**双向链表或压缩列表**,**在 Redis 3.2 版本之后，List 数据类型底层数据结构就只由 quicklist 实现了，替代了双向链表和压缩列表**

Hash 类型内部实现

**压缩列表或哈希表**

##  Redis 线程模型

### **Redis 单线程指的是「接收客户端请求->解析请求 ->进行数据读写等操作->发送数据给客户端」这个过程是由一个线程（主线程）来完成的**

**Redis 程序并不是单线程的**，Redis 在启动的时候，是会**启动后台线程**（BIO）的

后台线程主要用于关闭文件，AOF刷盘，释放redis内存，因为这些操作耗时，交给主线程的话可能会导致主线阻塞卡顿而无法处理新的请求

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/%E5%90%8E%E5%8F%B0%E7%BA%BF%E7%A8%8B.jpg)

- BIO_CLOSE_FILE，关闭文件任务队列：当队列有任务后，后台线程会调用 close(fd) ，将文件关闭；
- BIO_AOF_FSYNC，AOF刷盘任务队列：当 AOF 日志配置成 everysec 选项后，主线程会把 AOF 写日志操作封装成一个任务，也放到队列中。当发现队列有任务后，后台线程会调用 fsync(fd)，将 AOF 文件刷盘，
- BIO_LAZY_FREE，lazy free 任务队列：当队列有任务后，后台线程会 free(obj) 释放对象 / free(dict) 删除数据库所有对象 / free(skiplist) 释放跳表对象

### Redis单线程模式

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/redis%E5%8D%95%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B.drawio.png" alt="img" style="zoom:67%;" />

epoll是一种事件轮询机制，用于监视多个事件的机制，在发生事件时通知相应的处理程序

一旦有事件发生，系统就会通知应用程序，然后应用程序可以执行相应的处理逻辑

**redis初始化时**

- **创建 epoll 对象：** 调用 epoll_create() 函数创建一个 epoll 对象，用于管理事件的监视和通知
- **创建服务端 socket：** 调用 socket() 函数创建一个服务端 socket，并通过 bind() 绑定到指定的网络端口，然后通过 listen() 开始监听连接请求
- **将 listen socket 加入 epoll：** 调用 epoll_ctl() 将监听 socket 加入到 epoll 实例中，并注册对于 "连接事件"（EPOLLIN）的监听。这样一旦有客户端连接请求到达，就会触发 "连接事件"，服务器就可以接受并处理这些连接请求

初始化完后，主线程就进入到一个**事件循环函数**

- **处理发送队列函数：** 在处理客户端连接的过程中，服务器首先会检查发送队列是否有待发送的数据。如果有待发送的数据，则通过 write 函数将数据发送出去。如果一次发送没有完成，服务器将注册写事件处理函数，并等待 epoll_wait 发现可写事件后继续发送
- **调用 epoll_wait 函数等待事件的到来：** 服务器通过 epoll_wait 函数等待事件的到来。根据不同类型的事件，服务器会调用相应的事件处理函数来处理事件。
  - **连接事件处理函数：** 当有新的客户端连接到服务器时，服务器会调用连接事件处理函数。该函数会执行接受连接、将连接的 socket 加入到 epoll 中，并注册读事件处理函数等操作，以便服务器能够处理客户端发送的数据。
  - **读事件处理函数：** 当客户端向服务器发送数据时，服务器会调用读事件处理函数。该函数会执行读取客户端发送的数据、解析命令、处理命令、将执行结果写入发送队列等操作，以便服务器能够响应客户端的请求。
  - **写事件处理函数：** 当服务器可以向客户端发送数据时，服务器会调用写事件处理函数。该函数会执行通过 write 函数将发送缓存区中的数据发送出去的操作。如果一次发送没有完成，服务器将继续注册写事件处理函数，等待 epoll_wait 发现可写事件后继续发送

### Redis采用单线程速度快的原因

- 操作都在内存中完成
- 单线程减少的多线程的竞争，避免了上下文的切换成本
- IO采用多路复用机制，一个线程处理多个IO流，select/epoll 机制

### Redis的持久化

#### AOF日志实现

先操作内存再写入硬盘

<img src="https://cdn.xiaolincoding.com//mysql/other/6f0ab40396b7fc2c15e6f4487d3a0ad7-20230309232240301.png" alt="img" style="zoom:50%;" />

AOF的写回策略

<img src="https://cdn.xiaolincoding.com//mysql/other/4eeef4dd1bedd2ffe0b84d4eaa0dbdea-20230309232249413.png" alt="img" style="zoom:50%;" />

1. Redis 执行完写操作命令后，会将命令追加到 server.aof_buf 缓冲区
2. 然后通过 write() 系统调用，将 aof_buf 缓冲区的数据写入到 AOF 文件，此时数据并没有写入到硬盘，而是拷贝到了内核缓冲区 page cache，等待内核将数据写入硬盘
3. 具体内核缓冲区的数据什么时候写入到硬盘，由内核决定

redis有三种策略

<img src="https://cdn.xiaolincoding.com//mysql/other/98987d9417b2bab43087f45fc959d32a-20230309232253633.png" alt="img" style="zoom:67%;" />

### AOF日记过大时触发机制

采用AOF重写机制**:当 AOF 文件的大小超过所设定的阈值后，Redis 就会启用 AOF 重写机制，来压缩 AOF 文件**。

AOF 重写机制是在重写时，读取当前数据库中的所有键值对，然后将每一个键值对用一条命令记录到「新的 AOF 文件」，等到全部记录完后，就将新的 AOF 文件替换掉现有的 AOF 文件

![img](https://cdn.xiaolincoding.com//mysql/other/723d6c580c05400b3841bc69566dd61b-20230309232257343.png)

### AOF重写日志的过程

**重写 AOF 过程是由后台子进程 \*bgrewriteaof\* 来完成的**

由子进程完成的原因：

1. 子进程进行 AOF 重写期间，主进程可以继续处理命令请求，从而避免阻塞主进程
2. 子进程带有主进程的数据副本，这里使用子进程而不是线程，因为如果是使用线程，多线程之间会共享内存，那么在修改共享内存数据的时候，需要通过加锁来保证数据的安全，而这样就会降低性能。而使用子进程，创建子进程时，父子进程是共享内存数据的，不过这个共享的内存只能以只读的方式，而当父子进程任意一方修改了该共享内存，就会发生「写时复制」，于是父子进程就有了独立的数据副本，就不用加锁来保证数据安全

**但也有一个问题，就是如果再重写过程中，如果主进程发生了数据修改，这时就会数据不一致了**

为此Redis 设置了一个 **AOF 重写缓冲区**，这个缓冲区在创建 bgrewriteaof 子进程之后开始使用。

在重写 AOF 期间，当 Redis 执行完一个写命令之后，它会**同时将这个写命令写入到 「AOF 缓冲区」和 「AOF 重写缓冲区」**

当子进程完成重写后，会向主进程发送一条信号，信号是进程间通讯的一种方式，且是异步的。

主进程收到该信号后，会调用一个信号处理函数，该函数主要做以下工作：

- 将 AOF 重写缓冲区中的所有内容追加到新的 AOF 的文件中，使得新旧两个 AOF 文件所保存的数据库状态一致；
- 新的 AOF 的文件进行改名，覆盖现有的 AOF 文件

<img src="https://cdn.xiaolincoding.com//mysql/other/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0ODI3Njc0,size_16,color_FFFFFF,t_70-20230309232301042.png" alt="img" style="zoom: 50%;" />

### RDB快照

记录某一个瞬间的内存数据，记录的是实际数据，是全部数据

恢复数据时，直接读入内存即可

### 保存快照时是否会阻塞线程

Redis 提供了两个命令来生成 RDB 文件，分别是 save 和 bgsave，他们的区别就在于是否在「主线程」里执行

- save会在主线程执行，时间过长会阻塞主线程
- bgsave时后台生成一个子线程，可以每间隔一段时间内执行bgsave

### 执行快照时，数据的修改

执行 bgsave 命令的时候，会通过 fork() 创建子进程，此时子进程和父进程是共享同一片内存数据的，因为创建子进程的时候，会复制父进程的页表，但是页表指向的物理内存还是一个

<img src="https://cdn.xiaolincoding.com//mysql/other/c34a9d1f58d602ff1fe8601f7270baa7-20230309232304226.png" alt="img" style="zoom: 50%;" />

父进程修改数据后，会发生写时复制，会创建一个数据副本，子进程把数据副本记录在自身

<img src="https://cdn.xiaolincoding.com//mysql/other/ebd620db8a1af66fbeb8f4d4ef6adc68-20230309232308604.png" alt="img" style="zoom: 50%;" />

### 混合持久化

在aof文件重写时，子进程先以RDB的形式放入aof文件当中，然后主线程处理的操作命令会被记录在重写缓冲区里，，重写缓冲区里的增量命令会以 AOF 方式写入到 AOF 文件

![img](https://cdn.xiaolincoding.com//mysql/other/f67379b60d151262753fec3b817b8617-20230309232312657.png)

优缺点

优点：redis启动更快因为RDB，而且结合 AOF 的优点，有减低了大量数据丢失的风险

缺点：aof文件可读形式差，兼容性差，一些版本不适用

### Redis集群

三大模式

#### 主从复制

将主服务器的读写操作同步到其他的从服务器，并且主从服务器读写分离，从服务器只读，客户端只能对主服务器进行写操作。

并且主从复制的操作时异步，主服务器传播信息时，不会等待从服务器的返回，而会直接返回给客户端，这样可能会造成数据的不一致

<img src="https://cdn.xiaolincoding.com//mysql/other/2b7231b6aabb9a9a2e2390ab3a280b2d.png" alt="img" style="zoom:50%;" />

#### 哨兵模式

哨兵模式做到了可以监控主从服务器，并且提供**主从节点故障转移的功能**

<img src="https://cdn.xiaolincoding.com//mysql/other/26f88373d8454682b9e0c1d4fd1611b4.png" alt="img" style="zoom:50%;" />

#### 切片集群（Redis Cluster ）

当redis的缓存数据太大时，单个redis节点的性能收到影响，会将数据分布在不同的服务器上

**一个切片集群共有 16384 个哈希槽**，每个键值对都会根据它的 key，被映射到一个哈希槽中，具体执行过程分为两大步：

- 根据键值对的 key，按照 [CRC16 算法 (opens new window)](https://en.wikipedia.org/wiki/Cyclic_redundancy_check)计算一个 16 bit 的值。
- 再用 16bit 值对 16384 取模，得到 0~16383 范围内的模数，每个模数代表一个相应编号的哈希槽。

对于哈希槽由两种方案来映射到redis节点当中

- **平均分配：** 在使用 cluster create 命令创建 Redis 集群时，Redis 会自动把所有哈希槽平均分布到集群节点上。比如集群中有 9 个节点，则每个节点上槽的个数为 16384/9 个。
- **手动分配：** 可以使用 cluster meet 命令手动建立节点间的连接，组成集群，再使用 cluster addslots 命令，指定每个节点上的哈希槽个数。

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/redis%E5%88%87%E7%89%87%E9%9B%86%E7%BE%A4%E6%98%A0%E5%B0%84%E5%88%86%E5%B8%83%E5%85%B3%E7%B3%BB.jpg" alt="img" style="zoom:50%;" />

### Redis过期删除与内存淘汰

可以对 key 设置过期时间的，每当我们对一个 key 设置了过期时间时，Redis 会把该 key 带上过期时间存储到一个**过期字典**（expires dict）中

当我们查询一个 key 时，Redis 首先检查该 key 是否存在于过期字典中

- 不在则正常读取
- 在则比较过期时间是否生效

Redis 使用的过期删除策略是「**惰性删除+定期删除**」这两种策略配和使用

惰性删除策略的做法是，**不主动删除过期键，每次从数据库访问 key 时，都检测 key 是否过期，如果过期则删除该 key**

- 优点：只有访问才会检查key是否过期，该策略使用较少资源
- 缺点：会造成内存浪费

定期删除策略的做法是，**每隔一段时间「随机」从数据库（过期字典当中）中取出一定数量的 key 进行检查，并删除其中的过期key**

流程如下

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E8%BF%87%E6%9C%9F%E7%AD%96%E7%95%A5/%E5%AE%9A%E6%97%B6%E5%88%A0%E9%99%A4%E6%B5%81%E7%A8%8B.jpg" alt="img" style="zoom:50%;" />

### Redis 持久化时，对过期键会如何处理的？

**RDB生成阶段：从内存状态持久化成 RDB（文件）的时候，会对 key 进行过期检查，**过期的键「不会」被保存到新的 RDB 文件中**RDB 加载阶段**：

- **如果 Redis 是「主服务器」运行模式的话，在载入 RDB 文件时，程序会对文件中保存的键进行检查，过期键「不会」被载入到数据库中**。
- **如果 Redis 是「从服务器」运行模式的话，在载入 RDB 文件时，不论键是否过期都会被载入到数据库中**

- **AOF 文件写入阶段**：当 Redis 以 AOF 模式持久化时，**如果数据库某个过期键还没被删除，那么 AOF 文件会保留此过期键，当此过期键被删除后，Redis 会向 AOF 文件追加一条 DEL 命令来显式地删除该键值**。
- **AOF 重写阶段**：执行 AOF 重写时，会对 Redis 中的键值对进行检查，**已过期的键不会被保存到重写后的 AOF 文件中**

### 主从模式下的过期键

**从库不会进行过期扫描，从库对过期的处理是被动的**，如果客户端进行查询依旧可以查询到

**主库在 key 到期时，会在 AOF 文件里增加一条 del 指令，同步到所有的从库**

### Redis内存淘汰机制

redis内存到达阈值会触发此机制

有两类淘汰策略

***不进行数据淘汰的策略***

**noeviction**（Redis3.0之后，默认的内存淘汰策略） ：它表示当运行内存超过最大设置内存时，不淘汰任何数据，而是不再提供服务，直接返回错误。

***进行数据淘汰的策略***

针对「进行数据淘汰」这一类策略，又可以细分为「在设置了过期时间的数据中进行淘汰」和「在所有数据范围内进行淘汰」这两类策略。 在设置了过期时间的数据中进行淘汰

在所有数据范围内进行淘汰：

- **allkeys-random**：随机淘汰任意键值;
- **allkeys-lru**：淘汰整个键值中最久未使用的键值；
- **allkeys-lfu**（Redis 4.0 后新增的内存淘汰策略）：淘汰整个键值中最少使用的键值

### LRU算法和LFU算法

**LRU** 全称是 Least Recently Used 翻译为**最近最少使用**，会选择淘汰最近最少使用的数据。

传统算法是基于链表实现的

#### Redis的LRU实现

**实现方式是在 Redis 的对象结构体中添加一个额外的字段，用于记录此数据的最后一次访问时间**

当 Redis 进行内存淘汰时，会使用**随机采样的方式来淘汰数据**，它是随机取 5 个值（此值可配置），然后**淘汰最久没有使用的那个**

但会有缓存污染问题，就是大量数据只读取一次，但遗留在那里

#### LFU算法

LFU 全称是 Least Frequently Used 翻译为**最近最不常用的**，是根据访问次数来做判断

LFU 算法相比于 LRU 算法的实现，多记录了「数据的访问频次」的信息

对象结构

```c
typedef struct redisObject {
    ...
      
    // 24 bits，用于记录对象的访问信息
    unsigned lru:24;  
    ...
} robj;
```

**在 LRU 算法中**，Redis 对象头的 24 bits 的 lru 字段是用来记录 key 的访问时间戳

**在 LFU 算法中**，Redis对象头的 24 bits 的 lru 字段被分成两段来存储，高 16bit 存储 ldt(Last Decrement Time)，用来记录 key 的访问时间戳；低 8bit 存储 logc(Logistic Counter)，用来记录 key 的访问频次

## Redis 缓存设计

### 如何避免缓存雪崩、

当**大量缓存数据在同一时间过期（失效）\**时，如果此时有大量的用户请求，都无法在 Redis 中处理，于是全部请求都直接访问数据库，从而导致数据库的压力骤增，严重的会造成数据库宕机，从而形成一系列连锁反应，造成整个系统崩溃，这就是\**缓存雪崩**

两种方案：

- **将缓存失效时间随机打散：** 我们可以在原有的失效时间基础上增加一个随机值（比如 1 到 10 分钟）这样每个缓存的过期时间都不重复了，也就降低了缓存集体失效的概率。
- **设置缓存不过期：** 我们可以通过后台服务来更新缓存数据，从而避免因为缓存失效造成的缓存雪崩，也可以在一定程度上避免缓存并发问题

### 缓存更新策略

Cache Aside（旁路缓存）策略，这个策略适合读多写少的场景

**写策略的步骤：**

- 先更新数据库中的数据，再删除缓存中的数据。

  先写后删是因为缓存的写入比数据库的写入速度快

**读策略的步骤：**

- 如果读取的数据命中了缓存，则直接返回数据；
- 如果读取的数据没有命中缓存，则从数据库中读取数据，然后将数据写入到缓存，并且返回给用户

## Redis实战

### 实现延迟队列，可以由Zset数据结构来实现

ZSet 有一个 Score 属性可以用来存储延迟执行的时间。

使用 zadd score1 value1 命令就可以一直往内存中生产消息。再利用 zrangebysocre 查询符合条件的所有待处理的任务， 通过循环执行队列任务即可。

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/%E5%BB%B6%E8%BF%9F%E9%98%9F%E5%88%97.png)

### 大key处理

一般而言，下面这两种情况被称为大 key：key对应的数值比较大

- String 类型的值大于 10 KB；
- Hash、List、Set、ZSet 类型的元素的个数超过 5000个

大 key 会带来以下四种影响：

- **客户端超时阻塞**。由于 Redis 执行命令是单线程处理，然后在操作大 key 时会比较耗时，那么就会阻塞 Redis，从客户端这一视角看，就是很久很久都没有响应。
- **引发网络阻塞**。每次获取大 key 产生的网络流量较大，如果一个 key 的大小是 1 MB，每秒访问量为 1000，那么每秒会产生 1000MB 的流量，这对于普通千兆网卡的服务器来说是灾难性的。
- **阻塞工作线程**。如果使用 del 删除大 key 时，会阻塞工作线程，这样就没办法处理后续的命令。
- **内存分布不均**。集群模型在 slot 分片均匀情况下，会出现数据和查询倾斜情况，部分有大 key 的 Redis 节点占用内存多，QPS 也会比较大

如何删除大 key？

***1、分批次删除***

一次性删除可能会造成主线程的阻塞

***2、异步删除***

用一个异步线程去执行删除操作

### Redis 管道有什么用

管道技术（Pipeline）是客户端提供的一种批处理技术，用于一次处理多个 Redis 命令，从而提高整个交互的性能。

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/%E6%99%AE%E9%80%9A%E5%91%BD%E4%BB%A4%E6%A8%A1%E5%BC%8F.jpg" alt="img" style="zoom:50%;" />

管道模式

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/%E7%AE%A1%E9%81%93%E6%A8%A1%E5%BC%8F.jpg" alt="img" style="zoom:50%;" />

### 分布式锁的实现

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E5%85%AB%E8%82%A1%E6%96%87/%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81.jpg" alt="img" style="zoom: 33%;" />

Redis 本身可以被多个客户端共享访问，正好就是一个共享存储系统，可以用来保存分布式锁，而且 Redis 的读写性能高，可以应对高并发的锁操作场景。

用SETNX来实现分布式锁

#### 加锁应该有三个条件

- 加锁包括了读取锁变量、检查锁变量值和设置锁变量值三个操作，但需要以原子操作的方式完成，所以，我们使用 SET 命令带上 NX 选项来实现加锁；setnx是原子操作
- 锁变量需要设置过期时间，以免客户端拿到锁后发生异常，导致锁一直无法释放，所以，我们在 SET 命令执行时加上 EX/PX 选项，设置其过期时间；
- 锁变量的值需要能区分来自不同客户端的加锁操作，以免在释放锁时，出现误释放操作，所以，我们使用 SET 命令设置锁变量值时，每个客户端设置的值是一个唯一值，用于标识客户端

## Redis数据类型篇

### string

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/string.png" alt="img" style="zoom: 50%;" />

底层数据结构有**int 和 SDS（简单动态字符串）**

SDS相对于普通的字符串区别

- **SDS 不仅可以保存文本数据，还可以保存二进制数据**。因为 `SDS` 使用 `len` 属性的值而不是空字符来判断字符串是否结束。能保存图片、音频、视频、压缩文件这样的二进制数据。
- 有一个len属性来记录字符串长度，所以获取 **获取字符串长度的时间复杂度是 O(1)**
- **Redis 的 SDS API 是安全的，拼接字符串不会造成缓冲区溢出**，拼接前会检查字符串的空间是否充足

#### 内部对象编码有三种

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/string%E7%BB%93%E6%9E%84.png)

应用场景

1.  缓存对象
2. 常规计数，因为 Redis 处理命令是单线程，所以执行命令的过程是原子的。因此 String 数据类型适合计数场景，比如计算访问次数、点赞、转发、库存数量等等
3. 分布式锁，SET 命令有个 NX 参数可以实现「key不存在才插入」，可以用它来实现分布式锁

#### 共享 Session 信息

我们系统会使用 Session 来保存用户的会话(登录)状态，这些 Session 信息会被保存在服务器端，但在分布式情况下，有多个服务器，如果访问的一个服务器没有保存就要重复登录。

所以我们可以借助redis来实现，服务器都会去同一个 Redis 获取相关的 Session 信息，这样就解决了分布式系统下 Session 存储的问题

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/Session2.png)

### List

列表的最大长度为 `2^32 - 1`，也即每个列表支持超过 `40 亿`个元素,

**内部实现由双向链表或压缩列表，3.2版本后用quicklist实现**

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/list.png)

#### 应用场景

消息队列的实现，但也要保证消息的有序性，处理重复和消息可靠性

List来实现队列的话有个问题就是塞入消息后无法通知消费者，需要程序中不停地调用 `RPOP` 命令（比如使用一个while(1)循环来查看有没有新结果

因此List也出现了**阻塞式读取**，BRPOP 命令当没有消息会阻塞

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97.png" alt="img" style="zoom: 33%;" />

对于消息的重复性处理，需要我们为每个消息生成一个全局id，并且由另一个队列来记录已经完成消息消费的id(可以用redis来记录表示)

那消息的可靠性就是List 类型提供了 `BRPOPLPUSH` 命令，这个命令的**作用是让消费者程序从一个 List 中读取消息，同时，Redis 会把这个消息再插入到另一个 List（可以叫作备份 List）留存**，这样当出现异常时就会重新读取消息消费

#### List的缺陷

消息无法供多个消费者消费

### Hash

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/hash.png" alt="img" style="zoom:33%;" />

**由哈希表或压缩列表组成**

#### 应用场景

Hash 类型的 （key，field， value） 的结构与对象的（对象id， 属性， 值）的结构相似，也可以用来存储对象

### set

一个集合最多可以存储 `2^32-1` 个元素

Set 类型的底层数据结构是由**哈希表或整数集合**实现的

#### 应用场景

有一个潜在的风险。**Set 的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，会导致 Redis 实例阻塞**。

#### 点赞

Set 类型可以保证一个用户只能点一个赞

#### 共同关注

Set 类型支持交集运算，所以可以用来计算共同关注的好友、公众号等

#### 抽奖活动

存储某活动中中奖的用户名 ，Set 类型因为有去重功能，可以保证同一个用户不会中奖两次。

### Zset

<img src="https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/zset.png" alt="img" style="zoom:33%;" />

**由压缩列表或跳表实现**

#### 应用场景

#### 排行榜

有序集合比较典型的使用场景就是排行榜

###  BitMap

**底层大概是bit数组**

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/bitmap.png)

####  应用场景

#### 签到统计

在签到打卡的场景中，我们只用记录签到（1）或未签到（0）

```java
BITCOUNT uid:sign:100:202206 //可以用来统计签到次数
```

#### 判断用户登陆态

Bitmap 提供了 `GETBIT、SETBIT` 操作，通过一个偏移值 offset 对 bit 数组的 offset 位置的 bit 位进行读写操作，需要注意的是 offset 从 0 开始。

只需要一个 key = login_status 表示存储用户登陆状态集合数据， 将用户 ID 作为 offset，在线就设置为 1，下线设置 0

```shell
SETBIT login_status 10086 1
```

#### 连续签到用户总数

我们把每天的日期作为 Bitmap 的 key，userId 作为 offset，若是打卡则将 offset 位置的 bit 设置成 1。

key 对应的集合的每个 bit 位的数据则是一个用户在该日期的打卡记录。

一共有 7 个这样的 Bitmap，如果我们能对这 7 个 Bitmap 的对应的 bit 位做 『与』运算。

1. 000001000
2. 000001000
3. 000001000

这样就表示连续签到三天了

## 数据结构

不同于数据类型string，list等，每个redis对象的对象结构是一样的

<img src="https://cdn.xiaolincoding.com//mysql/other/58d3987af2af868dca965193fb27c464.png" alt="img" style="zoom:33%;" />

- type，标识该对象是什么类型的对象（String 对象、 List 对象、Hash 对象、Set 对象和 Zset 对象）；
- encoding，标识该对象使用了哪种底层的数据结构；
- **ptr，指向底层数据结构的指针**

### SDS

结构设计

<img src="https://cdn.xiaolincoding.com//mysql/other/516738c4058cdf9109e40a7812ef4239.png" alt="img" style="zoom: 50%;" />

- **alloc，分配给字符数组的空间长度**，可以通过 `alloc - len` 计算出剩余的空间大小，可以用来判断空间是否满足修改需求，不满足会自动扩容
- **flags，用来表示不同类型的 SDS**，有5种类型，5 种类型的主要**区别就在于，它们数据结构中的 len 和 alloc 成员变量的数据类型不同**，可以节约内存空间。
- 此外redis还**告诉编译器取消结构体在编译过程中的优化对齐，按照实际占用字节数进行对齐**

<img src="https://cdn.xiaolincoding.com//mysql/other/35820959e8cf4376391c427ed7f81495.png" alt="img" style="zoom: 50%;" />

<img src="https://cdn.xiaolincoding.com//mysql/other/47e6c8fbc17fd6c89bdfcb5eedaaacff.png" alt="img" style="zoom:50%;" />

### 链表

redis的链表结构

是一个双向链表，有专门记录头节点和尾节点还有len来记录链表长度

有专门的函数来操作节点，例如释放，比较，复制

因此内存开销比较大

![img](https://cdn.xiaolincoding.com//mysql/other/cadf797496816eb343a19c2451437f1e.png)

### 压缩列表

结构设计

本质上是一个数组，添加了元素个数，尾节点的偏移量，结尾标识符和列表长度

**由连续内存块组成的顺序型数据结构**，有点类似于数组

![img](https://cdn.xiaolincoding.com//mysql/other/ab0b44f557f8b5bc7acb3a53d43ebfcb.png)

**头尾元素时间复杂度为o（1），中间为o（n)**

节点构成

<img src="https://cdn.xiaolincoding.com//mysql/other/a3b1f6235cf0587115b21312fe60289c.png" alt="img" style="zoom:67%;" />

- ***prevlen***，记录了「前一个节点」的长度，目的是为了实现从后向前遍历；
- ***encoding***，记录了当前节点实际数据的「类型和长度」，类型主要有两种：字符串和整数。
- ***data***，记录了当前节点的实际数据，类型和长度都由 `encoding` 决定

会根据数据类型是字符串还是整数，以及数据的大小，会使用不同空间大小的 prevlen 和 encoding 这两个元素里保存的信息

就像prevlen，如果前面长度小于254，就用1字节存储，大于就用5字节

这样的缺陷是会造成**连锁更新**

假设当前所有节点长度为253，但我在entry1插入一个255的节点，那么e1就会把prevlen改为5字节，那么后续的节点也会持续更新，这样就造成内存空间的重新分配，造成性能下降

### 哈希表

哈希表是一个数组（dictEntry **table），数组的每个元素是一个指向「哈希表节点（dictEntry）」的指针。

哈希冲突就链式哈希解决，链式哈希也有局限就是会访问元素时是o（n）

<img src="https://cdn.xiaolincoding.com//mysql/other/dc495ffeaa3c3d8cb2e12129b3423118.png" alt="img" style="zoom:50%;" />

所以到达一定程度后会进行扩展也就是rehash

我们会定义两个哈希表，平时操作都会在哈希表1中进行，

但是当哈希表的负载因子过大时，就会触发渐进式rehash，

将哈希表1中的数据转移到哈希表2中，并且是渐进式就是每次客户的请求操作，我们都会转移一部分到hash2中，

因为一次性转移太多会造成主线程的阻塞

完成转移后，哈希表2变为哈希表1，哈希表1为空

- **当负载因子大于等于 1 ，并且 Redis 没有在执行 bgsave 命令或者 bgrewiteaof 命令，也就是没有执行 RDB 快照或没有进行 AOF 重写的时候，就会进行 rehash 操作。**
- **当负载因子大于等于 5 时，此时说明哈希冲突非常严重了，不管有没有有在执行 RDB 快照或 AOF 重写，都会强制进行 rehash 操作**

<img src="https://cdn.xiaolincoding.com//mysql/other/85f597f7851b90d6c78bb0d8e39690fc.png" alt="img" style="zoom:33%;" />

<img src="https://cdn.xiaolincoding.com//mysql/other/2fedbc9cd4cb7236c302d695686dd478.png" alt="img" style="zoom: 33%;" />

### 整数集合

整数集合是 Set 对象的底层实现之一。整数集合本质上是一块连续内存空间

结构定义如下

~~~c
typedef struct intset {
    //编码方式
    uint32_t encoding;
    //集合包含的元素数量
    uint32_t length;
    //保存元素的数组
    int8_t contents[];
} intset;
~~~

保存的元素看encoding的类型

encoding属性有 INTSET_ENC_INT16， INTSET_ENC_INT32，INTSET_ENC_INT64

如果加入的元素属性比当前的encoding大，就要整数集合升级，扩展空间内存

<img src="https://cdn.xiaolincoding.com//mysql/other/e84b052381e240eeb8cc97d6b729968b.png" alt="img" style="zoom: 50%;" />

一旦升级，不能降级

### 跳表

Redis 只有 Zset 对象的底层实现用到了跳表，跳表的优势是能支持平均 O(logN) 复杂度的节点查找

每两个相邻元素，第一个元素往上建一个冗余元素来建立索引，提高查找的性能

**实现一个多层的有序链表**

插入和删除都是（logn），就是用空间换时间

三级跳表

<img src="https://cdn.xiaolincoding.com//mysql/other/2ae0ed790c7e7403f215acb2bd82e884.png" alt="img" style="zoom: 50%;" />

![img](https://cdn.xiaolincoding.com/gh/xiaolincoder/redis/%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B/3%E5%B1%82%E8%B7%B3%E8%A1%A8-%E8%B7%A8%E5%BA%A6.drawio.png)

### quicklist

quicklist 就是「双向链表 + 压缩列表」组合，因为一个 quicklist 就是一个链表，而链表中的每个元素又是一个压缩列表

![img](https://cdn.xiaolincoding.com//mysql/other/f46cbe347f65ded522f1cc3fd8dba549.png)

在向 quicklist 添加一个元素的时候，不会像普通的链表那样，直接新建一个链表节点。而是会检查插入位置的压缩列表是否能容纳该元素，如果能容纳就直接保存到 quicklistNode 结构里的压缩列表，如果不能容纳，才会新建一个新的 quicklistNode 结构

## listpack

目的是替代压缩列表，解决连锁更新的问题，没有指向前端长度的数据结构

![img](https://cdn.xiaolincoding.com//mysql/other/c5fb0a602d4caaca37ff0357f05b0abf.png)

**listpack 没有压缩列表中记录前一个节点长度的字段了，listpack 只记录当前节点的长度，当我们向 listpack 加入一个新元素的时候，不会影响其他节点的长度字段的变化，从而避免了压缩列表的连锁更新问题**。

### Redis的缓存读写策略

有三种

**Cache Aside Pattern（旁路缓存模式）**

先更新数据库再删除缓存

**Read/Write Through Pattern（读写穿透）**

Read/Write Through Pattern 中服务端把 cache 视为主要数据存储，从中读取数据并将数据写入其中。cache 服务负责将此数据读取和写入 db，从而减轻了应用程序的职责，而不是应用

![img](https://img-blog.csdnimg.cn/img_convert/bf35f95b0b32c8d895c327b45250ad33.png)

![img](https://img-blog.csdnimg.cn/img_convert/54418c5f11b7a9fb65da1f0edb61ea96.png)

#### Write Behind Pattern（异步缓存写入）

也是由 cache 服务来负责 cache 和 db 的读写。**而 Write Behind 则是只更新缓存，不直接更新 db，而是改为异步批量的方式来更新 db。**读写穿透是同步更新

### Redis的数据结构

![img](https://cdn.xiaolincoding.com//mysql/other/9fa26a74965efbf0f56b707a03bb9b7f-20230309232459468.png)
