---
title: Redis
---

# 参考文章

[Redis 命令](https://www.runoob.com/redis/redis-commands.html)

[♥Redis教程 - Redis知识体系详解♥](https://pdai.tech/md/db/nosql-redis/db-redis-overview.html)

[Redis 命令参考](http://doc.redisfans.com/)

[Redis常见问题和解决办法梳理](https://www.cnblogs.com/kevingrace/p/5569987.html)

# 数据类型

## HyperLoglog

**作用**：

基数统计。

**使用场景**：

- [UV（Unique Visitor，独立访客量）统计](#uvunique-visitor独立访客量统计)
- [分时段活跃用户数统计](#分时段活跃用户数统计)

**说明**：

每个 HyperLogLog 只需要花费 12KB 内存，就可以计算接近 2^64^ 个不同元素的基数。在基数比较小时采用稀疏矩阵存储，空间占用很小；仅在基数慢慢变大，稀疏矩阵占用空间超过阈值时，才会一次性转变为稠密矩阵，并占用 12KB 的空间。

HyperLogLog 只会根据输入元素来计算基数，而不会存储输入元素本身。

基础估计的结果是一个带有 0.81% 标准错误的近似值。

# 编码类型

<table>
   <tr>
      <th>数据类型</th><th>条件</th><th>编码类型</th>
   </tr>
   <tr>
      <th rowspan="3">String</th><td>值是长度大于 44 字节的字符串</td><td>OBJ_ENCODING_RAW</td>
   </tr>
   <tr>
      <td>值是长度 ≤ 44 字节的字符串</td><td>OBJ_ENCODING_EMBSTR</td>
   </tr>
   <tr>
      <td>值是可以用 long 类型表示的整数</td><td>OBJ_ENCODING_INT</td>
   </tr>
   <tr>
      <th>List</th><td></td><td>OBJ_ENCODING_QUICKLIST</td>
   </tr>
   <tr>
      <th rowspan="2">Hash</th>
      <td>保存的元素个数小于<code>hash-max-ziplist-entries</code>个，且每个元素长度小于<code>hash-max-ziplist-value</code>字节</td>
      <td>OBJ_ENCODING_ZIPLIST</td>
   </tr>
   <tr>
      <td>其他</td><td>OBJ_ENCODING_HT</td>
   </tr>
   <tr>
      <th rowspan="2">Set</th>
      <td>保存的所有元素都是整数，且数量不超过<code>set-max-intset-entries</code></td>
      <td>OBJ_ENCODING_INTSET</td>
   </tr>
   <tr>
      <td>其他</td><td>OBJ_ENCODING_HT</td>
   </tr>
   <tr>
      <th rowspan="2">Sorted Set</th>
      <td>保存的元素个数小于<code>zset-max-ziplist-entries</code>个，且每个元素长度小于<code>zset-max-ziplist-value</code>字节</td>
      <td>OBJ_ENCODING_ZIPLIST</td>
   </tr>
   <tr>
      <td>其他</td><td>OBJ_ENCODING_SKIPLIST</td>
   </tr>
   <tr>
      <th>Stream</th><td></td><td>OBJ_ENCODING_STREAM</td>
   </tr>
</table>

配置项默认值：

- hash-max-ziplist-entries：512
- hash-max-ziplist-value：64
- set-max-intset-entries：512
- zset-max-ziplist-entries：128
- zset-max-ziplist-value：64

```redis
# 查看配置
config get hash*
# 修改配置
config set hash-max-ziplist-entries 256
# 查看值编码类型
object encoding <key>
```

# 事务

Redis 的事务不具备原 ACID 特性，它仅仅是打包的批量执行的脚本，脚本中某条命令执行失败并不会回滚已执行成功的命令，也不会中止剩余命令的执行。

# 运行机制

Redis 中，**命令处理**由**主线程**（**单线程**）完成。

先后出现了 3 个后台线程，用于处理慢操作：

- close_file：关闭 AOF、RDB 等持久化过程中产生的大临时文件。
- aof_fsync：将追加至 AOF 文件的数据刷盘。
   > 一般情况下 [write](#write) 调用之后，数据被写入 Page Cache，通过调用 [fsync](#sync) 才能将数据刷盘。
- lazy_free：惰性释放已删除的大对象占用的内存空间。

6.0 版本开始，增加了 **IO 线程**（多个）来解决网络模块由单线程 [IO 多路复用导致的瓶颈](TODO)（数据量较大时阻塞时间较长，影响吞吐量）。

# 持久化

## RDB 持久化

RDB（内存快照，Redis DataBase）持久化是把当前进程数据生成快照保存到磁盘上的过程，由于是某一时刻的快照，所有快照中的值要早于或等于当前内存中的值。

触发方式：

- 手动
   - `save`命令：阻塞 Redis 主进程（期间无法处理任何请求）直至持久化完成
   - `bgsave`命令：阻塞 Redis 主进程直至 fork 出子进程，在子进程中执行持久化
- 自动
   - 配置`save m v`，即在`m`秒内修改次数达到`n`次时，自动触发`bgsave`生成 rdb 文件
   - 主动复制。从节点要从主节点进行全量复制时触发`bgsave`操作，生成当时的快照发送到从节点
   - 执行`debug reload`命令重新加载 Redis 时触发`bgsave`
   - 默认情况下执行`shutdown`命令时，如果没有开启 AOF 持久化，则会触发`bgsave`

> fork 生成子进程的同时，也完成了主进程的内存拷贝，拷贝策略为 Copy-on-Write，仅在主进程或子进程要对内存进行修改时，才会将共享内存以页为单位进行拷贝。

优点：

- 默认使用 LZF 算法进行压缩，压缩后的文件体积远远小于内存大小，**适用于备份、全量复制等场景**
- Redis 加载 rdb 文件恢复数据的速度要远远快于 AOF 方式

缺点：

- 每次调用`bgsave`都需要 fork 子进程，fork 子进程属于重量级操作，会阻塞主进程，频繁执行成本太高
- 由于 fork 成本高，磁盘压力大，所以无法频繁执行，实时性不够，无法做到秒级的持久化
- Redis 服务以外宕机前最后一次成功的持久化之后的数据会丢失。由于 RDB 持久化触发的频率较低，因此丢失的数据较多
- rdb 文件时二进制的，没有可读性，无法人工修改
- rdb 文件存在版本兼容性问题

## AOF 持久化

对于 Redis 收到的每一个写命令，先写内存，再以文本形式写入 aof 日志文件。

处理流程：

- 命令追加（append）：服务器在执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器的 aof_buf 用户缓冲区。
- <span id="write">文件写入（write）</span>：将 aof_buf 中的数据拷贝到 Page Cache。
- <span id="sync">同步（sync）</span>：按照配置的刷盘策略将 Page Cache 中的数据写入 aof 文件。

配置：

```conf
# 启用 AOF
appendonly yes
# AOF 文件名
appendfilename "appendonly.aof"
# 刷盘策略（调用 fsync 的时机），默认 everysec
appendfsync <always|everysec|no>
```

|策略|刷盘时机|优点|缺点|
|-|-|-|-|
|always|写入后立即刷盘|可靠性高，数据基本不丢失|频繁写磁盘对性能影响较大|
|everysec|aof_fsync 线程每秒刷盘|性能适中|宕机时最多丢失 1 秒数据|
|no|操作系统控制刷盘|性能高|宕机时丢失数据较多|

可以通过 AOF 重写来缩小 aof 文件的体积。处理流程：

1. 主进程 fork 出 bgrewriteaof 子进程（阻塞主进程）
2. 子进程按照现有 aof 文件创建一个内容经过精简处理的 aof 文件
3. 主进程将 AOF 重写缓冲区内的命令追加入新的 aof 文件（阻塞主进程）
4. 用新的 aof 文件替代现有 aof 文件

优点：

- 实时性、可靠性较高
- 文本文件可读，可人工修改

缺点：

- aof 文件大
- 重写开销大（fork 子进程、追加重写缓冲区数据时阻塞主进程）
- Redis 加载 aof 文件恢复数据速度远慢于 RDB 方式

## RDB、AOF 混合持久化

Redis 4.0 版本开始支持，RDB 持久化与 AOF 持久化同时运行，每次全量快照完成时清空 aof 文件。

# 高可用：主从复制

## 参考文章

[Redis总结6——复制](https://blog.csdn.net/qq_48238787/article/details/120934765)

## 定义

将主节点上的数据复制到从节点。

## 作用

- 数据冗余：主从复制实现类数据的热备份，是持久化之外的一种数据冗余方式。
- 故障恢复：当主节点出现问题时，可以由从节点提供服务，实现快速的故障恢复；实际上是一种服务的冗余。
- 负载均衡：读写分离，分担服务器负载，提高吞吐量。
- 高可用基石：主从复制是**哨兵**和**集群**能够实施的基础。

## 命令和配置

数据同步命令：

```redis
psync <runId> <offset>
```

- `runId`：主节点 ID，`?`表示主节点未知。
- `offset`：复制进度，起点偏移量。`-1`表示第一次复制。

```redis
# 全量复制
psync ? -1
# 增量复制
psync <runId> <slave_repl_offset>
```

Redis 2.8 版本开始支持增量复制。如果主从同步期间出现网络闪断，那么连接恢复后，主节点会补发丢失的数据给从节点。

每个从节点会记录自己的复制进度`slave_repl_offset`。在和主节点重连进行恢复时，从节点会通过`psync`命令把自己记录的`slave_repl_offset`发送给主节点，主节点会根据双方的复制进度，来决定这个从库执行增量复制还是全量复制。

配置：`slave-serve-stale-data`

- yes：（AP）主从复制过程中，从节点可以响应客户端请求。
- no：（CP）主从复制过程中，从节点对于除了`INFO`和`SLAVEOF`之外的请求，返回`SYNC with master in progress`。

# 高可用：哨兵机制

## 参考文章

[【Redis学习笔记（十二）】之 Sentinel详细介绍](https://blog.csdn.net/Mrwxxxx/article/details/114293968)

## 功能概述

- 配置提供者 Configuration provider：客户端在初始化时，通过连接哨兵来获得当前 Redis 服务的主节点地址。
- 监控 Monitoring：哨兵会不断地检查主节点和从节点是否运作正常。
- 自动故障转移 Automatic failover：当主节点不能正常工作时，哨兵会开始自动故障转移操作。它会将失效主节点的一个从节点升级为新的主节点，并让其他从节点改为复制新的主节点。
- 通知 Notification：哨兵可以将故障转移的结果发送给客户端。

## 哨兵集群的创建

所有哨兵节点通过所有被监控的主从节点的`__sentinel__:hello`频道来相互通信，组件集群。

## 监控 Redis

哨兵节点通过向主节点发送`INFO`命令来获取从节点信息，通过向所有主从节点发送`PING`命令来判断节点的**主观下线**状态。

当某个哨兵节点主观判定主节点下线时，会给其他哨兵发送`SENTINEL is-master-down-by-addr`命令，获取其他哨兵对于主节点是否下线的判断，若得到下线判定的数量达到了哨兵配置里的`quorum`的值，则判定主节点**客观下线**。

## 故障转移

当主节点客观下线后，哨兵集群通过[Raft](TODO)选举选出 Leader 哨兵节点，来执行 Redis 集群的主从切换。

新的主节点的选择：

1. 排除被标记为下线的从节点
2. 选择优先级（redis.conf 中配置的`replica-priority`）最高的
3. 选择复制偏移量（offset）最大的
3. 选择最早启动的（runId 最小的）

切换步骤：

1. 向被选出作为新的主节点的从节点发送命令`replicaof no one`
2. 向其他从节点发送命令`replicaof <新主节点host> <新主节点port>`
3. 通知应用程序新主节点的地址

# 高可扩展：Redis Cluster

## 参考文章

[Redis Cluster 集群详解](https://blog.csdn.net/qq_41432730/article/details/121591008)

## 说明

特点：

- 有 16384（2^14）个 slot，hash 算法：`crc16(key) & 16383`。
   > 槽数为 16384（2KB）而非 65536（8KB）的原因：为了减小心跳包的体积（心跳包中包含发送者负责的 slots 信息）。
- key 中用正则表达式`\{.+?\}`匹配到的第一个字符串为 hash tag，hash tag 相同的 key 会被 hash 到同一个 slot 中。
- 无中心架构。
- 集群由多个节点组构成，每个节点组包含一个 master 节点和任意数量的 slave 节点，存放特定 slot 范围的数据，通过异步主从复制保证数据最终一致性。不同节点组存放的 slot 没有交集。
- 集群中任意两个节点之间都保持着 TCP 长连接，端口为普通命令端口号 +10000。节点在该网状拓扑上以 gossip 协议进行信息交换。
- 集群中 master 节点数必须为奇数，至少需要 6 个节点（3 主 3 从模式）来保证高可用。
   > 原因：master 节点 fail 之后，slave 节点 failover 时要获得超过半数 master 节点（含下线的 master 数）的 ack 才能成为新的 master 节点，master 节点数少于 2 个时必定无法进行。
- 与[基于 replication 搭建的主从架构](#高可用主从复制)不同，Redis Cluster 的 slave 节点默认不支持读写（需通过`readonly`命令启用读服务），仅作为数据热备使用，以及在故障恢复时配合主从切换实现高可用。

优点：

- 可扩展性强：可线性扩展到 1000 个主节点（官方推荐不超过 1000 个），节点可动态添加或删除。
- 高可用性：部分节点不可用时，集群仍可用。通过增加 slave 做 standby 数据副本，能够实现故障自动 failover，节点之间通过 gossip 协议交换状态信息，用投票机制完成 slave 到 master 的角色提升。

缺点：

- 所有节点都使用 db0，不同业务间隔离性较差，容易相互影响

# 使用场景

## 缓存

如热点数据缓存、session 缓存。

## 限时业务

如验证码、限时优惠活动。

实现：`EXPIRE`命令。

## 计数器相关

如秒杀活动、序列号生成。

实现：`INCR`/`DECR`命令。

## 限流

### 生成 key

按业务限流粒度生成 key，比如限制 IP 访问频率时 key 根据 IP 生成，限制 IP 对某接口访问频率时根据 IP 和接口标识生成。

### 滑动窗口计数

通过 ZSet 实现，服务器收到请求时执行以下 lua 脚本，返回值大于 0 时可处理：

```lua
-- KEYS[1]=key
-- ARGV[1]=当前时间-统计时间（如限制1分钟内访问次数，则ARGV[1]=当前时间-1min）
-- ARGV[2]=当前时间
-- ARGV[3]=访问次数上限
-- 返回值=窗口内访问次数（0代表当前请求不可处理）

-- 移除过时的数据
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1])
local num = redis.call('ZCARD', KEYS[1])
if num >= tonumber(ARGV[3]) then
    -- 已达到上限
    return 0
end
-- 未达到上限，记录
return num + redis.call('ZADD', KEYS[1], ARGV[2], ARGV[2])
```

> 该实现不适用于访问次数上限极高的情况，如 1 天内不超过 100 万次访问。

### 令牌桶

应用中通过定时任务以恒定速率补充令牌：

```lua
-- KEYS[1]=key
-- ARGV[1]=补充数量
-- ARGV[2]=令牌桶容量
-- 返回值=补充后令牌总数

local n = tonumber(redis.call('get', KEYS[1])) or 0
if n <= 0 then
    -- 令牌耗尽
    n = ARGV[1]
else
    n = n + ARGV[1]
end
if n - ARGV[2] > 0 then
    -- 补充过多
    n = ARGV[2]
end
redis.call('set', KEYS[1], n)
return n
```

服务器收到请求时执行`decr <key>`，返回值 ≥0 时可处理。

## 排行榜

适用于实时性要求较高且变更频率较高的热点数据排行榜。

实现：ZSet。

- 操作对分数造成影响：\
`zincrby key <分数增量> <影响成员>`
- 查询榜单前N名及对应分数：\
`zrevrange key 0 <N-1> withscores`

## 分布式锁

### 参考文章

[redis分布式锁的8大坑【Redis分布式锁】](https://blog.csdn.net/wang121213145/article/details/125051770)

### 说明

1.	按业务加锁粒度生成 key
2.	加锁：\
`SET key value [EX seconds|PX milliseconds] NX`\
<font color="green">:o:</font> OK\
<font color="red">:x:</font> (nil)
3.	启动deamon线程，周期性续持锁\
`EXPIRE key seconds`\
<font color="green">:o:</font> 1\
<font color="red">:x:</font> 0
4.	执行业务处理
5.	解锁\
`DEL key`\
<font color="green">:o:</font> 1\
<font color="red">:x:</font> 0

## 用户上线次数统计

通过 String 的 Bitmap 实现。

- 第 N 日登陆标记：\
`setbit <key> N 1`
- 查询总登陆次数：\
`bitcount <key>`
- 查询第 N 日是否登陆：\
`getbit <key> N`

> String存储上限为 512MB，即 2^32^ 位。

## UV（Unique Visitor，独立访客量）统计

实现：HyperLogLog。

- 增加计数：\
`pfadd [页面key] [用户1 key] [用户2 key] … [用户n key]`
- 获取计数：\
`pfcount [页面key]`
- 合并计数：\
`pfmerge [合并页面key] [页面1 key] [页面2 key] … [页面n key]`\
`pfcount [合并页面key]`

## 分时段活跃用户数统计

参考 [UV 统计](#uvunique-visitor独立访客量统计)。

# 缓存淘汰

## 参考文章

[redis淘汰策略](https://blog.csdn.net/weixin_40980639/article/details/125446002)

[Redis的LFU算法源码实现解析](https://blog.csdn.net/qq_41688840/article/details/122423518)

## 配置

```ini
# [必须]最大缓存
maxmemory 4gb
# 缓存淘汰策略（默认 noeviction）
maxmemory-policy noeviction
```

## 淘汰策略

![Redis maxmemory-policy](img/Redis/Redis-maxmemory-policy.png)

- noeviction：默认策略。缓存写满后，对于写请求不再提供服务，直接返回错误。
   > 这种策略不会淘汰数据，所以无法解决缓存污染问题。一般生产环境不建议使用。\
   其他七种规则都会根据自己相应的规则来选择数据进行删除操作。
- volatile-ttl：对于设置了过期时间的键值对，越早过期的越先被删除。
- (volatile|allkeys)-random：从待淘汰数据集合中随机删除。
- (volatile|allkeys)-lru：对待淘汰数据集合使用LRU算法进行淘汰，淘汰最久未访问的数据。
   > Redis在每个数据对应的 RedisObject 结构体中设置一个 lru 字段（24bits），用来记录数据的访问时间戳。在进行淘汰时，从待淘汰数据集中随机采样`maxmemory-samples`（默认值5）个元素，淘汰时间戳最小的元素。\
   LRU 算法的弊端：假如一个 key 访问频率很低，但是最近被访问过，那么它会被 LRU 判定为热点数据，不会被淘汰；假如一个 key 访问频率很高，但是最近一段时间没有被访问，那么它会被 LRU 误判并淘汰。于是在 Redis 4.0 中加入了 LFU 策略。
- (volatile|allkeys)-lfu：对待淘汰数据集合使用LFU算法进行淘汰，淘汰访问频率最低的数据。
   > lru 字段的高 16 位用于记录最近访问时间戳，低 8 位用于记录访问次数。每次访问数据时：
   > 1. 根据距离上次访问的时长，衰减访问次数\
   衰减量：`(当前时间 - lru 时间戳)(min) / lfu-decay-time`\
   `lfu-decay-time` 默认值 10
   > 2. 根据当前访问更新访问次数\
   增加量：`1 / (lru 计数器 * lfu-log-factor + 1) > rand(0, 1) ? 1 : 0`\
   `lfu-log-factor`默认值 1
   > 3. 更新 lru 变量值（时间戳、计数器）

<style type="text/css">* {font-family: YaHei Consolas Hybrid, Consolas;}</style>