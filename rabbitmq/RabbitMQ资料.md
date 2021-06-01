### 1.RabbitMQ流控

#### 1.1服务端流控（Flow Control）：

> https://www.rabbitmq.com/configure.html 
>
> https://www.rabbitmq.com/flow-control.html 
>
> https://www.rabbitmq.com/memory.html 
>
> https://www.rabbitmq.com/disk-alarms.html 
>
> 当 RabbitMQ 生产 MQ 消息的速度远大于消费消息的速度时，会产生大量的消息堆积，占用系统资源，导致机器的性能下降。我们想要控制服务端接收的消息的数量，应该怎么做呢？

##### 1.1.1 队列有两个控制长度的属性： 

**x-max-length：**

​		队列中最大存储最大消息数，超过这个数量，队头的消息会被丢弃。 

**x-max-length-bytes：**

​		队列中存储的最大消息容量（单位 bytes），超过这个容量，队头的消息会被丢弃      

![image-20210318151938844](D:\学习\面试资料\java-interview\rabbitmq\images\image-20210318151938844.png)

需要注意的是，设置队列长度只在消息堆积的情况下有意义，而且会删除先入队的消息，不能真正地实现服务端限流。 

有没有其他办法实现服务端限流呢？ 



##### 1.1.2 内存控制

> RabbitMQ 会在启动时检测机器的物理内存数值。
>
> 默认当 MQ 占用 40% 以上内存时，MQ 会主动抛出一个内存警告并阻塞所有连接（Connections）。
>
> 可以通过修改rabbitmq.config 文件来调整内存阈值，默认值是 0.4，

如下所示：  

~~~properties
[{rabbit, [{vm_memory_high_watermark, 0.4}]}].
~~~

也可以用命令动态设置，如果设置成 0，则所有的消息都不能发布。

~~~properties
rabbitmqctl set_vm_memory_high_watermark 0.3
~~~



##### 1.1.3 磁盘控制

另一种方式是通过磁盘来控制消息的发布。当磁盘空间低于指定的值时（默认 50MB），触发流控措施。

例如：指定为磁盘的 30%或者 2GB： 

https://www.rabbitmq.com/configure.html 

~~~properties
disk_free_limit.relative = 3.0 
disk_free_limit.absolute = 2GB
~~~



#### 1.2 消费端控制流控： 

##### 1.2.1 basicQos

##### 1.2.2 basicConsume

**prefetch count（预取消息数量），控制消息消费端预取数量的消息 未 给broker处理结果恢复，则不再去队列取消息。** 

> https://www.rabbitmq.com/consumer-prefetch.html 
>
> ​		默认情况下，如果不进行配置，RabbitMQ 会尽可能快速地把队列中的消息发送到消费者。因为消费者会在本地缓存消息，如果消息数量过多，可能会导致 OOM 或者影响其他进程的正常运行。  
>
> ​		在消费者处理消息的能力有限，例如消费者数量太少，或者单条消息的处理时间过长的情况下，如果我们希望在一定数量的消息消费完之前，不再推送消息过来，就要用到消费端的流量限制措施。  
>
> ​		可以基于 Consumer 或者 channel 设置 prefetch count 的值，含义为 Consumer端的最大的 unacked messages 数目。当超过这个数值的消息未被确认，RabbitMQ 会停止投递新的消息给该消费者。  

~~~properties
channel.basicQos(2); // 如果超过 2 条消息没有发送 ACK，当前消费者不再接受队列消息 channel.basicConsume(QUEUE_NAME, false, consumer);
~~~







### 2.RabbitMQ消息可靠性投递



> RabbitMQ工作模型，1.2.3.4各节点容易出现投递不成功的情况。

![image-20210319172039891](D:\学习\面试资料\java-interview\rabbitmq\images\image-20210319172039891.png)



#### 2.1服务端确认(消息可靠性投递)

> 如何保证生产者发送Message到Broker不丢失
>
> 请参考文章，可解读2.1*的所有内容：[RabbitMQ系列（四）RabbitMQ事务和Confirm发送方消息确认——深入解读.md](https://www.cnblogs.com/vipstone/p/9350075.html)

上述文章简单描述：

##### 2.1.1 服务端确认-Transaction模式 

Transaction的实现主要是对信道（Channel）的设置，主要的方法有三个：

1. channel.txSelect()声明启动事务模式；
2. channel.txComment()提交事务；
3. channel.txRollback()回滚事务；

<img src="D:\学习\面试资料\java-interview\rabbitmq\images\image-20210319162444725.png" alt="image-20210319162444725" style="zoom:50%;" />

##### 2.1.2 服务端确认-Confirm模式

> Confirm发送方确认模式使用和事务类似，也是通过设置Channel进行发送方确认的。
>
> **Confirm的三种实现方式：**
>
> ~~~java
> // 将channel设置为Confirm模式 
> channel.confirmSelect(); 
> 
> if (channel.waitForConfirms()) {
> // 消息发送成功 
>     
> }
> ~~~
>
> <img src="D:\学习\面试资料\java-interview\rabbitmq\images\image-20210319165929064.png" alt="image-20210319165929064" style="zoom:90%;" />
>
> 此代码方式，适用方式一/方式二。
>
> 方式一：channel.waitForConfirms()普通发送方确认模式；
>
> 方式二：channel.waitForConfirmsOrDie()批量确认模式；
>
> 方式三：channel.addConfirmListener()异步监听发送方确认模式；----->对应springboot rabbitTemplate.setReturnCallback方法。





#### 2.2 路由保证(消息可靠性投递)

##### 2.2.1、mandatory = true + ReturnListener

> mandatory   -->  rabbitTemplate.setMandatory(true);
>
> ReturnListener  -->  rabbitTemplate.setReturnCallback ( new RabbitTemplate.ReturnCallback(){......});

~~~java
@Configuration
public class TemplateConfig {
    @Bean
    public ConnectionFactory connectionFactory() throws Exception {
        CachingConnectionFactory cachingConnectionFactory = new CachingConnectionFactory();
        cachingConnectionFactory.setUri(ResourceUtil.getKey("rabbitmq.uri"));
        return cachingConnectionFactory;
    }

    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
        return new RabbitAdmin(connectionFactory);
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
/***
* 路由保证producer发送的消息，正确路由到queue中。--start
*/
        rabbitTemplate.setMandatory(true);
        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback(){
            public void returnedMessage(Message message,
                                        int replyCode,
                                        String replyText,
                                        String exchange,
                                        String routingKey){
                System.out.println("回发的消息：");
                System.out.println("replyCode: "+replyCode);
                System.out.println("replyText: "+replyText);
                System.out.println("exchange: "+exchange);
                System.out.println("routingKey: "+routingKey);
            }
        });
/***
* 路由保证producer发送的消息，正确路由到queue中。--end
*/
        
        
/***
* 服务端设置Transaction模式，保证producer发送的消息，broker能够接收到。
*/
        rabbitTemplate.setChannelTransacted(true);
/***
* 服务端设置  异步Confirm模式，异步监听broker已收到消息 和未收到的消息。
*/
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                if (!ack) {
                    System.out.println("发送消息失败：" + cause);
                    throw new RuntimeException("发送异常：" + cause);
                }
            }
        });

        return rabbitTemplate;
    }
}
~~~



##### 2.2.2、指定交换机的备份交换机

消息路由到备份交换机的方式：在创建交换机的时候，从属性中指定备份交换机。 

~~~java
Map<String,Object> arguments = new HashMap<String,Object>(); 

arguments.put("alternate-exchange","ALTERNATE_EXCHANGE"); // 指定交换机的备份交换机 

channel.exchangeDeclare("TEST_EXCHANGE","topic", false, false, false, arguments); 
~~~

（注意区别，队列可以指定死信交换机；交换机可以指定备份交换机） 

参考： 

gupaoedu-vip-rabbitmq-javaapi：com.gupaoedu.returnlistener 

gupaoedu-vip-springboot-amqp：com.gupaoedu.amqp.template.TemplateConfig 





### 3.TTL(Time To Live)

#### 3.1 消息的过期时间

有两种设置方式： 

**1） 通过队列属性设置消息过期时间** 

所有队列中的消息超过时间未被消费时，都会过期。 

代码位置 ：com.gupaoedu.ttl.TtlConfig.java

~~~java
@Bean("ttlQueue") 
public Queue queue() { 
	Map<String, Object> map = new HashMap<String, Object>(); 
	map.put("x-message-ttl", 11000); // 队列中的消息未被消费 11 秒后过期 
    return new Queue("GP_TTL_QUEUE", true, false, false, map); 
}
~~~

**2） 设置单条消息的过期时间** 

在发送消息的时候指定消息属性。 

代码位置：com.gupaoedu.ttl.TtlSender.java

~~~java
MessageProperties messageProperties = new MessageProperties(); messageProperties.setExpiration("4000"); // 消息的过期属性，单位 ms 
Message message = new Message("这条消息 4 秒后过期".getBytes(), messageProperties); rabbitTemplate.send("GP_TTL_EXCHANGE", "gupao.ttl", message);
~~~

**如果同时指定了 Message TTL 和 Queue TTL，则小的那个时间生效**



### 4.死信队列

> **死信队列适用于下单后，未付款，超时系统自动取消订单功能。**
>
> (同时适用不同业务订单的超时设置不同的死信队列)
>
> 如图，代码所示：
>
> <img src="D:\学习\面试资料\java-interview\rabbitmq\images\image-20210325154020689.png" alt="image-20210325154020689" style="zoom:50%;" />



​		消息在某些情况下会变成死信（Dead Letter）。队列在创建的时候可以指定一个死信交换机DLX（Dead Letter Exchange）。死信交换机绑定的队列被称为死信队列DLQ（Dead Letter Queue），DLX实际上也是普通的交换机，DLQ也是普通的队列。 

<img src="D:\学习\面试资料\java-interview\rabbitmq\images\dead_letter_exchange" alt="img" style="zoom:50%;" /> 



#### 4.1 什么情况下消息会变成死信？ 

1）消息被消费者拒绝并且未设置重回队列：

​	  (NACK || Reject ) && requeue == false

2）消息过期 

3）队列达到最大长度，超过了 Max length（消息数）或者 Max length bytes（字节数），最先入队的消息会被发送到 DLX。 



#### 4.2 死信队列实现过程

##### 1. 声明死信交换机 DEAD_LETTER_EXCHANGE 、死信队列 DEAD_LETTER_QUEUE，相互绑定。

~~~java
@Bean("deadLetterExchange")
public TopicExchange deadLetterExchange() {
    return new TopicExchange("GP_DEAD_LETTER_EXCHANGE", true, false, new HashMap<>()); 
}

@Bean("deadLetterQueue")
public Queue deadLetterQueue() {
    return new Queue("DEAD_LETTER_QUEUE", true, false, false, new HashMap<>()); 
}

@Bean
public Binding bindingDead(@Qualifier("deadLetterQueue") Queue queue, @Qualifier("deadLetterExchange") TopicExchange exchange) {
    return BindingBuilder.bind(queue).to(exchange).with("#"); // 无条件路由 
}
~~~



##### 2.声明交换机 SIMPLE_EXCHANGE、队列 SIMPLE_QUEUE，相互绑定。

##### 3.指定队列的死信交换机 DEAD_LETTER_EXCHANGE。

##### 4.队列中的消息10秒钟过期，因为没有消费者，会变成死信，通过死信交换机进入死信队列。

~~~java
@Bean("simpleExchange")
public DirectExchange exchange() {
    return new DirectExchange("SIMPLE_EXCHANGE", true, false, new HashMap<>()); 
}

@Bean("simpleQueue") 
public Queue queue() {
    Map<String, Object> map = new HashMap<>();
    // 10 秒钟后成为死信
    map.put("x-message-ttl", 10000); 
    // 队列中的消息变成死信后，进入死信交换机
    map.put("x-dead-letter-exchange", "DEAD_LETTER_EXCHANGE"); 
    return new Queue("SIMPLE_QUEUE", true, false, false, map); 
}

@Bean
public Binding binding(@Qualifier("simpleQueue") Queue queue,@Qualifier("simpleExchange") DirectExchange exchange) {
    return BindingBuilder.bind(queue).to(exchange).with("hujy.test"); 
}
~~~













### 5.RabbitMQ消费端Push/Pull模式讲解

> 消息中间件push和pull模式
>
> **Push/Pull,均是针对于 消费者 获取消息 的方式而言**
>
> 具体Push/Pull模式详细使用文档参考：[RabbitMQ之Consumer消费模式（Push & Pull）.md](https://honeypps.com/mq/rabbitmq-consumer-consumption-pattern/)

#### 5.1 概念

MQ的消费模式分两种：push和pull。

所谓push就是服务端主动推送消息给客户端，而pull则是客户端需要主动到服务端取数据。

#### 5.2 不同消息中间件支持的模式 

 ![img](D:\学习\面试资料\java-interview\rabbitmq\images\webp) 



#### 5.3 Push模式



##### 5.3.1 push模式编码

发送端向broker端发送数据，数据内容为：

RabbitMQ Demo Test, Send Messages 0；

RabbitMQ Demo Test, Send Messages 1；

RabbitMQ Demo Test, Send Messages 2，一次类推……

Push模式消费端的示例代码（Push模式）：

> channel.basicConsume代表使用push订阅模式

~~~java
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();
QueueingConsumer consumer = new QueueingConsumer(channel);
channel.basicQos(1);
channel.basicConsume(QUEUE_NAME, false, "consumer_zzh",consumer);

while (true) {
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();
    String message = new String(delivery.getBody());
    System.out.println(" [X] Received '" + message + "'");
    channel.basicAck(delivery.getEnvelope().getDeliveryTag(),false);
    break;
}
channel.close();
connection.close();
~~~



##### 5.3.2 push优点：

服务端主动推送给客户端，及时性很高

##### 5.3.3 push缺点：

1.当客户端消费能力远低于服务端生产能力，那么一旦服务端推送大量消息到客户端时，就会导致客户端消息堆积，处理缓慢，甚至服务崩溃。（那么如何解决这个问题呢？需要mq提供流控制，也就是依据客户端消费能力做流控。比如rabbitmq设置Qos，限制消费数量。）

2.服务端需要维护每次传输状态，以防消息传递失败进行重试。





#### 5.4 Pull模式



##### 5.4.1 Pull模式编码

pull模式主要是通过**channel,basicGet**方法来获取消息，示例代码如下：

```java
GetResponse response = channel.basicGet(QUEUE_NAME, false);
System.out.println(new String(response.getBody()));
channel.basicAck(response.getEnvelope().getDeliveryTag(),false);
```



##### 5.4.2 pull模式优点：

1.客户端可以依据自己的消费能力进行消费

2.传输失败时不需要重试，反正数据还在服务端。

##### 5.4.3 pull模式缺点：

1.主动到服务端拉取消息。这个拉取消息的间隔需要设置好，不太好设置。间隔太短，对服务器请求压力过大。间隔时间过长，那么必然会造成一部分数据的延迟。（也有一些解决方案，间隔时间指数级增长，5ms，10ms，20ms，40ms，80ms。。。然后再回到5ms，一定程度上解决，但是如果在41ms时来了数据，那么到80ms就有40ms左右的时间延迟。另外在腾讯的CMQ里有一套长轮询的解决方案，就是取数据时要是没有数据可消费，不是直接返回而是连接等待，一直有数据来了再返回）



#### 5.5 push和pull模式不同适用场景

对于服务端生产消息数据比较大时，而消费端处理比较复杂，消费能力相对较低时，这种情况就适用pull模式。

对于数据实时性要求高的场景，就比较适用与push模式。



### 6.多费者共同监听同一个队列，同一消息会不会发到多个消费者

> 问题：
>
> RabbitMQ的一个队列多个消费者消费【如果部署两个消费者共同消费同一个队列】，会不会出现多个消费者消费了同一条信息？有的话如何避免？
>
> 当然，这个问题的前提是用direct模式，fanout模式也可以。

答：JavaEdge微信好友给出的答案：
JavaEdge:
不会
默认都是消费者竞争去抢消息的
拉不就是消费者去拉取消息嘛一般都是 mq 默认的机制
不然就得手动发消息给消费者，也就是推模式
一般就用默认拉啥都不需要干
配好了广播模式（fanout）或者直连模式（direct）就行



#### 6.1RabbitMQ中@RabbitListener/@RabbitMQTemplate使用的是Pull or Push模式

> @RabbitListener --> SimpleRabbitListenerContainerFactory  这种方式 使用的是pull还是push方式





### 7. AcknowledgeMode.MANUAL/AUTO

```properties
AcknowledgeMode.MANUAL  #手动确认模式
AcknowledgeMode.AUTO    #自动确认模式
AcknowledgeMode.NONE    #无ack模式
```

自动的ACK

​		1.consumer接收到消息，就自动发送ack确认消息。还是2.执行完consumer的所有方法，之后，自动发送ack给broker？

答案：选1。AutoACK，consumer接收到消息就发送ACK

[原文链接](https://blog.csdn.net/weixin_38380858/article/details/84963944)

**无ack消费模式与有ack消费模式对比**

####  7.1 无ack模式

> 无ack模式（AcknowledgeMode.NONE）

**server端行为**
rabbitmq server默认推送的所有消息都已经消费成功，会不断地向消费端推送消息。
因为rabbitmq server认为推送的消息已被成功消费，所以推送出去的消息不会暂存在server端。
消息丢失的风险
当BlockingQueue<Runnable>堆满时（BlockingQueue<Delivery>一定会先满），server端推送消息会失败，然后断开connection。消费端从Socket读取Frame将会抛出SocketException，触发异常处理，shutdown掉connection和所有的channel，channel shutdown后WorkPool中的channel信息（包括channel inProgress,channel ready以及Map）全部清空，所以BlockingQueue<Runnable>中的数据会全部丢失。

此外，服务重启时也需对内存中未处理完的消息做必要的处理，以免丢失。

而在rabbitmq server，connection断掉后就没有消费者去消费这个queue，因此在server端会看到消息堆积的现象。

#### 7.2 有ack模式

> 有ack模式（AcknowledgeMode.AUTO，AcknowledgeMode.MANUAL）

**AcknowledgeMode.MANUAL模式：**

​		需要人为地获取到channel之后调用方法向server发送ack（或消费失败时的nack）信息。

**AcknowledgeMode.AUTO模式：**

​		由spring-rabbit依据消息处理逻辑是否抛出异常自动发送ack（无异常）或nack（异常）到server端。



server端行为
rabbitmq server推送给每个channel的消息数量有限制，会保证每个channel没有收到ack的消息数量不会超过prefetchCount。
server端会暂存没有收到ack的消息，等消费端ack后才会丢掉；如果收到消费端的nack（消费失败的标识）或connection断开没收到反馈，会将消息放回到原队列头部。
这种模式不会丢消息，但效率较低，因为server端需要等收到消费端的答复之后才会继续推送消息，当然，推送消息和等待答复是异步的，可适当增大prefetchCount提高效率。

注意，有ack的模式下，需要考虑setDefaultRequeueRejected(false)，否则当消费消息抛出异常没有catch住时，这条消息会被rabbitmq放回到queue头部，再被推送过来，然后再抛异常再放回…死循环了。设置false的作用是抛异常时不放回，而是直接丢弃，所以可能需要对这条消息做处理，以免丢失。更详细的配置参考这里。

对比
无ack模式：效率高，存在丢失大量消息的风险。
有ack模式：效率低，不会丢消息。









### 8.RabbitMQ的三种交换机

> 详细文档参考：Rabbit MQ的三种exchange.md

1. Direct Exchange – 处理路由键。
2. Fanout Exchange – 不处理路由键。
3. Topic Exchange – 将路由键和某模式进行匹配。



### 9. RabbitMQ集群（普通模式、镜像模式）



#### 9.1 普通模式

> 默认集群模式，以两个几点（rabbit01、rabbit02）为例进行说明。
>
> 对于队列来说，消息实体只存在于其中一个节点rabbit01，rabbit01和rabbit02两个节点有相同的数据。
>
> 当消息进入rabbit01节点的队列后，若消费者从2节点消费，则rabbitmq会临时在rabbit01、rabbit02间进行消息传输，把A中的消息实体取出并经过B发送给consumer。
>
> 所以consumer应尽量连接每一个节点，从中取消息。即对于同一个逻辑队列，要在多个节点建立物理queue。否则无论consumer连rabbit01还是rabbit02，出口总在rabbit01，会产生瓶颈。
>
> 当rabbit01节点故障后，rabbit02节点无法取到rabbit01节点中还未消费的消息实体。如果做了消息持久化，那么得等rabbit01 节点恢复，然后才可被消费；如果没有持久化的话，就会产生消息丢失的现象

普通集群模式下，不同的节点之间只会相互同步元数据。

 ![img](D:\学习\面试资料\java-interview\rabbitmq\images\mq_cluster01) 



**疑问：为什么不直接把队列的内容（消息）在所有节点上复制一份？** 

​		主要是出于存储和同步数据的网络开销的考虑，如果所有节点都存储相同的数据， 

就无法达到线性地增加性能和存储容量的目的（堆机器）。 

​		假如生产者连接的是节点 3，要将消息通过交换机 A 路由到队列 1，最终消息还是会 

转发到节点 1 上存储，因为队列 1 的内容只在节点 1 上。 

同理，如果消费者连接是节点 2，要从队列 1 上拉取消息，消息会从节点 1 转发到 

节点 2。其它节点起到一个路由的作用，类似于指针。 

**普通集群模式不能保证队列的高可用性，因为队列内容不会复制。**

如果节点失效将 导致相关队列不可用，因此我们需要第二种集群模式。









#### 9.2 镜像模式

第二种集群模式叫做镜像队列。 

镜像队列模式下，消息内容会在镜像节点间同步，可用性更高。不过也有一定的副作用，系统性能会降低，节点过多的情况下同步的代价比较大。

> 基于普通模式集群，在策略里面添加镜像策略即可。
>
> 把需要的队列做成镜像队列，属于HA方案。该模式解决了普通模式中的问题，根本区别在于消息实体会主动在镜像节点间同步，而不是在客户端拉取数据时临时拉取。当然弊端就是降低系统性能，大量消息写入，集群内的网路带宽会被大大消耗。 

  

**操作方式** 

**命令或步骤** 

| **操作方式**          | **命令或步骤**                                               |
| --------------------- | ------------------------------------------------------------ |
| rabbitmqctl (Windows) | rabbitmqctl set_policy ha-all "^ha." "{""ha-mode"":""all""}" |
| HTTP API              | PUT /api/policies/%2f/ha-all {"pattern":"^ha.", "definition":{"ha-mode":"all"}} |
| Web UI                | 1、avigate to Admin > Policies > Add / update a policy       |
|                       | 2、Name 输入：mirror_image                                   |
|                       | 3、Pattern 输入：^（代表匹配所有）                           |
|                       | 4、Definition 点击 HA mode，右边输入：all                    |
|                       | 5、Add policy                                                |





 #### 9.3 高可用

集群搭建成功后，如果有多个内存节点，那么生产者和消费者应该连接到哪个内存 

节点？如果在我们的代码中根据一定的策略来选择要使用的服务器，那每个地方都要修 

改，客户端的代码就会出现很多的重复，修改起来也比较麻烦。



![image-20210323174858465](D:\学习\面试资料\java-interview\rabbitmq\images\image-20210323174858465.png)



所以需要一个负载均衡的组件（例如 HAProxy，LVS，Nignx），由负载的组件来做 

路由。这个时候，只需要连接到负载组件的 IP 地址就可以了。 

![image-20210323174932278](D:\学习\面试资料\java-interview\rabbitmq\images\image-20210323174932278.png)

负载分为四层负载和七层负载。 

![image-20210323174958146](D:\学习\面试资料\java-interview\rabbitmq\images\image-20210323174958146.png)

四层负载：工作在 OSI 模型的第四层，即传输层（TCP 位于第四层），它是根据 IP 

端口进行转发（LVS 支持四层负载）。RabbitMQ 是 TCP 的 5672 端口。 

七层负载：工作在第七层，应用层（HTTP 位于第七层）。可以根据请求资源类型分 

配到后端服务器（Nginx 支持七层负载；HAProxy 支持四层和七层负载）。 

https://www.cnblogs.com/readygood/p/9757951.html 

但是，如果这个负载的组件也挂了呢？客户端就无法连接到任意一台 MQ 的服务器 

了。所以负载软件本身也需要做一个集群。新的问题又来了，如果有两台负载的软件， 

客户端应该连哪个？ 

![image-20210323175037817](D:\学习\面试资料\java-interview\rabbitmq\images\image-20210323175037817.png)

负载之上再负载？陷入死循环了。这个时候我们就要换个思路了。 

我们应该需要这样一个组件： 

1、 它本身有路由（负载）功能，可以监控集群中节点的状态（比如监控 

HAProxy），如果某个节点出现异常或者发生故障，就把它剔除掉。 

2、 为了提高可用性，它也可以部署多个服务，但是只有一个自动选举出 

来的 MASTER 服务器（叫做主路由器），通过广播心跳消息实现。 

3、 MASTER 服务器对外提供一个虚拟 IP，提供各种网络功能。也就是 

谁抢占到 VIP，就由谁对外提供网络服务。应用端只需要连接到这一 

个 IP 就行了。 

这个协议叫做 VRRP 协议（虚拟路由冗余协议 Virtual Router Redundancy 

Protocol），这个组件就是 Keepalived，它具有 Load Balance 和 High Availability 

的功能。

下面我们看下用 HAProxy 和 Keepalived 如何实现 RabbitMQ 的高可用 

（MySQL、Mycat、Redis 类似）。 



#### 9.3.1**HAproxy 负载+Keepalived 高可用**

<img src="D:\学习\面试资料\java-interview\rabbitmq\images\image-20210323175200831.png" alt="image-20210323175200831" style="zoom:50%;" />

规划： 

内存节点 1：192.168.8.40 

内存节点 2：192.168.8.45 

磁盘节点：192.168.8.150 

VIP：192.168.8.220 

1、我们规划了两个内存节点，一个磁盘节点。所有的节点之间通过镜像队列的 

方式同步数据。内存节点用来给应用访问，磁盘节点用来持久化数据。 

2、为了实现对两个内存节点的负载，我们安装了两个 HAProxy，监听两个 5672 

和 15672 的端口。 

3、安 装 两 个 Keepalived ， 一 主 一 备 。 两 个 Keepalived 抢 占 一 个 

VIP192.168.8.220。谁抢占到这个 VIP，应用就连接到谁，来执行对 MQ 的负载。 

这种情况下，我们的 Keepalived 挂了一个节点，没有影响，因为 BACKUP 会变 

成 MASTER，抢占 VIP。HAProxy 挂了一个节点，没有影响，我们的 VIP 会自动路 

由的可用的 HAProxy 服务。RabbitMQ 挂了一个节点，没有影响， 因为 HAProxy 

会自动负载到可用的节点



### 10. **如何减少连接数** 

在发送大批量消息的情况下，创建和释放连接依然有不小的开销。

**我们可以跟接收方约定批量消息的格式，比如支持 JSON 数组的格式，通过合并消息内容，可以减少生** 

**产者/消费者与 Broker 的连接。**

比如：活动过后，要全范围下线产品，通过 Excel 导入模板，通常有几万到几十万条 

解绑数据，合并发送的效率更高。 

**建议单条消息不要超过 4M（4096KB），一次发送的消息数需要合理地控制**

> 示例：

~~~json
msgContent = 
		{ "type":"add", "num":3, "detail": 
           [ 
               { "merchName":"黄金手机店", "address":"黄金路 999 号" ] }, 
               { "merchName":"银星手机店", "address":"银星路 168 号" ] }, 
               { "merchName":"青山手机店", "address":"青山路 73 号" ] } 
            ] 
          }
~~~



### 11.**面试题**

2.Channel 和 vhost 的作用是什么？ 

​	Channel：减少 TCP 资源的消耗。也是最重要的编程接口。 

​	Vhost：提高硬件资源利用率，实现资源隔离。 

3、RabbitMQ 的消息有哪些路由方式？适合在什么业务场景使用？ 

​	Direct、Topic、Fanout 

4、交换机与队列、队列与消费者的绑定关系是什么样的？ 

​		多对多； 

​	多个消费者监听一个队列时（比如一个服务部署多个实例），消息会重复消费吗？ 

  	不会，轮询（平均分发） 

5、 无法被路由的消息，去了哪里？ 

​		直接丢弃。可用备份交换机（alternate-exchange）接收。 

6、 消息在什么时候会变成 Dead Letter（死信）？ 

​		消息过期；消息超过队列长度或容量；消息被拒绝并且未设置重回队列 

7、 如果一个项目要从多个服务器接收消息，怎么做？ 

​		如果一个项目要发送消息到多个服务器，怎么做？ 

​		定义多个 ConnectionFactory，注入到消费者监听类/Temaplate。 

8、 RabbitMQ 如何实现延迟队列？ 

​		基于数据库+定时任务； 

​		或者消息过期+死信队列； 

​		或者延迟队列插件。 

9、 哪些情况会导致消息丢失？怎么解决？ 

​		哪些情况会导致消息重复？怎么解决？ 

​		从消息发送的整个流程来分析。 

10、 一个队列最多可以存放多少条消息？ 

​		由硬件决定。 

11、 可以用队列的 x-max-length 最大消息数来实现限流吗？例如秒杀场景。 

​		不能，因为会删除先入队的消息，不公平。 

12、 如何提高消息的消费速率？ 

​		创建多个消费者。 

13、 AmqpTemplate 和 RabbitTemplate 的区别？ 

​		Spring AMQP 是 Spring 整合 AMQP 的一个抽象。Spring-Rabbit 是一个实现。 

 

14、 如何动态地创建消费者监听队列？ 

通过 ListenerContainer 

com.gupaoedu.amqp.container.ContainerConfig.java 

container.setQueues(getSecondQueue(), getThirdQueue()); //监听的队列 

15、 Spring AMQP 中消息怎么封装？用什么转换？ 

Message，MessageConvertor 

16、 如何保证消息的顺序性？ 

一个队列只有一个消费者 

17、 RabbitMQ 的集群节点类型？ 

磁盘节点和内存节点 

18、 如何保证 RabbitMQ 的高可用？ 

HAProxy（LVS）+Keepalived 

19、 大量消息堆积怎么办？ 

1） 重启（不是开玩笑的） 

2） 多创建几个消费者同时消费

3） 直接清空队列，重发消息 