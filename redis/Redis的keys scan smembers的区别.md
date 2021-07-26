

## [redis中scan和keys的区别](https://www.cnblogs.com/guanbin-529/p/12741638.html)

**目录**

- [scan和keys的区别](https://www.cnblogs.com/guanbin-529/p/12741638.html#_label0)
- [java中使用redisTemplate实现](https://www.cnblogs.com/guanbin-529/p/12741638.html#_label1)

 

------

![img](D:\学习\面试资料\java-interview\redis\images\1089423-20200420235333775-187454879.png)

 

## scan和keys的区别

​		redis的keys命令，通来在用来删除相关的key时使用，但这个命令有一个弊端，在redis拥有数**百万及以上的keys**的时候，会执行的比较慢，更为致命的是，这个**命令会阻塞redis多路复用的io主线程**，如果这个线程阻塞，在此执行之间其他的发送向redis服务端的命令，都会阻塞，从而引发一系列级联反应，导致瞬间响应卡顿，从而引发超时等问题，所以应该在生产环境禁止用使用keys和类似的命令smembers，这种时间复杂度为O（N），且会阻塞主线程的命令，是非常危险的。

**keys命令的原理就是扫描整个redis里面所有的db的key数据**，然后根据我们的**通配的字符串进行模糊查找**出来。官网详细的介绍如下。

```http
https://redis.io/commands/KEYS
```

取而代之的，如果需要**查找然后删除key的需求，那么在生产环境我们应该使用scan命令，代替keys命令**，同样是O（N）复杂度的**scan命令，支持通配查找**，scan命令或者其他的scan如**SSCAN** ，**HSCAN**，**ZSCAN**命令，可以不用阻塞主线程，并支持游标按批次迭代返回数据，所以是比较理想的选择。**keys相比scan命令优点是，keys是一次返回，而scan是需要迭代多次返回**。

```http
https://redis.io/commands/scan
```

但**scan命令的也有缺点**，**返回的数据有可能重复，需要我们在业务层按需要去重**，scan命令的游标从0开始，也从0结束，每次返回的数据，都会返回下一次游标应该传的值，我们根据这个值，再去进行下一次的访问，如果返回的数据为空，并不代表没有数据了，只有游标返回的值是0的情况下代表结束。

redis命令例子如下：

```shell
scan 0 match my*key count 10000
```

在Java项目里面，使用jedis执行scan命令的模板例子如下：

~~~java
		Jedis jedis = getJedis();               //存储返回的结果                
        Set<String> result=new HashSet<String>();                //设置查询的参数                
        ScanParams scanParams=new ScanParams().count(scanLimitSize).match(pattern);            //游标的开始                
        String cur=ScanParams.SCAN_POINTER_START;                
        do{                   //遍历每一批数据                    
           ScanResult<String> scan=jedis.scan(cur, scanParams);                    //得到结果返回
            List<String> scanResult =scan.getResult();                    
            result.addAll(scanResult);                    //获取新的游标                    
            cur=scan.getStringCursor();                //判断游标迭代是否结束                    
         }while (!cur.equals(ScanParams.SCAN_POINTER_START));                //返回结果                
         return result;
~~~

## [redis中KEYS、SMEMBERS、SCAN 、SSCAN 的区别](https://www.cnblogs.com/zhuyeshen/p/12496406.html)

​		在看项目中大神写的框架中关于redis存储相关代码时，发现了再获取set数据类型的全部元素时，采用的是sscan函数，而不是采用的smembers函数，这两个到底有什么区别呢？

先看这两个命令：

keys：用于获取当前数据库的模式匹配的所有key

smembers：获取set集合中的所有元素

而scan又包含多个类似命令

SCAN 增量迭代当前数据库中的数据库键。
SSCAN 增量迭代集合键中的元素。
HSCAN 增量迭代哈希键中的键值对。
ZSCAN 增量迭代有序集合中的元素（包括元素成员和元素分值）。
也就是说，keys、smembers和scan家族命令的最大区别是：

​     **keys和smembers是获取全部**，如果**当redis中key的数量过去庞大**（或者set的元素很多），则很耗费内存，**会阻塞redis几秒钟,或者链接超时**。

​     scan家族是逐步增量获取。即遍历获取一定数量的key或者元素，在获取一定数量的key或元素，不会一次获取。

那么，scan命令就比keys、smembers命令好吗？不是这样的，scan命令家族也是有缺点的。由于scan采用的增量迭代，当redis中的key是随时变化的，比如key增加减少或者key的名字变更，这种情况，scan就暴露他的弊端了，可能无法获取所有的key了。

所以采用哪种方式获取key或者获取元素，得根据自己的业务，如果你key是随时变化，就采用keys或者smembers吧。因为我们业务中redis的初始化只是在项目启动时初始化一次，所以在获取set的全部元素时采用的sscan命令。