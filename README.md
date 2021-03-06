## 常见问题

### Redis的基本数据类型有哪些？

共有八种类型，String，List，Hash，Set，SortSet，HyperLogLog，Geo，Pub/Sub。

### 如果有大量的Key要设置在同一时间失效，一般要注意什么？

注意过期时间不要过于集中，防止出现Redis卡顿甚至雪崩。 可以为每个Key的失效时间加一个随机值，使得他们不在同一时间失效。

### 你使用过Redis分布式锁么，他是怎么回事？

使用过，我曾经使用`SETNX`指令+Spring表达式实现并应用的分布式锁来防止表单重复提交。 Redis的SETNX命令类似于Zookeeper中的创建文件的操作，是一个独占的操作，多个客户端操作，只有一个客户端能够成功，其他会失败，
利用这个特性可以来感知谁抢占到了资源，从而实现分布式锁。

### 缓存穿透和击穿，与雪崩有什么区别么，如何应对他们？

**缓存击穿**：缓存中某个热点Key失效瞬间，请求直接涌入数据库，对数据库的开销影响造成了可见的影响。

**缓存穿透**：在数据请求时候，缓存层数据库层都没有找到符合条件的的数据。

**缓存雪崩**：缓存中大面积的key（不止一个）失效，使得超大量请求直接转向数据库层，数据库开销突然陡增，使得数据库崩溃。

对于解决他们的办法如下：

**解决缓存穿透**：

1. 在未请求未达到缓存层和DB层之前做各类校验，如：鉴权校验，参数校验等，直接拦截不合法参数请求，对于重复请求适当限流。
2. 对于在数据库也没有查询到数据，在缓存中给一个null值，并设定一个较短的过期时间来减缓数据库压力。
3. 对于那些使用单个key值，进行超过阈值超高频次请求的ip，直接在nginx上拉黑，因为正常用户一定不会达到阈值频率。
4. 使用布隆过滤器（Bloom Filter）来检查key对应的value是否在BD中也存在，如果存在，才去DB中查询，否则直接返回null。

**解决缓存击穿**：

1. 对于非严格一致性的数据，可以将热点数据设置为永不过期，然后再使用异步线程进行更新缓存。
2. 对于那些一定要过期的数据，可以加上一个互斥锁，使得当缓存中的数据已经为空的时候，只有一个线程可以通过DB来获取数据并且更新到缓存。

**解决雪崩**：

1. 既然数据是热点数据，如果数据可以永不过期，则将数据设置为永不过期。
2. 如果热点数据没有严格的过期时间规定，则在热点数据集原来的过期时间基础上加上一个1-3分钟的随机时间，使得热点数据集中的数据不会在同一个时间失效，降低数据库的瞬时压力。
3. 双缓存模式，设置A,B两个缓存，B缓存总是随着数据库进行更新，当A中没有找到数据时候，再从B去获取，同时更新到A缓存。
4. 对请求进行降级或者限流处理，使得不会有非常的数据库请求。

### 布隆过滤器（Bloom Filter）解决缓存穿透的工作原理是什么？

### Redis为什么这么快？

Redis是一个基于内存的采用单线程的模型的KV数据库，由C语言编写，官方测试单机可以达10万+QPS。

1. 基于内存，绝大部分的请求操作都是基于内存，而且他的内部存储结构类似于HashMap，所以时间复杂度为O(1)。
2. 数据结构简单，没有复杂的数据结构，Redis中的数据结构也是专门去设计的。
3. 采用单线程，避免不必要的上下文切换和竞争条件，不用考虑锁的问题。
4. 使用多路复用IO，非阻塞IO模型。
5. 优化与底层的通信，使得更快。

### Redis的持久化方式有哪些？有什么优缺点？

有RDB模式和AOF模式两种持久化模式。

1. RDB（Redis Database）就是在指定的时间间隔执行Redis的快照。
2. AOF（Append Only File）,这种模式下，Redis会记录每一个写操作，并追加到AOF文件中，当Redis重新启动的时候，会从AOF文件中以此读入这些操作。
3. 也可以设置RDB+AOF，则是以追加方式记录修改操作，同时定期对内存中的数据进行快照，但是Redis重启时候会优先从AOF记录中重建数据集。

**RBD优点**：

1. 是一个结构紧凑的单文件，代表了某一个时刻的Redis数据，非常适合冷备份。
2. RBD快照时，是fork了一个子进程进行操作，所以对Redis的性能影响小。
3. 对于紧急重启而言，紧凑的RBD文件相对于AOF可以支持更大的数据集重启。

**RBD缺点**：

1. RDB快照时间间隔一般较长，所以重启恢复之后与之前数据有较大的时间差，而AOF的时间差可能是秒级的。
2. 如果数据集十分庞大，而机器的剩余CPU性能足，可能会导致客户端暂停数毫秒甚至数秒。
3. 因为IO时间长，且文件是一个整体所以更加容易在断电时候出现不可恢复的文件损坏。

**AOF优点**：

1. 因为AOF是追加的，可以通过调整fsync策略来设置刷盘时间，一般的刷盘时间设置1s，使得AOF文件与真实数据集的时间差只有1s。
2. 因为AOF是一个追加文件，不容易损坏（比如写入时断电）。
3. AOF文件是一个具有可阅读性的文件，我们甚至可以直接用文本阅读器打开。

**AOF缺点**：

1. AOF文件不是紧凑的结构，所以文件比RDB更大。
2. 因为每次在修改数据的时候，都需要去追加AOF记录，另外每秒还要进行刷盘操作，所以对Redis的读写性能影响大于RDB备份。

### 什么是哨兵模式（Sentinel）？它有什么作用，基本原理是什么？

哨兵模式是Redis提供的一种高可用的集群主从模式，集群用于一个Master节点，其他节点均为Slave节点，通过主从复制的能力使得集群具有读写分离的特性， 通过自动选举能力使得集群具有故障转移能力。

在哨兵模式下，每一台机器都是一个哨兵，哨兵提供了如下

### Redis Sentinel的主从切换过程是怎么样的？

1. S1，S2，S3通过sentitnel is-master-down-by-addr命令并附带自己的ip-port来请求其他sentinel为自己投票。
2. 一个Sentinel收到一个被投票的Sentinel的投票请求后，在以下两种情况会将自己的票投给被投票的Sentinel。
    1. Sentinel自己的配置纪元小于或等于被投票的Sentitnel的配置纪元。
    2. 在当前配置纪元中，这是当前Sentinel收到的第一个投票请求。

### Redis中各个数据类型都有什么应用场景？

1. String：
    - 缓存，比如缓存用户的请求参数等。
    - 计数，比如点赞树，喜欢数，收藏数等，快速查询，快速修改，异步落地到数据库等。
    - 共享Session，Token等，比如在分布式程序中实现单点登录等。

2. List：
    - 存储粉丝列表，文章列表等。
    - 可以通过lrange实现快速分页查询。
    - 可以利用FIFO的特性实现一个消息队列。

3. Hash：
    - 主要是存储对象嵌套结构的数据。

4. Set：
    - 自动去重功能，特别是数量特别大时候，不好在JVM上操作的时候。
    - 利用集合的交并补性质，实现诸如共同好友等功能。

5. Sorted-Set：
    - 排行榜，可以按照多维度进行排名。
    - 可以利用权重制作优先任务队列，使得客户端可以根据任务优先级进行执行。

### 如果多个线程同时操作某个Redis Key，如何保证顺序性？

1. 对于下单，支付，退款这种需要顺序的操作，写入操作可以使用zookeeper等提供的锁操作将并发操作顺序化。
2. 对于刷新缓存这种操作，可以在待写入数据中标记版本号（诸如时间错等），如果要写入的数据的版本号比已经存在于Redis中的数据的版本号小，则不写入。

### 什么是双写一致性？

是指某个需要更新的数据，既需要更新缓存，也需要更新数据库，并且要求更新后，其他用户从缓存查询到的数据要和数据库中的数据一致性。

### 你知道对于双写情况下，有哪些双写模式么？

有以下3个模式：

1. Cache Aside Pattern（旁路缓存模式）：
    1. 读流程：先读缓存，未命中则读数据库并更新缓存然后返回。
    2. 写流程：先写数据库，然后删除缓存。
2. Read Through/Write Through（读写穿透模式）：
    1. 读流程：先读缓存，未命中则读数据库并更新到缓存然后返回。
    2. 写缓存：写数据库同时写缓存。
3. Write Behind（异步缓存写入）：
    1. 读流程：和读写穿透模式一样。
    2. 写流程：先写入缓存，然后等待批量写入数据库。

### 在Cache Aside Pattern下，为什么在更新时候是删除缓存而不是更新缓存？

无论什么模式下，我们都希望缓存能够尽量跟随数据库保持数据一致性。

1. 当更新数据非常复杂而庞大时候（比如涉及多张表，比如缓存需要复杂计算），更新缓存非常麻烦，但是缓存也不一定被使用。
2. 假如一个数据一分钟被更新了20次，但是缓存被使用的频率并不高。
3. 这一种懒加载思想，比如MyBatis等都有用。

### 在集群模式下，Redis的Key是如何寻址的？分布式寻址都有哪些算法？了解一致性 Hash 算法吗？

Redis采用了哈希槽（Hash Slot）的概念来解决分布式寻址问题。一个Redis集群被均等划分为16831个槽位，每个服务器会均等分配到槽位个数。

### Redis和Memcached有什么区别？

1. Memcached仅支持kv格式的存储，而Redis则支持5种基本类型和另外几种特殊数据类型存储。
2. Redis支持持久化复制等功能，而Memcached则是一个纯粹的内存缓存系统。
3. Memcached本身不支持分布式，需要在客户端使用类似一致性哈希算法这种分布式寻址算法来实现分布式缓存，而Redis则是在服务端自带多种分布式部署模式。



## 参考

- [Redis基础 - 敖丙](https://juejin.cn/post/6844903982066827277)
- [Redis常见问题 - 敖丙](https://zhuanlan.zhihu.com/p/91539644)
- [缓存问题和解决方案](https://medium.com/interviewnoodle/cache-problems-cache-penetration-cache-breakdown-cache-avalanche-9b866483e2b7)
- [Redis Sentinel主从切换源码解析](https://bbs.huaweicloud.com/blogs/246151)