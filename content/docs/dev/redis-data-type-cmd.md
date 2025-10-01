---
title: "Redis 的常用数据类型与命令"
weight: 1
date: 2025-09-28
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

# Redis 的常用数据类型与命令

Redis 是一种 KV 数据库.
由于其数据全部存放于内存中, 因此速度相当快.
常常被用于与 SQL 数据库配合作为缓存使用.

操作 Redis 时, 主要是使用命令来直接对 key 做读写, 不需要 SQL 语法.

Redis 的类型系统和传统数据库不一样, 它是一个 Key-Value 数据库, 但 Value 部分不是只能是字符串, 而是支持多种内置数据结构类型.

Redis 把这些称为 数据类型. 每个 key 都只能对应一个类型的值, 命令也都是和类型强相关的.

## 通用命令

在 Redis 里除了数据类型相关的专用命令 (比如 `HSET` 只对 hash 用, `LPUSH` 只对 list 用) , 还存在一类 通用命令, 这些命令几乎对所有类型的键都适用.

- `DEL key`: 删除 key.
- `EXISTS key`: 判断 key 是否存在.
- `TYPE key`: 查看 value 对应的数据类型.
- `RENAME key newkey`: 重命名 key
  - `RENAMENX`: 用法相同, 如果存在就重命名
- `COPY source destnation`: 从一个键复制到另一个键
- `EXPIRE key seconds`: 给 key 指定一个过期时间, 单位是秒.
  - `PEXPIRE key milliseconds`: 同上, 时间单位为毫秒.
- `EXPIREAT key timestamp`: 给 key 指定过期时间戳, 单位为秒
  - `PEXPIREAT key timestamp`: 同上, 单位为毫秒.
- `TTL key`: 查询过期时间, 单位为秒
  - `PTTL key`: 查询过期时间, 单位为毫秒
- `PERSIST key`: 移除过期时间, 使其永久存在.

## String

最基本的类型, 二进制安全, 可以存储文本, JSON, 图片/序列化后的对象. 最大 512 MB.

常见的操作命令如下:
- `SET key value`: 设置 key 对应的值为 value.
- `GET key`: 获取 key 对应的值.
- `INCR key`: 将 key 对应的值当成整数进行自增. 如果对应的值不是整数会报错. 比如:
    - `"10"` 变成 `"11"`
    - `"0010"` 变成 `"11"` (前导 0 会消失)
    - `"abc"` 报错 `(error) ERR value is not an integer or out of range`
- `DECR key`: 自减操作, 类似上面的 `INCR`.
- `INCRBY key increment`: 给某个 field 的整数值自增.
  - **值得注意的是没有 `DECRBY` 命令**, 可以用负数来实现.
- `INCRBYFLOAT key increment`: 给某个 field 的值自增浮点数.
  - **同样没有 `DECRBYFLOAT` 命令**, 可以用负数来实现.
- `MSET key1 value1 key2 value2 ...`: 设置多个 KV 对.
- `MGET key1 key2 ...`: 获取多个 key 对应的值.
- `SETNX key value`: 如果 key 不存在就设置 key 对应的值为 value.
- `SETEX key seconds value`: 命令设置指定的键值, 并且为该键指定一个过期时间, 单位是秒.
  - 当键的过期时间到达时, 键会自动被删除.
- `APPEND key value`: 追加 value 到 key 的值后面.
- `STRLEN key`: 获取 key 对应值的长度.

## List

Redis 的 List 是一种有序的数据结构, 底层是一个 双端链表 (早期实现是 linked list, 现在是压缩列表+ quicklist 优化). 它支持在两端高效地插入和删除, 非常适合用来实现消息队列、任务队列、栈、队列等场景

- `LPUSH key value1 value2 ...`: 从左边插入一个或多个元素.
  - 在 Redis 里面没有专门的创建 list 命令. 只要对一个 key 使用了 list 类型相关的命令 (比如 `LPUSH`, `RPUSH`) 就会自动创建一个新的 list 并存储
- `RPUSH key value1 value2 ...`: 从右边插入.
- `LPOP key`: 从左边弹出并返回元素.
- `RPOP key`: 从右边弹出.
- `LRANGE key start stop`: 获取列表中 `[start, stop]` 的元素
  - 类似 Python, 支持负数索引 (-1 表示最后一个)
- `LINDEX key index`: 根据索引获取元素.
- `LLEN key`: 获取列表长度.
- `LREM key count value`: 删除列表中指定数量的 value.
  - 当 count > 0 时从左边删除
  - 当 count < 0 时从右边删除
  - **因此没有 `RREM` 命令**
- `LSET key index value`: 设置指定索引的元素值.
  - 由于 index 可以是负数, 因此 **没有 `RSET` 命令**
- `LTRIM key start stop`: 截取列表, 保留 `[start, stop]` 的元素.
- `RPOPLPUSH source destination`: 从 source 的尾部弹出一个元素并插入到 destination 的头部
  - 常用于任务队列「取出 + 分发」
- `BLPOP key1 key2 ... timeout`: 阻塞式地从左边弹出 (如果列表为空则等待)
  - 如果指定多个 key 则按顺序依次检查
  - 哪个 key 最先有元素可弹出, 就返回哪个
  - 如果多个 key 同时有值, 优先取列表顺序最靠前的 key
- `BRPOP key1 key2 ... timeout`: 阻塞式地从右边弹出 (如果列表为空则等待)

> [!TIP]
> 一些 List 的使用场景:
> - 消息队列/任务队列: 使用 `LPUSH` + `BRPOP` 或 `RPUSH` + `BLPOP` 模拟生产者-消费者模型。
> - 栈: 用 `LPUSH` + `LPOP` 实现。
> - 队列: 用 `LPUSH` + `RPOP` 或 `RPUSH` + `LPOP` 实现。
> - 限长日志列表: 用 `LPUSH` + `LTRIM` 保持最新 N 条。

## Hash

Redis 的 Hash 类型 (哈希表) 是用来存储 键值对集合 的一种数据结构. 你可以把它理解为在 Redis 里存了一个小型的字典 (map / object),适合表示对象结构化的数据. 

- `HSET key field value [field value ...]`: 设置单个或多个 field.
  - `HMSET` 等价于 `HSET` 设置多个 field. Redis 4.0 之后推荐用 `HSET` 替代
- `HINCRBY key field increment`: 给某个 field 的整数值自增.
  - **这里没有 `HINCR` 命令**.
  - **这里也没有 `HDECRBY` 命令**, 可以用负数来实现.
- `HINCRBYFLOAT key field increment`: 对浮点数进行增减.
  - **这里也没有 `HDECRBYFLOAT` 命令**, 可以用负数来实现.
- `HGET key field`: 获取单个 field 的值.
  - `HGET` **只能**获取单个 field 的值.
- `HMGET key field1 field2 ...`: 获取多个 field 的值.
- `HGETALL key`: 获取所有 field 和 value.
- `HKEYS key`: 获取所有 field.
- `HVALS key`: 获取所有 value.
- `HLEN key`: 获取 field 的个数.
- `HDEL key field [field ...]`: 删除指定的 field.

## Sorted Set (ZSet)

Redis 的 Sorted Set (有序集合) 是一种类似集合 (Set) 的数据类型, **不允许重复元素**, 但每个元素会关联一个 score (浮点数), Redis 会根据 score 进行排序.

- `ZADD key score member [score member ...]`: 向集合添加一个或多个元素, 并设置分数. 如果元素已存在, 则更新分数.
- `ZREM key member [member ...]`: 移除一个或多个成员.
- `ZINCRBY key increment member`: 给指定成员的分数加上增量 (可为负数).
- `ZRANGE key start stop [WITHSCORES]`: 按分数升序获取区间内元素.
  - 默认只返回 member.
  - 如果指定 `WITHSCORES` 则返回 score, 与 member 交替排列
- `ZREVRANGE key start stop [WITHSCORES]`: 按分数降序获取区间内元素.
- `ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]`: 获取分数在指定区间 `[min, max]` 内的元素
  - 如果指定 `LIMIT` 则对结果集做分页.
  - 从第 `offset` 个结果开始，最多取 `count` 个
- `ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]`: 反向获取分数范围内的元素
- `ZRANK key member`: 获取 member 的排名. (升序, 从 0 开始)
- `ZREVRANK key member`: 获取 member 的排名. (降序)
- `ZSCORE key member`: 获取 member 的分数.
- `ZCARD key`: 获取元素的数量.
- `ZCOUNT key min max`: 统计分数在 `[min, max]` 区间的元素数量.
- `ZPOPMIN key [count]`: 弹出分数最小的 count 个元素.
- `ZPOPMAX key [count]`: 弹出分数最大的 count 个元素.
- `ZREMRANGEBYRANK key start stop`: 删除指定排名范围的元素.
- `ZREMRANGEBYSCORE key min max`: 删除分数范围内的元素.
- `ZUNIONSTORE dest numkeys key [key ...] [WEIGHTS w1 w2 ...] [AGGREGATE SUM|MIN|MAX]`: 多个有序集合做并集, 结果存入新集合
  - `WEIGHT` 为权重, 会为每个集合的元素的 score 乘以该集合的权重
  - `AGGREGATE` 为 score 聚合方式:
    - `SUM`: 为求和
    - `MIN`: 取最小值
    - `MAX`: 取最大值
- `ZINTERSTORE dest numkeys key [key ...] [WEIGHTS w1 w2 ...] [AGGREGATE SUM|MIN|MAX]`: 多个有序集合做交集

> [!INFO]
> Redis 6.2+ 增加了 `ZUNION` / `ZINTER` 直接返回结果

> [!TIP]
> 一些 ZSet 的使用场景:
> - 排行榜: 按分数记录分数, 使用 `ZREVRANGE` 获取前 N 名
> - 优先队列: 分数作为优先级, 使用 `ZPOPMIN` 弹出优先级最高的任务
> - 时间线: 分数设为时间戳, 用 `ZRANGEBYSCORE` 查询某时间段的内容


## Stream

Redis Stream 是 Redis 在 5.0 引入的一种数据类型, 主要用于消息队列 (Message Queue)、事件日志 (Event Log) 和 实时数据流 (Real-time Stream) 场景. 它类似于 Kafka、RabbitMQ 的一些功能, 但更轻量, 直接内置在 Redis 里.

Stream 中每条消息由唯一 ID 进行标识. ID 格式为时间戳-序号. 消息按 ID 排序, 保证消息有序.
每条消息的生命周期如下:

1. 产生：通过 `XADD` 写入
2. 存活：保存在 Stream，直到被删除或裁剪
3. 消费：
   - 普通读取：不会影响消息
   - 消费者组：进入等待队列 (PEL)，直到 `XACK`
4. 确认/转移：`XACK` 确认 or `XCLAIM` 转移
5. 消亡：通过 `XDEL` / `XTRIM` / `DEL` 被清理

常用的命令有:
- `XADD key ID field value [field value ...]`: 向 stream 添加消息
  - ID 通常用 * 来自动生成.
- `XRANGE key start end [COUNT n]`: 按 ID 范围读取消息 (正序)
  - start 与 end 可以用 `-` `+` 来指代最小 ID 与最大 ID.
  - 例如 `XRANGE mystream - + COUNT 10` 就是从 `mystream` 里 正序 (从小到大 ID) 返回 前 10 条消息
- `XREVRANGE key end start [COUNT n]`: 按 ID 范围读取消息 (倒序)
- `XREAD [COUNT n] [BLOCK ms] STREAMS key id [key id ...]`: 读多个 stream, 从指定 ID 开始读取 ($ 表示最新消息)
- `XGROUP CREATE key groupname id [MKSTREAM]`: 创建消费者组 (id 一般用 $ 表示从最新开始)
  - 加上 `MKSTREAM` 参数时, 如果 stream 不存在, Redis 会自动创建一个 空的 stream, 然后再建立消费者组
- `XREADGROUP GROUP group consumer [COUNT n] [BLOCK ms] STREAMS key id ...`: 从组里读取消息 (分配给具体 consumer)
- `XACK key group id [id ...]`: 确认消息已消费
- `XCLAIM key group consumer min-idle-time id [id ...]`: 抢占未确认的消息（常用于消费者崩溃后转移）
  - `min-idle-time` 为最小空闲时间, 防止刚分配的消息被抢占
- `XPENDING key group [start end count [consumer]]`: 查看未确认消息的状态
- `XLEN key`: 获取 stream 长度
- `XDEL key id [id ...]`: 删除指定消息
- `XTRIM key MAXLEN [~] count`: 裁剪消息流，限制最大长度
  - 当指定 `~` 时为近似裁剪, 性能更好但不会保证精确数量
- `XINFO STREAM key`: 查看 stream 信息 (消息数、消费者组等)


除此以外, redis 还有其他的数据类型例如: Bitmap, Geo, HyperLogLog. 但这些数据类型用的比较少:
- Bitmap
  - 典型场景比较窄（签到、活跃用户统计）.
  - 优点是极致的内存利用率.
- Geo
  - 用于地理信息检索（外卖、打车、LBS 服务）.
  - 但大多数公司直接用 ES / 专门的地理数据库替代.
- HyperLogLog
  - 适合做去重统计，但精度有限.
  - 多数业务还是用大数据工具（如 Hive/Spark）统计，Redis 里用得不算多.