# [Redis支持的数据类型](https://www.cnblogs.com/jasontec/p/9699242.html)和[持久化方式]( https://blog.csdn.net/yhl_jxy/article/details/91879874 )

##  1.Redis支持的数据类型？



redis内核kernel的epoll模型

redis单进程，单线程，单实例，并发很多的请求，如何变得很快的呢？

redis的客户端（多个），发送请求，打到Linux的网卡上，redis处理请求是单线程的，redis通过kernel与用户态空间的共享内存空间，里面存在一个红黑树，链表。redis读取client请求的字节流存放，省去序列化成本。

string类型的key中存储了value的type类型：

string类型的key中存储了value的encoding类型：int string raw 

<img src=".\images\1" alt="img" style="zoom:80%;" />





string类型的value的字符，每个字节，设置一个索引位置，同时维护正向/反向 两个索引。

如图：

<img src=".\images\2" alt="img" style="zoom:60%;" />

> 问：Redis单线程是为了减少用户态到内核态的切换吗？
> 答：不是，至少主要原因不是。
> 操作系统为了响应多用户的请求，而进行的从用户态到内核态的切换，造成的的性能损耗，远不及为了保证数据一致性加锁带来的损耗。
> Redis单线程是为了避免加锁的过程。







### 1.1 String的实际应用场景比较广泛的有：

- **缓存功能：String**字符串是最常用的数据类型，不仅仅是**Redis**，各个语言都是最基本类型，因此，利用**Redis**作为缓存，配合其它数据库作为存储层，利用**Redis**支持高并发的特点，可以大大加快系统的读写速度、以及降低后端数据库的压力。
- **计数器：**许多系统都会使用**Redis**作为系统的实时计数器，可以快速实现计数和查询的功能。而且最终的数据结果可以按照特定的时间落地到数据库或者其它存储介质当中进行永久保存。
  -  **`incr命令`**是增加键的整数值一次，如果键没有被设置，则设置为1，返回自增后的数值。这里利用了redis的原子性，即同一个键只允许一个客户端操作，确保自增值准确。 
- **共享用户Session：**用户重新刷新一次界面，可能需要访问一下数据进行重新登录，或者访问页面缓存**Cookie**，但是可以利用**Redis**将用户的**Session**集中管理，在这种模式只需要保证**Redis**的高可用，每次用户**Session**的更新和获取都可以快速完成。大大提高效率。
- bitmap 位图

> 使用string的bitmap位图用作统计用户登陆天数统计，签到等

#### 统计某用户登陆天数 【在某个统计窗口上】

​      一位表示一年种第几天登陆情况 1表示登陆
用户    1  2 3 4 5 6 7 8 9 10 11 12 13 14 15 16  ...... 364
tom    0  1 0 0 1 0 0 0 1  0   0   0   0    0    0  0    ......  1
jac      0  1 1 0 0 0 0 1 0  1   1   0   0    1    0  1    ......  0

setbit tom 1 1
setbit tom 4 1
setbit tom 8 1
setbit tom 364 1
STRLEN tom
bitcount tom -2 -1 [计算tom全年最后两周的登陆天数]

![image-20210428112253230](.\images\image-20210428112253230.png)

结果：一年中 tom最后两周登陆过1次。





#### 活跃用户统计|随机窗口
比如说1号-3号，只要有一天登陆过就算登陆。 3天连续登陆需要去重
tom 占第1号位
jack 占第7号位
如下表示 tom 16/18号登陆了；jack 16/17/18三天登陆了

setbit 20210416 1 1          0100 0000
setbit 20210416 7 1          0000 0010  
setbit 20210417 7 1          0000 0010
setbit 20210418 1 1          0100 0000
setbit 20210418 7 1          0000 0010

#得到目标key 已去重
bitop or destkey 20210416 20210417 20210418
#计算3天内有过登陆的人数
bitcount destkey 0 -1

<img src=".\images\image-20210428113420357.png" alt="image-20210428113420357" style="zoom:80%;" />

结果：连续3天内 两人登陆过。

### 1.2 Hash：

​		这个是类似 **Map** 的一种结构，这个一般就是可以将结构化的数据，比如一个对象（前提是**这个对象没嵌套其他的对象**）给缓存在 **Redis** 里，然后每次读写缓存的时候，可以就操作 **Hash** 里的**某个字段**。

​		但是这个的场景其实还是多少单一了一些，因为现在很多对象都是比较复杂的，比如你的商品对象可能里面就包含了很多属性，其中也有对象。我自己使用的场景用得不是那么多。

[详细使用参考Redis的Hash用法 ](https://blog.csdn.net/qq_41307443/article/details/79766930)

> 1.hset  key value(key value) :向Hash中存入值。
>
> 2.hget key value(key）:取出Hash中key的值。
>
> 3.hmset ：向Hash表中存入该对象的多个属性值。
>
> ​		注意：当向同一个对象的同一个属性赋多个值时，会覆盖。不同属性时，会拼接。
>
> 4.hsetnx key：判断新增的属性值是否存在，存在新增失败返回0，不存在新增成功，返回1.



### 1.3 List：

list的key中存在head和tail两个指针，分别指向链表的头和尾。

list 的value 是一个双向链表的结构，同时维护了正向/反向两个索引。

如图：

<img src=".\images\2" alt="img" style="zoom:60%;" />



**List** 是有序列表，这个还是可以玩儿出很多花样的。

* 比如可以通过 **List** 存储一些列表型的数据结构，类似粉丝列表、文章的评论列表之类的东西。

* 比如可以通过 **lrange** 命令，读取某个闭区间内的元素，可以基于 **List** 实现分页查询，这个是很棒的一个功能，基于 **Redis** 实现简单的高性能分页，可以做类似微博那种下拉不断分页的东西，性能高，就一页一页走。

  * Redis Lrange 返回列表中指定区间内的元素，区间以偏移量 START 和 END 指定。 其中 0 表示列表的第一个元素， 1 表示列表的第二个元素，以此类推。 你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。

    redis Lrange 命令基本语法如下：

    ```shell
    redis 127.0.0.1:6379> LRANGE KEY_NAME START END
    ```

* 比如可以搞个简单的消息队列，从 **List** 头怼进去，从 **List** 屁股那里弄出来。

**List**本身就是我们在开发过程中比较常用的数据结构了，热点数据更不用说了。

- **消息队列：Redis**的链表结构，可以轻松实现阻塞队列，可以使用左进右出的命令组成来完成队列的设计。比如：数据的生产者可以通过**Lpush**命令从左边插入数据，多个数据消费者，可以使用**BRpop**命令阻塞的“抢”列表尾部的数据。
- 文章列表或者数据分页展示的应用。

比如，我们常用的博客网站的文章列表，当用户量越来越多时，而且每一个用户都有自己的文章列表，而且当文章多时，都需要分页展示，这时可以考虑使用**Redis**的列表，列表不但有序同时还支持按照范围内获取元素，可以完美解决分页查询功能。大大提高查询效率。

### 1.4 Set：

**Set** 是无序集合，会自动去重的那种。

直接基于 **Set** 将系统里需要去重的数据扔进去，自动就给去重了，如果你需要对一些数据进行快速的全局去重，你当然也可以基于 **JVM** 内存里的 **HashSet** 进行去重，但是如果你的某个系统部署在多台机器上呢？得基于**Redis**进行全局的 **Set** 去重。

可以基于 **Set** 玩儿交集、并集、差集的操作，比如交集吧，我们可以把两个人的好友列表整一个交集，看看俩人的共同好友是谁？对吧。

反正这些场景比较多，因为对比很快，操作也简单，两个查询一个**Set**搞定。

### 1.5 Sorted Set：

**Sorted set** 是排序的 **Set**，去重但可以排序，写进去的时候给一个分数，自动根据分数排序。

有序集合的使用场景与集合类似，但是set集合不是自动有序的，而**Sorted set**可以利用分数进行成员间的排序，而且是插入时就排序好。所以当你需要一个有序且不重复的集合列表时，就可以选择**Sorted set**数据结构作为选择方案。

- 排行榜：有序集合经典使用场景。例如视频网站需要对用户上传的视频做排行榜，榜单维护可能是多方面：按照时间、按照播放量、按照获得的赞数等。
- 用**Sorted Sets**来做带权重的队列，比如普通消息的score为1，重要消息的score为2，然后工作线程可以选择按score的倒序来获取工作任务。让重要的任务优先执行。

微博热搜榜，就是有个后面的热度值，前面就是名称

```shell
redis 127.0.0.1:6379> ZADD runoobkey 1 redis
(integer) 1
redis 127.0.0.1:6379> ZADD runoobkey 2 mongodb
(integer) 1
redis 127.0.0.1:6379> ZADD runoobkey 3 mysql
(integer) 1
redis 127.0.0.1:6379> ZADD runoobkey 3 mysql
(integer) 0
redis 127.0.0.1:6379> ZADD runoobkey 4 mysql
(integer) 0
redis 127.0.0.1:6379> ZRANGE runoobkey 0 10 WITHSCORES

1) "redis"
2) "1"
3) "mongodb"
4) "2"
5) "mysql"
6) "4"
```

### 高级用法：

### **Bitmap** :

位图是支持按 bit 位来存储信息，可以用来实现 **布隆过滤器（BloomFilter）**；





## 2.Redis持久化及持久化方式优缺点

### 文章目录

- [前言](https://blog.csdn.net/sinat_36759535/article/details/108731617?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-1.control#_7)
- [一、RDB](https://blog.csdn.net/sinat_36759535/article/details/108731617?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-1.control#RDB_10)
- [二、AOF](https://blog.csdn.net/sinat_36759535/article/details/108731617?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-1.control#AOF_37)
- [总结](https://blog.csdn.net/sinat_36759535/article/details/108731617?utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromMachineLearnPai2~default-1.control#_76)

### 前言

redis作为内存数据库，存在断电数据丢失的问题，所以redis有两种技术实现来保证数据的完整性。rdb和aof。分别代表内存数据库两种思路，全量快照保存和日志形式保存。

### 一、RDB

学习rdb最权威的方式就是去看他的redis.conf配置文件，里面有很多详细说明

![img](.\images\20210514001)

rdb是全量保存当前时刻内存数据到磁盘。
从文档的描述可以看出，rdb保存的周期是根据这个公式来的
save
在多少时间内，key变化了多少次
就比如上面的三个配置，可以得出，
redis在60秒内变化了10000次则触发rdb写进磁盘
或者redis在300秒内key变化了10次
再或者redis在900秒内key变化了1次写磁盘

这里其实有个问题存在的

假设晚上12点redis需要将rdb文件写到磁盘，那么我们知道写磁盘涉及到IO，那么redis是不是会阻塞？
答案是，redis肯定不会阻塞，因为一旦阻塞，就会影响redis的使用。那么redis是怎么做的呢？
答案是，redis在这里通过linux的系统调用接口fork出一个子进程。fork为什么可以实现？
理由是fork的话，那么在内存中会消耗极少的空间来存储redis中对应key的引用指针，然后fork底层利用copy-on-write写时复制的方式实现高效。写时复制相当于java的copyonwriteList的原理，当你需要在redis主进程操作key的时候，在内存中会复制一份原来value值，然后把redis主进程的key的引用指针改写到这个新的内存空间，这样就不影响到fork出来的进程。写时复制是在大量工程中验证过得，因为很少发生大量对原来进程所有的引用重写更改的情况。所以fork的效率非常高。



![img](.\images\20210514002)

但是如果单纯使用rdb，可以想象得到，在我进行rdb文件写磁盘的时候，同时用户对redis有key的操作的话，那这些数据就会丢失。要解决这个问题，就需要aof了。

### 二、AOF

![img](.\images\20210514003)



通过aof的描述，可以知道aof相比rdb对于数据完整性更好
aof是实时记录用户对key的新增，修改，删除操作，然后将这些操作日志写在和rdb相同目录下的appendonly.aof文件中。
aof有三种方式写入磁盘aof文件中，分别是

~~~shell
#这是最安全的方式，用户每次对key的更改操作，redis都会调用linux内核直接flush到磁盘中，这种模式丢失数据最少，丢失一条数据
# appendfsync always

#redis会把数据先发送都linux内核缓存起来，时间每间隔一秒调用系统的flush写到磁盘，这种模式丢失数据中等，一秒的数据，小于一个缓冲区的大小
appendfsync everysec

#redis不管linux内核什么时候将数据flush 到磁盘，只要linux内核的数据缓存满了，linux内核会flush，这种模式丢失数据最多，最多一个缓冲区
# appendfsync no


~~~

aof还存在一个问题，如果持续往磁盘中aof文件中写入日志记录，随着时间增长，aof的文件也就变得巨大
为解决这个问题，redis有个重写的机制。
什么是重写？
就是将aof文件中，可以抵消的指令抵消掉，只留下精简的指令来就可以恢复当前内存中数据。举例，比如set k1 xx 然后set k1 yy。那么redis重写之后在aof文件中只会留下set k1 yy这个指令，因为前面的操作对于恢复当前内存数据没有帮助。
怎么样重写？
Redis提供两种方式重写，第一种是认为在redis客户端执行

~~~shell
127.0.0.1:6379> BGREWRITEAOF
~~~

第二种是通过redis自动的方式进行重写。

![img](.\images\20210514004)

百分比100，是指redis会记录上次重写的aof文件的大小，当aof文件的大于等于上次重写文件大小的100%（可以改）同时重写时候文件的大小最少是64M（可以改），防止重写的aof文件偏小而频繁进行重写。

听起来，aof比rdb更好点，实际上，在redis4之前使用的是aof。即使你开启rdb也不生效。
但redis4之后就使用rdb和aof混合的方式。rdb负载在某个时刻把内存整个数据dump到磁盘中，aof记录redis从dump开始，对redis key的更改操作。一句话rdb全量，aof增量。
在数据进行重写的时候，redis会把rdb文件先进行重写进aof文件，然后把剩下的aof文件追加到重写的aof文件后面

### 总结

redis4之前使用aof的方式，会产生一个问题，就是aof重写的时间过长。恢复较慢
redis5之后使用rdb和aof的方式，重写的时候先把rdb文件先重写，然后把aof文件追加到重写的文件中，速度较快