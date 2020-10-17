1. 缓存的更新模式

[连接](https://user-gold-cdn.xitu.io/2018/5/11/1634fc3319beda97?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

# 2. 这段代码有什么问题
```
$ok = $redis->setNX($key, $value);
if ($ok) {
    $cache->update();
    $redis->del($key);
}
```
# 跳跃表和双向链表的区别？
1. 跳跃表的查询效率更好
2. 双向链表的话，查询效率没有那么好


# 1. redis遇到过什么坑？
1. 内存被使用满了


# 布隆过滤器
1. 对一个值，进行多次hash，然后将对应位置的hash值置为1，就代表着个值，同时查询的时候计算多次，然后只要有一个位置为0，说明就是不存在。


# rehash
因为插入和删除，当数量太多或者太少的时候，就会进行rehash，为了让hash表达的负载因子在一个合理的范围内。

需要扩展时，需要将ht[1] = ht[0].used *2 的 2^n

需要收缩时，需要将ht[1]= ht[0].used*2 的    2^n

然后将 ht[0]  置为空，ht1[1] = ht[0]， 然后再创建新的ht[1] 为下一次rehash做准备

### 哈希表的扩展与收缩

当没有bgsave 和BGREWRITEAOF 的时候，负载因子为1

当有bgsave 和bswriteaof的时候，负载因为大于5

负载因子 = 哈希表已保存节点数量 / 哈希表大小
load_factor = ht[0].used / ht[0].size
- 渐进式rehash

- 不是一次性，集中式的执行，而是分批次，渐进执行
	- 因为如果键值数量过大的话，会导致阻塞，在一段时间内不能提供服务
	- 采用的是分而治之的方式来执行的
	- 索引期间，查询的话，会先在ht[0] 上面查询，然后新家的话，只会在ht[1]上面新加，这样的话，会保证ht[0] 上面的数据只增不减
# 限流，滑动窗口

```
# coding: utf8

import time
import redis

client = redis.StrictRedis()
def is_action_allowed(user_id, action_key, period, max_count):
    key = 'hist:%s:%s' % (user_id, action_key)
    now_ts = int(time.time() * 1000)  # 毫秒时间戳
    with client.pipeline() as pipe:  # client 是 StrictRedis 实例
        # 记录行为
        pipe.zadd(key, now_ts, now_ts)  # value 和 score 都使用毫秒时间戳
        # 移除时间窗口之前的行为记录，剩下的都是时间窗口内的
        pipe.zremrangebyscore(key, 0, now_ts - period * 1000)
        # 获取窗口内的行为数量
        pipe.zcard(key)
        # 设置 zset 过期时间，避免冷用户持续占用内存
        # 过期时间应该等于时间窗口的长度，再多宽限 1s
        pipe.expire(key, period + 1)
        # 批量执行
        _, _, current_count, _ = pipe.execute()
    # 比较数量是否超标
    return current_count <= max_count
for i in range(20):
    print is_action_allowed("laoqian", "reply", 60, 5)
```
# 漏斗限流

```
# coding: utf8
import time
class Funnel(object):
    def __init__(self, capacity, leaking_rate):
        self.capacity = capacity  # 漏斗容量
        self.leaking_rate = leaking_rate  # 漏嘴流水速率
        self.left_quota = capacity  # 漏斗剩余空间
        self.leaking_ts = time.time()  # 上一次漏水时间

    def make_space(self):
        now_ts = time.time()
        delta_ts = now_ts - self.leaking_ts  # 距离上一次漏水过去了多久
        delta_quota = delta_ts * self.leaking_rate  # 又可以腾出不少空间了
        if delta_quota < 1:  # 腾的空间太少，那就等下次吧
            return
        self.left_quota += delta_quota  # 增加剩余空间
        self.leaking_ts = now_ts  # 记录漏水时间
        if self.left_quota > self.capacity:  # 剩余空间不得高于容量
            self.left_quota = self.capacity

    def watering(self, quota):
        self.make_space()
        if self.left_quota >= quota:  # 判断剩余空间是否足够
            self.left_quota -= quota
            return True
        return False

funnels = {}  # 所有的漏斗
# capacity  漏斗容量
# leaking_rate 漏嘴流水速率 quota/s
def is_action_allowed(
        user_id, action_key, capacity, leaking_rate):
    key = '%s:%s' % (user_id, action_key)
    funnel = funnels.get(key)
    if not funnel:
        funnel = Funnel(capacity, leaking_rate)
        funnels[key] = funnel
    return funnel.watering(1)

for i in range(20):
    print is_action_allowed('laoqian', 'reply', 15, 0.5)
```

# 最初级的缓存不一致问题及解决方案
问题：先更新数据库，再删除缓存。如果删除缓存失败了，那么会导致数据库中是新数据，缓存中是旧数据，数据就出现了不一致。

解决思路：先删除缓存，再更新数据库。如果数据库更新失败了，那么数据库中是旧数据，缓存中是空的，那么数据不会不一致。因为读的时候缓存没有，所以去读了数据库中的旧数据，然后更新到缓存中。

# 过期
提供了几种可选策略
- noeviction 不会继续服务写请求 (DEL 请求可以继续服务)，读请求可以继续进行。这样可以保证不会丢失数据，但是会让线上的业务不能持续进行。这是默认的淘汰策略。
- volatile-lru 尝试淘汰设置了过期时间的 key，最少使用的 key 优先被淘汰。没有设置过期时间的 key 不会被淘汰，这样可以保证需要持久化的数据不会突然丢失。
- volatile-ttl 跟上面一样，除了淘汰的策略不是 LRU，而是 key 的剩余寿命 ttl 的值，ttl 越小越优先被淘汰。
- volatile-random 跟上面一样，不过淘汰的 key 是过期 key 集合中随机的 key。
- allkeys-lru 区别于 volatile-lru，这个策略要淘汰的 key 对象是全体的 key 集合，而不只是过期的 key 集合。这意味着没有设置过期时间的 key 也会被淘汰。
- allkeys-random 跟上面一样，不过淘汰的策略是随机的 key。

# 单台机器的话，使用.
set  nx就可以了。
需要考虑加锁时间过长的问题。
还要考虑加锁之后，进程突然挂掉的话，锁无法释放。所以，使用setnx 的时候注意一下。

还有就是在分布式的时候，在master上面加了锁之后master挂掉之后，会被其他进程给加锁，这个时候就出现了多个锁的情况。

这种情况下， redlock算法就出来了。就是会向所有的实例去发起加锁的请求，只要过半以上同意了就算加锁成功了，此时，向所有节点发送del的请求。

# ROB的原理
copy on write。父进程会fork一个子进程，父进程和子进程共享内存空间。父进程继续提供读写服务，写脏的页面数据会继续和子进程区分开来。

你给出两个词汇就可以了，fork和cow。fork是指redis通过创建子进程来进行RDB操作，cow指的是copy on write，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，写脏的页面数据会逐渐和子进程分离开来。
# redis 常用的数据结构【2月18好未来】【2月20滴滴】
- string 可以用来计数，缓存，分布式锁
- hash 可以用来保存用户的一些属性信息，用户的详情页
- list 可以用来做队列,可以用来做栈，可以用来做数据， 可以维护一个评论列表。lrange区间操作。
- set 可以用来做 交集 并集 差集，微博抽奖，随机事件问题。无序、去重
- sorted set  可以用来做排行榜，带权重的队列。
- bitmap  用来保存用户的登录信息，可以查询最近几个月的登录情况：bitop 可以用来做and or意味着有更多的选择。
- pub/sub 发布订阅
- stream
- hyperloglog  布隆过滤器器（滑动窗口）

stream的备注：支持多播的可持久化的消息队列。
Stream 的消费模型借鉴了 Kafka 的消费分组的概念，它弥补了 Redis Pub/Sub 不能持久化消息的缺陷。但是它又不同于 kafka，Kafka 的消息可以分 partition，而 Stream 不行。如果非要分 parition 的话，得在客户端做，提供不同的 Stream 名称，对消息进行 hash 取模来选择往哪个 Stream 里塞。
# 2. redis五种基本数据结构的底层实现？【2月18好未来】
sorted set 的底层数据结构：ziplist 和skiplist 吧。

https://mp.weixin.qq.com/s/GLqZf-0sLQ7nnJ8Xb9oVZQ
# redis常用的数据类型？
- string =》 int string
- hash =》  ziplist hashable
- zset  =》 ziplist  skiplist
- set  intset hash
- list   ziplist linkedlist

# 3. redis 为什么快
 因为全部是内存操作，纯是内存操作，查找和操作的时间都是O(1)

单线程操作避免了上下文切换。不存在多线程和多进程导致的切换和消耗CPU。不用考虑各种锁问题

可以将一些事务由并行改为串行。

使用的是非阻塞io，

是使用自己的特定的数据结构，且数据结构比较简单，

完全基于内存，绝大部分请求是纯粹的内存操作，非常快速。

--它的数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；--

完全基于内存，查询操作是o(1)；数据结构简单，对数据操作也简单，Redis中的数据结构是专门进行设计的；数据结构简单；采用单线程，避免了不必要的上下文切换和竞争条件，也不存在多进程或者多线程导致的切换而消耗CPU，不用去考虑各种锁的问题，不存在加锁释放锁操作，没有因为可能出现死锁而导致的性能消耗；采用单线程；避免了上下文切换；不用考虑各种锁的问题；使用多路I/O复用模型，非阻塞IO；使用底层模型不同，它们之间底层实现方式以及与客户端之间通信的应用协议不一样，Redis直接自己构建了VM 机制，因为一般的系统调用系统函数的话，会浪费一定的时间去移动和请求；
自己搭建了MV机制
# 4. select epoll
select 是每次去拿文件描述符去查，看哪些符合条件，然后去执行

epoll 是采用事件监听的形式，只会执行符合条件的事件。
## 5. redis 分布式锁
采用set          nx 就可以了。最好设计过期时间，防止锁不释放的情况。

```
// 获取锁 // NX是指如果key不存在就成功，key存在返回false，PX可以指定过期时间 
SET anyLock unique_value NX PX 30000 
// 释放锁：通过执行一段lua脚本 
// 释放锁涉及到两条指令，这两条指令不是原子性的 
// 需要用到redis的lua脚本支持特性，redis执行lua脚本是原子性的 
if redis.call("get",KEYS[1]) == ARGV[1] then 
    return redis.call("del",KEYS[1]) 
else 
    return 0 
end
```


因为redis大部分是单机部署，如果master加锁成功之后，突然宕机了怎么办呢? 会出现锁丢失的情况。
这个时候就得需要redlock锁了。只要大多数加锁成功了，就算是加锁成功了。只要加锁时间小于当前时间，就是加锁成功了。其他的节点就得需要不断的来轮询了。

# 6. redis 的集群。
redis cluster 着眼于可扩展。当单个redis不足时，使用cluster进行分片存储。。

来看 Redis 的高可用。Redis 支持主从同步，提供 Cluster 集群部署模式，通过 Sentine l哨兵来监控 Redis 主服务器的状态。当主挂掉时，在从节点中根据一定策略选出新主，并调整其他从 slaveof 到新主。
选主的策略简单来说有三个：
- slave 的 priority 设置的越低，优先级越高；
- 同等情况下，slave 复制的数据越多优先级越高；
- 相同的条件下 runid 越小越容易被选中。
在 Redis 集群中，sentinel 也会进行多实例部署，sentinel 之间通过 Raft 协议来保证自身的高可用。

Redis Cluster 使用分片机制，在内部分为 16384 个 slot 插槽，分布在所有 master 节点上，每个 master 节点负责一部分 slot。数据操作时按 key 做 CRC16 来计算在哪个 slot，由哪个 master 进行处理。数据的冗余是通过 slave 节点来保障。

# 7. redis的Sentinal哨兵模式。
就是高可用，master宕机之后，会将slave提升为master继续提供服务。

哨兵组件的作用：
- 集群监控：负责监控 Redis master 和 slave 进程是否正常工作。
- 消息通知：如果某个 Redis 实例有故障，那么哨兵负责发送消息作为报警通知给管理员。
- 故障转移：如果 master node 挂掉了，会自动转移到 slave node 上。
- 配置中心：如果故障转移发生了，通知 client 客户端新的 master 地址。

# 8. redis 的持久化,以及丢数据的问题。
rdb 是做的全量的快照，rdb是fork一个子进程去做持久化的。【可以做冷备份，一旦挂了想恢复多久之前的数据，直接拷贝一份就好了】如果丢失数据的话，丢失数据还是比较多的。

aof 就是做增量添加。可以设置持久化的频率。也可以手动在低频率请求的时候做持久化。【适合做热备】后台一秒执行一次fsync的话，丢失的最多也就一秒的的数据。aof是一个非常可读的数据。可以通过修改aof文件，来恢复之前的数据。

redis在重启的时候，会优先使用aof，因为aof要比rdb完整。

rdb丢数据的话，丢失的数据还是比较多的。

aof的话，丢数据的话，丢失的还是比较少的。

# 9.redis 常见的数据结构有哪些，各自的使用场景
string 这个字段还是很常用的.
hash 文章详情(文章有很多属性,就获取)
list 消息队列
set 微博的共同好友
zset 点赞(有序集合,分值,是时间)
# 10. 缓存雪崩和缓存穿透
### 缓存雪崩。
如果大量缓存同时失效，导致流量打到数据库，会给数据库造成压力。

为了避免流量全部打到数据库，解决办法可以参考以下几种方式。

1. 给过期时间加一个随机值。使过期时间随机一些。比如1-3分钟的过期时间。
2. 如果是redis实例挂了的话，可以采用限流或者服务降级。针对非核心业务的话，直接降级，等服务恢复。针对核心业务，服务不能停的话，采用限流的方式，每1w次请求，只允许1000个请求通过。可以采用集群的方式，避免实例不可用的情况

### 缓存穿透：
就是一直访问不存在的key，绕过缓存。这种情况的解决办法就是使用布隆过滤器。

1. 每次设置一个空缓存。这个值可以和业务方沟通好。
2. 使用布隆过滤器。若是缓存中，可以知道数据是否存在就可以直接响应返回。若缓存缺失的话，就去数据库中读取。
3. 前端可以做一些简单的检测，如果是恶意请求，直接过滤掉，不向后端发请求。

### 缓存击穿
 是一个热点的key，在这个key失效的瞬间，持续的高并发就突破了缓存，给数据库造成压力。好想在一个桶上面凿了一个洞。【让热点key的时间设置的长一点，应该就可以了吧】
可以将热点数据设置为永远不过期；

上面的现象是多个线程同时去查询数据库的这条数据，那么我们可以在第一个查询数据的请求上使用一个 **互斥锁** 来锁住它。其他的线程走到这一步拿不到锁就等着，等第一个线程查询到了数据，然后做缓存。后面的线程进来发现已经有缓存了，就直接走缓存。

# 11. keys读取。
key会堵塞。所以，最好还是使用scan来进行查询。scan可以分批查。
# 12.实现一个队列
lpush rpop。如果没有数据时，会怎么办？会一直阻塞。所以这种情况下，最好还是sleep一会。如果不sleep呢？那就使用brpop。非阻塞模式。
# 13.主从同步是怎么实现的
因为单机QPS是有上限的。

当启动一个slave的时候，他发送psync给master。如果是首次连接master，那么master会启动一个线程，进行全量的rdb快照，然后发送给slave。然后把新的请求写到缓存里面，然后slave会执行rdb文件，然后写到自己的本地。然后再读取master里面新增的请求。
# 14. redis删除方式
惰性删除。
定期删除。

惰性删除，避免对每个键，维护删除机制，查询时过期的健值，返回空。无法删除过期但不删除的健

定时删除，10s运行一次，快慢模式删除
# 15. redis和memcache
- redis支持多种数据结构，也就意味着支持多种操作。memcache支持string
- redis单线程，避免上下文切换。memcache是多线程。
- redis支持集群。
- redis支持持久化。【故障自动切换】
- 当key的数量超过100w时，memcache的性能要比redis要好。
- 能应对高并发， 将并行命令转为串行命令。
- mm最大key为1M 。如果key过大的话，建议使用redis。
- mm预分配内存，要比redis快一些。redis临时申请内存，会导致内存切片。
- redis提供计算，要比mm强一些。

# 16. redis 实现分布式锁的原理？基于什么特性
1. 单点的redis锁的话，就是使用 set nx 命令就可以实现
2. 就是使用redlock原理，就能实现吧，只要有一定数量的子节点同意之后，就能获取这个锁。
```
$servers = [
    ['127.0.0.1', 6379, 0.01],
    ['127.0.0.1', 6389, 0.01],
    ['127.0.0.1', 6399, 0.01],
];

$redLock = new RedLock($servers);

$lock = $redLock->lock('my_resource_name', 1000);

Array
(
    [validity] => 9897.3020019531
    [resource] => my_resource_name
    [token] => 53771bfa1e775
)
```

validity, an integer representing the number of milliseconds the lock will be valid.  代表能获取锁的时间，是毫秒级别的

resource, the name of the locked resource as specified by the user.代表着用户获取的一个明确的资源，就是key值

token, a random token value which is used to safe reclaim the lock. 一个随机的token值，用来安全的重申这个key
```
 $redLock->unlock($lock)
```

# Redlock算法
在Redis的分布式环境中，我们假设有N个Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。之前我们已经描述了在Redis单实例下怎么安全地获取和释放锁。我们确保将在每（N)个实例上使用此方法获取和释放锁。在这个样例中，我们假设有5个Redis master节点，这是一个比较合理的设置，所以我们需要在5台机器上面或者5台虚拟机上面运行这些实例，这样保证他们不会同时都宕掉。

为了取到锁，客户端应该执行以下操作:

1. 获取当前Unix时间，以毫秒为单位。
2. 依次尝试从N个实例，使用相同的key和随机值获取锁。在步骤2，当向Redis设置锁时,客户端应该设置一个网络连接和响应超时时间，这个超时时间应该**小于锁的失效时间**。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试另外一个Redis实例。
3. 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数（这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
4. 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。
如果因为某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功）。

# 脑裂问题是啥？
当客户端无法取到锁时，应该在一个随机延迟后重试,防止多个客户端在同时抢夺同一资源的锁（**这样会导致脑裂，没有人会取到锁**）。同样，客户端取得大部分Redis实例锁所花费的时间越短，脑裂出现的概率就会越低（必要的重试），所以，理想情况一下，客户端应该同时（并发地）向所有Redis发送SET命令。

需要强调，当客户端从大多数Redis实例获取锁失败时，应该尽快地释放（部分）已经成功取到的锁，这样其他的客户端就不必非得等到锁过完“有效时间”才能取到（然而，如果已经存在网络分裂，客户端已经无法和Redis实例通信，此时就只能等待key的自动释放了，等于被惩罚了）。

# redis 原子执行的命令
multi  + exec
```
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```

# 100个账号,每个账号每天最多能用100次,一个时间段只能用一个账号，设计一个算法
还没有想到办法？

# 超卖问题的解决？
1. 有一个key用来记录 库存数量
2. 用一个key用来记录下单的情况  每个订单保存30分钟，然后也记录库存id。
3. 然后去下单，去获得paycode   ，当库存id为负数时，生成paycode失败，不允许支付。并通知用户下单失败。并将库存加1.
4. 然后前段去支付，等待支付结果。在支付过程中失败的话，然后将库存加1，并通知用户支付失败。

对于普通库存来说，如果售完是否允许补货，如果能补货的话，补货就好了。对于不能补货的情况，那就是为负数的时候直接返回。
# redis的数据库是用什么数据结构进行存储的？
是dict。

# 超卖问题
如何避免超卖？如果在redis中扣减库存，可以利用decr命令扣减库存，decr是原子操作，在分布式环境下也不会有并发问题，decr扣减库存后，判断返回值，如果返回值小于0，扣减库存失败，秒杀也就失败了；如果在数据库中扣减库存可以在where后面加上库存大于0的条件，来避免库存被减成负值。这样就可以避免超卖情况发生了。

# redis的LRU删除机制
LRU 淘汰不一样，它的处理方式只有懒惰处理。当 Redis 执行写操作时，发现内存超出 maxmemory，就会执行一次 LRU 淘汰算法。这个算法也很简单，就是随机采样出 5(可以配置) 个 key，然后淘汰掉最旧的 key，如果淘汰后内存还是超出 maxmemory，那就继续随机采样淘汰，直到内存低于 maxmemory 为止。

# redis 底层存储也是使用的字典
也就是hashmap，基本随机性比较大，然后很少有冲突。

# 竟然问我redis set 和get为什么这么快
就是hash计算啊

# redis脑裂问题
解决方案
redis的配置文件中，存在两个参数
````
min-slaves-to-write 3
min-slaves-max-lag 10
````
第一个参数表示连接到master的最少slave数量

第二个参数表示slave连接到master的最大延迟时间

如果连接到master的slave数量小于第一个参数，且ping的延迟时间小于等于第二个参数，那么master就会拒绝写请求，配置了这两个参数之后，如果发生集群脑裂，原先的master节点接收到客户端的写入请求会拒绝，就可以减少数据同步之后的数据丢失。

# redis 连接函数 
pconnect connect

# redis rdb持久化机制【作业帮】

save 900 1  900秒（15分钟）内有1个更改就同步
save 300 10 300秒（5分钟）内有10个更改就同步
save 60 10000   60秒内有10000个更改就同步

# redis aof 持久化机制
appendfsync=always 每次同步 安全最高，最多丢失一个写入的数据
appendfsync=everysec 每秒同步 安全最折中，最多丢失1S的数据
appendfsync=no 安全最低，最多上一次保存AOF文件到当前时刻的全部数据

# 跳跃表的时间复杂度
一般是O(logn)  最差是 O(n)

# redis 集群

# codis 方案

集群模式—Codis

数据分片策略对客户端透明
Zookeeper作为name server
提供支持Redis协议的Proxy 
Proxy负责请求路由

# 跳跃表 插入和删除的时间复杂度


