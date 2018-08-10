---
title: Redis 命令学习
date: 2018-08-10 14:59:54
tags: Redis
---

## Redis 命令

### 键值对

- **set** key value 设置键值对
- **mset** key1 value1 key2 value ... 批量设置键值对
- **exists** key  检查键是否存在
- **del** key 删除键
- **get** key 获取键的值
- **mget** key1 key2 批量获取键的值，返回为list
- **setex** key expireTime value 设置键的同时设置值
- **expire** key expireTime 给key 设置过期时间 expireTime单位为s
- **setnx** key value 如果key 不存在就执行set，存在则不执行，（使用场景：分布式锁）
- **incr** key  对key的值自增1
- **incrby** key value 对key的值增加vlaue个数，vlaue 为负数时为减

### List 列表

> Redis 的列表相当于Java语音中的LinkedList，List 是链表而非数组
>
> List数据量较小时使用ziplist，数据量大之后数据结构为quicklist ，quicklist 由链表和ziplist组成
>
> 插入、删除时间复杂度均为O(1)，定位索引时间复杂度为O(n)	

#### 队列

> 右边进左边出

- **rpush** key value （或:**rpush** key value1 value2 value3） 向队列中推入值
- **llen** key  计算队列长度
- **lpop** key 从左往右弹出队列的值

#### 栈

> 右边进右边出

- **rpop** key 从右往左弹出队列的值

#### 慢操作

- **lindex** key index 从队列中获取相应index的值，相当于Java链表的get(int index)
- **ltrim**  key start_index end_index 截取保留key从start_index 至end_index的值（左右均闭区间，区间内保留），可用于实现定长链表

### Hash 字典

> Redis字典相当于Java的HashMap,无序字典。实现方式上采用数组+链表的二维结构。
>
> 第一维hash的数组位置碰撞时，就会将碰撞的元素使用链表串起来
>
> **值只能是字符串**

- **hset** key hash_key hash_value 设置hash的key和字典中key value
- **hgetall**  key 获取key对应的hash,返回值为list，hash的key 和value间隔出现
- **hlen** key 获取该key有多少个hash_key
- **hget**  key hash_key 获取某个key下面的hash_key的值
- **hmset**  key hash_key1 hash_value1 hash_key2 hash_value2 批量设置
- **hmget**  key hash_key1 hash_key2  批量获取
- **hincrby** key hash_key1  value 对单个hash_key下面的值进行加减

### Set 集合

> 值无序且唯一

- **sadd** key value 向key集合中添加value值，成功返回1，失败返回0

- **smembers**  key 获取key全部值

- **sismember** key value 查询key集合中是否存在value这个值，存在返回1，反之返回0，相当于contains （）

- **scard** key 获取key的长度，相当于count()

- **spop** key 从key中弹出一个值

   

### Zset (有序列表)

> zset 可能是 Redis 提供的最为特色的数据结构，它也是在面试中面试官最爱问的数据结构。 
>
> 一方面它是一个 set，保证了内部 value 的唯一性，另一方面它可以给每个 value 赋予一个 score，代表这个 value 的排序权重。它的内部实现用的是一种叫着「跳跃列表」的数据结构。 

- **zadd** key score value 添加给key添加某个vlaue
- **zrange** key start end 按 score 排序列出，start、end为参数区间为排名范围 
- **zrevrange** key start end 按 score 逆序列出，start、end为参数区间为排名范围 
- **zcard** key 相当于 count() 
- **zscore**  key value 获取指定 value 的 score 
- **zrank**  key value 获取value的排名
- **zrangebyscore**  key start end withscores  根据分值区间 (start, end] 遍历 zset，同时返回分值。start 和end 可以使用inf 代表 infinite，无穷大的意思。 withscores 表示返回分值
- **zrem**  key value 删除 key的某个 value 
- 

### HyperLogLog

> 主要应用于PV统计，具有自动去重功能
>
> HyperLogLog 提供不精确的去重计数方案，虽然不精确但是也不是非常不精确，标准误差是 0.81% 
>
> 
>
> 该数据结构需要占据12k的存储空间，不适用于为单个用户计数，相比set内存空间占用太大。在数据量较小时，存储空间采用稀疏矩阵此时占用的空间比较小，当计数慢慢变大后，稀疏矩阵的占用空间超过了阈值时才会一次性转变为	稠密矩阵，此时稠密矩阵占用12k内存。
>
> 

- **pfadd** key value 给某个key添加value，对value有自动去重功能，与set的sadd功能相似
- **pfcount** key 统计该key有多少值，与set的scard功能相似
- **pfmerge**   用于将多个 pf 计数值累加在一起形成一个新的 pf 值。 

### 布隆过滤器

> HyperLogLog无法实现查询某个值是否存在问题，布隆过滤器边解决了该问题
>
> 布隆过滤器的查询不一定100%准确，当某个值在布隆过滤器当中的时候那查询该值肯定准确，但某个值不在过滤器中，过滤器可能会误判为在过滤器中
>
> 

### 简单限流

```python
# 指定用户 user_id 的某个行为 action_key 在特定的时间内 period 只允许发生一定的次数 max_count
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
    return True

```

### 漏斗限流

#### 算法思想

```python
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
def is_action_allowed(user_id, action_key, capacity, leaking_rate):
    key = '%s:%s' % (user_id, action_key)
    funnel = funnels.get(key)
    if not funnel:
        funnel = Funnel(capacity, leaking_rate)
        funnels[key] = funnel
    return funnel.watering(1)
```

Redis可将以上算法实现为分布式漏洞限流，Redis 4.0 提供了一个限流 Redis 模块，它叫 redis-cell。该模块也使用了漏斗算法，并提供了原子的限流指令。 

- cl.throttle key capacity  operations  seconds  quota  

  该指令的参数分别为：

  - key: 对应的key
  - capacity  初始容量，当一分钟执行动作超过该容量才会受限流速率影响
  - seconds  时间范围
  - operations  时间范围seconds内的动作数量的值，operations/seconds  代表速率
  - quota   可选参数，默认为1，每次动作的数量

  > \> cl.throttle laoqian:reply 15 30 60 
  >
  > 1) (integer) 0   # 0 表示允许，1表示拒绝 
  >
  > 2) (integer) 15  # 漏斗容量capacity 
  >
  > 3) (integer) 14  # 漏斗剩余空间left_quota 
  >
  > 4) (integer) -1  # 如果拒绝了，需要多长时间后再试(漏斗有空间了，单位秒) 
  >
  > 5) (integer) 2   # 多长时间后，漏斗完全空出来(left_quota==capacity，单位秒)

### GeoHash

> 地图元素的位置数据使用二维的经纬度表示，经度范围 (-180, 180]，纬度范围 (-90, 90]，经度正负以赤道为界，北正南负，纬度正负以本初子午线 (英国格林尼治天文台) 为界，东正西负。 
>
> 

- **geoadd** key  latitude longitude zkey  在key上添加zkey的地理信息
- **geodist**  key zkey1 zkey2 km 计算zkey1与zkey2的距离以km形式显示。距离单位可以是 m、km、ml、ft，分别代表米、千米、英里和尺。 
- **geopos**   key zkey (zkey1 zkey2) 获取集合总zkey元素的经纬度坐标
- **geohash** key zkey 获取元素zkey的经纬度编码字符串, http://geohash.org/${hash} 
- **georadiusbymember** 指令是最为关键的指令，它可以用来查询指定元素附近的其它元素，它的参数非常复杂。 
  - **georadiusbymember** key zkey 20 km count 3 asc   范围 20 公里以内最多 3 个元素按距离正排，它不会排除自身
  - **georadiusbymember** key zkey 20 km count 3 desc  范围 20 公里以内最多 3 个元素按距离倒排 
- **georadius** key  latitude longitude 20 km withdist count 3 asc  根据经纬度坐标找出20km范围内最多三个元素按距离正排

在一个地图应用中，车的数据、餐馆的数据、人的数据可能会有百万千万条，如果使用 Redis 的 Geo 数据结构，它们将全部放在一个 zset 集合中。在 Redis 的集群环境中，集合可能会从一个节点迁移到另一个节点，如果单个 key 的数据过大，会对集群的迁移工作造成较大的影响，在集群环境中单个 key 对应的数据量不宜超过 1M，否则会导致集群迁移出现卡顿现象，影响线上服务的正常运行。 所以，这里建议 Geo 的数据使用单独的 Redis 实例部署，不使用集群环境。 如果数据量过亿甚至更大，就需要对 Geo 数据进行拆分，按国家拆分、按省拆分，按市拆分，在人口特大城市甚至可以按区拆分。这样就可以显著降低单个 zset 集合的大小。... https://juejin.im 掘金 — 一个帮助开发者成长的社区 

## 原理

### 线程IO模型

#### 问题

- **Redis 单线程为什么还能这么快？** 

  > 因为它所有的数据都在内存中，所有的运算都是内存级别的运算。正因为 Redis 是单线程，所以要小心使用 Redis 指令，对于那些时间复杂度为 O(n) 级别的指令，一定要谨慎使用，一不小心就可能会导致 Redis 卡顿。 

- **Redis 单线程如何处理那么多的并发客户端连接？** 

  > **事件轮询 (多路复用)**
  > **非阻塞 IO**

#### 非阻塞 IO

