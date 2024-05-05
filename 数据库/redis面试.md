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

### Redis缓存三兄弟

String（字符串）

- Redis 中最简单的数据类型，可以存储字符串、整数或者浮点数。
- String 类型是 Redis 的基本数据结构，可以执行诸如设置值、获取值、增加值等操作。
- 通常用于缓存简单的键值对数据，如用户会话信息、计数器等。

```
plaintextCopy codeSET key value
GET key
INCR key
```

List（列表）

- **列表类型是一个有序、可重复的数据结构，它可以包含多个元素。**
- 列表类型支持在列表的两端进行元素的插入和删除操作，可以用来实现队列、栈等数据结构。
- 通常用于缓存需要按顺序存储的数据，如消息队列、最新消息列表等。

```
plaintextCopy codeLPUSH key value1 value2 ...   // 从列表左侧插入元素
RPUSH key value1 value2 ...   // 从列表右侧插入元素
LPOP key                      // 从列表左侧删除元素
RPOP key                      // 从列表右侧删除元素
```

Hash（哈希）

- **哈希类型是一种键值对的无序集合，其中每个键都映射到一个值。**
- 哈希类型适合用于存储对象的字段和值的映射关系，可以方便地对对象的字段进行操作。
- 通常用于缓存对象的属性或元数据信息，如用户信息、商品信息等。

```
plaintextCopy codeHSET key field value          // 设置哈希字段的值
HGET key field                // 获取哈希字段的值
HDEL key field1 field2 ...    // 删除一个或多个哈希字段
```

这三种数据结构在 Redis 中都有广泛的应用，它们提供了不同的数据操作接口，适用于不同的缓存需求场景。
