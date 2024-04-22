Redis NOSQL因为是在内存中执行，所以性能较好，运行快
命令行
redis-cli 进入本地redis端口
auth password 输入redis密码

Redis的java客户端
一般用Jedis和Lettuce，而后面SpringData是Spring中数据操作的模块，包含对各种数据库的集成，其中对Redis的集成模块就叫做SpringDataRedis
它整合了jedis和Lettuce
SpringDataRedis中提供了RedisTemplate工具类，其中封装了各种对Redis的操作。并且将不同数据类型的操作API封装到了不同的类型中：

SpringDataRedis中提供了RedisTemplate工具类，其中封装了各种对Redis的操作。并且将不同数据类型的操作API封装到了不同的类型中：

![](C:\Users\86147\Desktop\Redis点评\Redis点评\01-入门篇\讲义\Redis注释版\assets\UFlNIV0.png)

并且规定存入Redis的类型通用为String，这就要求我们在存入之前对对象进行手动序列化
