# Redis实战（三）：Redis的List、Set、Hash、sorted_set、skip list

寒泉Hq 2020-06-24 18:57:57  8962  已收藏 3  原力计划
分类专栏： # Redis实战



## String类型（上节回顾）

<img src=".\images\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NpbmF0XzQyNDgzMzQx,size_1,color_FFFFFF,t_70" alt="在这里插入图片描述" style="zoom:60%;" />

## List 类型
``help @list``查看帮助

![在这里插入图片描述](.\images\20210506001)

![在这里插入图片描述](.\images\20210506002)

### 可以用List类型实现一个栈：
lpush k1 a b c d e左边push
lpop k1 a b c d e左边pop（后进先出）

### 可以用List类型实现一个队列：
lpush k1 a b c d e左边push
rpop k1 a b c d e右边pop（先进先出）

### 获取List中某个范围之间的所有元素（支持负向索引）
LRANGE k1 0 -1：获取整个List

### 所有关于List的命令：

~~~properties
BLPOP key [key ...] timeout (阻塞，单播队列)删除并获取列表中的第一个元素，或块中的第一个元素，直到有一个可用
BRPOP key [key ...] timeout 删除并获取列表中的最后一个元素，或块，直到有一个可用
BRPOPLPUSH source destination timeout 从列表中弹出一个值，推送到另一个列表并返回;或阻塞，直到可用为止
LINDEX key index 通过索引从列表中获取元素
LINSERT key BEFORE|AFTER pivot value 在列表的另一个元素之前或之后插入一个元素
LLEN key 获取列表的长度
LPOP key 删除并获取列表中的第一个元素
LPUSH key value [value ...] 在列表前添加一个或多个值
LPUSHX key value 仅当列表存在时，在列表前添加一个值
LRANGE key start stop 从列表中获取元素的范围
LREM key count value 从列表中删除元素
LSET key index value 通过索引设置列表中元素的值
LTRIM key start stop 将列表修剪到指定范围
RPOP key 删除并获取列表中的最后一个元素
RPOPLPUSH source destination 删除列表中的最后一个元素，将其添加到另一个列表并返回
RPUSH key value [value ...] 向列表追加一个或多个值
RPUSHX key value 仅当列表存在时，向列表追加一个值

~~~

## HashMap 类型

![在这里插入图片描述](.\images\2020062417085796.png)

<img src="D:\学习\面试资料\java-interview\redis\images\20210506003" alt="在这里插入图片描述" style="zoom:67%;" />

``help @hash``

~~~properties
HDEL key field [field ...] 删除一个或多个散列字段
HEXISTS key field 确定散列字段是否存在
HGET key field 获取散列字段的值
HGETALL key 获取散列中的所有字段和值
HINCRBY key field increment 将哈希字段的整数值按给定的数字递增
HINCRBYFLOAT key field increment 将哈希字段的浮点值按给定的量递增
HKEYS key 获取散列中的所有字段
HLEN key 获取散列中的字段数
HMGET key field [field ...] 获取所有给定哈希字段的值
HMSET key field value [field value ...] 将多个哈希字段设置为多个值
HSCAN key cursor [MATCH pattern] [COUNT count]  递增迭代哈希字段和关联值
HSET key field value 设置散列字段的字符串值
HSETNX key field value 设置散列字段的值，仅当该字段不存在时
HSTRLEN key field 获取散列字段值的长度
HVALS key 获取散列中的所有值

~~~

业务场景：微博点赞，数量增加；收藏、详情页



## Set 类型

List 是有序的（插入顺序）
Set 是乱序的，去重的

<img src=".\images\20210506004" alt="在这里插入图片描述" style="zoom:65%;" />

~~~properties
SADD key member [member ...] 向集合中添加一个或多个成员
SPOP key [count] 从集合中移除并返回一个或多个随机成员
SREM key member [member ...] 从集合中删除一个或多个成员
SDIFF key [key ...] 方向性地求差集
SUNION key [key ...] 多个set求并集
SINTER key [key ...] 多个set取交集
SDIFFSTORE destination key [key ...] 减去多个集合并将结果集存储在一个键中
SINTERSTORE destination key [key ...] 交叉多个集合并将结果集存储在一个键中
SISMEMBER key member 确定给定值是否是集合的成员
SMEMBERS key 获取集合中的所有成员
SCARD key 获取集合中的成员数
SMOVE source destination member 将成员从一个集合移动到另一个集合
SRANDMEMBER key [count] 从集合中随机获取一个或多个成员
						count是正数：取出一个不重复的结果集（不能超过已有集）
						count是负数：取出一个有可能重复的结果集（一定满足你要求的数量）
						人多于奖品/奖品多于人/可以重复/不能重复 不同的场景
SSCAN key cursor [MATCH pattern] [COUNT count]
SUNIONSTORE destination key [key ...] 添加多个集合并将结果集存储在一个键中

~~~





## SortedSet 类型

自带元素**排序**；自带去重

<img src=".\images\20210506005" alt="在这里插入图片描述" style="zoom:80%;" />

你想怎么排序？

- 名称
- 含糖量（前端不展示）
- 大小（前端不展示）
- 价格（前端不展示）
- 粉丝数（前端不展示）

因此，除了**元素本身**以外，你需要有**分值**这个维度，用来排序。
如果分值相同，则按照名称字典序排列。

正序？逆序？
每个元素都有自己的正负向索引

<img src=".\images\20210506006" alt="在这里插入图片描述" style="zoom:80%;" />

<img src=".\images\20210506007" alt="在这里插入图片描述" style="zoom:70%;" />

``help @sorted_set``

~~~properties
BZPOPMAX key [key ...] timeout 删除并返回得分最高的成员从一个或多个sorted set,或阻塞,直到一个是可用的
BZPOPMIN key [key ...] timeout 删除并返回分数最低的成员从一个或多个sorted set,或阻塞,直到一个是可用的
ZADD key [NX|XX] [CH] [INCR] score member [score member ...] 向sorted set中添加一个或多个成员，如果已经存在，则更新其分数
ZCARD key 获取sorted set中的成员数
ZCOUNT key min max 在给定值内对sorted set中的成员进行计数
ZINCRBY key increment member 递增sorted set中成员的分数
ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX] 交叉多个sorted set，并将结果sorted set存储在一个新的键中
ZLEXCOUNT key min max 计算给定字典法范围之间sorted set中的成员数
ZPOPMAX key [count] 删除并返回sorted set中得分最高的成员
ZPOPMIN key [count] 删除并返回sorted set中得分最低的成员
ZRANGE key start stop [WITHSCORES] Return a range of members in a sorted set, by index 按索引返回sorted set中成员的范围
ZRANGEBYLEX key min max [LIMIT offset count] 按分数返回sorted set中的成员范围
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count] 按分数返回sorted set中的成员范围
ZRANK key member 确定sorted set中成员的索引
ZREM key member [member ...] 从sorted set中移除一个或多个成员
ZREMRANGEBYLEX key min max 删除给定字典排序范围之间sorted set中的所有成员
ZREMRANGEBYRANK key start stop 删除给定索引内sorted set中的所有成员
ZREMRANGEBYSCORE key min max 删除sorted set中给定分数内的所有成员
ZREVRANGE key start stop [WITHSCORES] 按索引返回sorted set中成员的范围，分数从高到低排序
ZREVRANGEBYLEX key max min [LIMIT offset count] 按字典顺序从较高的字符串到较低的字符串，返回sorted set中的成员的范围。
ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count] 按分数返回sorted set中的成员范围，分数从高到低排序
ZREVRANK key member 确定sorted set中成员的索引，分数从高到低排序
ZSCAN key cursor [MATCH pattern] [COUNT count] 递增迭代sorted set元素和相关分数
ZSCORE key member 获取sorted set中与给定成员相关的分数
ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight] [AGGREGATE SUM|MIN|MAX] 多个sorted set求并集，分数相同时，将分数取最大/最小/平均，并将结果sorted set存储到一个新键中

~~~

flushall

zadd yuwen 80 tom 60 sean 70 baby

zadd shuxue 60 tom 80 sean 40 yiming

#zunionstore 目标key  2个key   2个key的值 相同key做相加 不同的保留下

zunionstore unkey 2 yuwen shuxue



#将两个sortedset的 相同的key的值 取最大保留出来，不同key保留value

zunionstore unkey1 2 yuwen shuxue aggregate max









`ZUNIONSTORE` 示例：

<img src=".\images\20210506008" alt="在这里插入图片描述" style="zoom:67%;" />



## skip list 跳跃表

#### 排序是怎么实现的？skip list 跳跃表

<img src=".\images\20210506009" alt="在这里插入图片描述" style="zoom:80%;" />





