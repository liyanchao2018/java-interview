# Redis 的 String 类型 & bitmap



##  1. string类型 & bitmap



### 0. redis二进制安全

redis 是二进制安全的，并不会去破坏你的编码，也不去关心你是什么编码。底层存储的时候，是按照Byte字节存储的。我们前面看到的encoding，只是为了让加减之类的运算方法变得更快一些。
（HBASE也是二进制安全的）

在redis进程与外界交互的时候，redis存储的是字节流，而不会转换成字符流，也不会擅自按照某种数据类型存储，这样保证了数据不会被破坏，不会发生数据被截断/溢出等错误。

编码并不会影响数据的存储
因此，在多人使用redis的时候，我们一定要在用户端约定好数据的编码和解码。

<img src=".\images\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10" alt="img" style="zoom:67%;" />

> 问：Redis单线程是为了减少用户态到内核态的切换吗？
> 答：不是，至少主要原因不是。
> 操作系统为了响应多用户的请求，而进行的从用户态到内核态的切换，造成的的性能损耗，远不及为了保证数据一致性加锁带来的损耗。
> Redis单线程是为了避免加锁的过程。



Redis的string类型分类：

### 1.字符串

#### 1.1Redis client 到Redis请求流程示意图



<img src=".\images\watermark5" alt="img" style="zoom:80%;" />

#### 1.2 Redis string常用命令、数据类型



![img](.\images\watermark4)

#### 1.3 查看帮助 

~~~shell
To get help about Redis commands type:
      "help @<group>" to get a list of commands in <group>
      "help <command>" for help on <command>
      "help <tab>" to get a list of possible help topics
      "quit" to exit
~~~


例如，`help @string`

#### 1.4 字符串操作

``set k1 value1 nx`` nx表示无值的时候才执行成功

``set k1 value1 xx`` xx表示有值的时候才执行成功

``mset k3 value3 k4 value4`` 含义为 more set，用来设置多个值

``mget k1 k2`` 获取两个值

``msetnx k2 c k3 d`` 这个命令可以保证多笔操作是原子操作，类似于一个事务中

``appenk k1 "world" ``在k1后面追加world

``getrange k1 2 5`` 获取k1某两个索引之间的字符串子串（支持正负索引）

#### 1.5 string类型索引

string类型的value的字符，每个字节，设置一个索引位置，同时维护正向/反向 两个索引。

`set k1 hel`

`strlen k1`

<img src=".\images\image-20210429102515433.png" alt="image-20210429102515433" style="zoom:80%;" />

则，value hel 每个字母分别占一个字节,redis string类型中，正反向索引是以字节为粒度。

如下图示意索引下标，正向索引：0 1 2

反向索引：-2 -1



<img src=".\images\watermark" alt="img" style="zoom:65%;" />

``setrange k1 mashibing`` 从某个索引开始覆盖

`strlen k1` 获取字符串长度

`getset k1 bashibing` 取出旧值，并将旧值设置为新值（这样可以减少一次IO通信）



### 2.int数值类型



#### 2.1 查看某个key的类型 

`type k1`查看k1的类型

如果value为1，查看的时候显示是string类型，但是是可以进行int类型的自增操作的：`incr k1`

#### 2.2 查看encoding编码类型 
`object encoding k1`
如果value为1，上面的命令显示为int类型

有一些方法会改变数据的类型

#### 2.3 数值类型的加减操作
`decr k1`    代表： k1-1

`decrby k1 22`   代表：k1+22

`incrbyfloat k1 -0.5`   代表：k1-0.5



### 3.bitmap位图

> bitmap可做登录统计/签到等。

#### 3.1 位图 bitmap

![img](.\images\watermark2)

**bitop按位操作**

#### 3.2 业务场景：

- 公司有用户系统，让你统计用户登录天数，且时间窗口随机。例如，A用户在某一年中登陆了几次。怎么优化？
  可以使用redis实现，假设一年400天，让每一天对应一个二进制位，需要50个字节即可。
  setbit Tom 1 1 表示 Tom 在第2天登录了一次（下标从0开始）
  setbit Tom 364 1 表示 Tom 在第365天登录了一次
  bitcount Tom -2 -1 统计 Tom 在最后16天的总登录次数
- 你是京东，618做活动，登录就送礼物，假设京东有2亿用户。大库应该备货多少礼物？
  用户应该分为：僵尸用户、冷热用户、忠诚用户
  你需要统计活跃用户，也是随机时间窗口
  比如说，统计本月1号~3号范围内的活跃过的用户，需要去重

![img](.\images\watermark3)

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

![image-20210428112253230](D:\学习\面试资料\java-interview\redis\images\image-20210428112253230.png)

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

<img src="D:\学习\面试资料\java-interview\redis\images\image-20210428113420357.png" alt="image-20210428113420357" style="zoom:80%;" />

结果：连续3天内 两人登陆过。



## 2. list



 ### 2.1 list数据结构及应用

![image-20210429164923230](.\images\image-20210429164923230.png)



### 2.2 下图所示，先进后出，实现栈：

<img src=".\images\image-20210429163513625.png" alt="image-20210429163513625" style="zoom:80%;" />

### 2.3 下图所示，实现数组，有序 可重复：

<img src=".\images\image-20210429163405024.png" alt="image-20210429163405024" style="zoom:75%;" />