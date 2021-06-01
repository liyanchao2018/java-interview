# RabbitMQ之Consumer消费模式（Push & Pull）

# 1. 概述

消息中间件有很多种，进程也会拿几个来对比对比，其中一种对比项就是消费模式。消息的消费模式分Push、Pull两种，或者两者兼具。RabbitMQ的消费模式就是兼具Push和Pull。

本文通过demo代码以及借助wireshark抓包工具来观察RabbitMQ的消费模式。



# 2. push模式

发送端向broker端发送数据，数据内容为：RabbitMQ Demo Test, Send Messages 0；RabbitMQ Demo Test, Send Messages 1；RabbitMQ Demo Test, Send Messages 2，一次类推……

下面是消费端的示例代码：

```java
Connection connection = factory.newConnection();
Channel channel = connection.createChannel();
QueueingConsumer consumer = new QueueingConsumer(channel);
channel.basicQos(1);
channel.basicConsume(QUEUE_NAME, false, "consumer_zzh",consumer);
while (true) {    
    QueueingConsumer.Delivery delivery = consumer.nextDelivery();    
    String message = new String(delivery.getBody());    
    System.out.println(" [X] Received '" + message + "'");          channel.basicAck(delivery.getEnvelope().getDeliveryTag(),false);    
    break;
}
channel.close();
connection.close();
```

运行输出：RabbitMQ Demo Test, Send Messages 0

通过wirkshark工具来查看上面示例代码的整个AMQP的过程（图1）：
![img](http://image.honeypps.com/images/papers/2017/144.png)

上图可以对照实例代码来看，比如图中：Basic.Qos和Basic.Qos-Ok就是示例代码中的：channel.basicQos(1);

再比如图中的Basic.Ack就是示例代码中的channel.basicAck(delivery.getEnvelope().getDeliveryTag(),false);

对于上图中的带蓝色背影的那行（即No.545，展开如下图，broker to client）
[![img](http://image.honeypps.com/images/papers/2017/145.png)

可以看到Delivery-Tag = 1, 消息内容：“RabbitMQ Demo Test, Send Messages 0”。

展开No.546，即Basic.Ack这行：
![img](http://image.honeypps.com/images/papers/2017/146.png)

可以看到client to broker的Ack:delievery-tag=1,和上面的符合。

仔细的朋友可以看到No.548同样是broker向client发送了一条数据，通过展开数据可知：
![img](http://image.honeypps.com/images/papers/2017/147.png)

其内包含的数据正好是下一条的消息——“RabbitMQ Demo Test, Send Messages 1”。但是在运行实例的时候是没有打印出来的，可以看图1，是broker端主动向client端发送的数据，client端没有请求。在broker端发送第一条数据，即”RabbitMQ Demo Test, Send Messages 0”之后发送Ack，之后关闭Channel,到真正关闭完channel之间，broker端还是会发送(push)数据给Client, 此时Client不会在Ack此条数据了。那么这样这条消息会不会丢失呢？答案是否定的，你可以再运行下consumer程序，就能消费到这条消息，rabbitmq对设置autoAck=false之后没有被Ack的消息是不会清除掉的。

> 实际上如果不设置channel.basicQos(1),那么broker端会一次推送多条数据
> RabbitMQ的每一数据帧（Frame）都是以0xCE结尾。

------

# 3. pull模式

同样采用wirkshark工具来观察pull模式的AMQP过程，pull模式主要是通过channel,basicGet方法来获取消息，示例代码如下：

```
GetResponse response = channel.basicGet(QUEUE_NAME, false);System.out.println(new String(response.getBody()));channel.basicAck(response.getEnvelope().getDeliveryTag(),false);
```

wireshark抓包结果：
[![img](http://image.honeypps.com/images/papers/2017/148.png)](http://image.honeypps.com/images/papers/2017/148.png)

可以观察No.122, No.123, No.124，这些对于上面的示例代码。

首先是client端发送Get请求，然后broker响应请求回传消息，最后client端发送Ack. 可以看到有别于push模式，broker端不会在client端没有请求的情况下来回传消息。