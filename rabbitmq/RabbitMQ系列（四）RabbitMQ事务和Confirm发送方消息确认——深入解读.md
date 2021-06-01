# [RabbitMQ系列（四）RabbitMQ事务和Confirm发送方消息确认——深入解读](https://www.cnblogs.com/vipstone/p/9350075.html)

RabbitMQ事务和Confirm发送方消息确认——深入解读

## RabbitMQ系列文章

1. [RabbitMQ在Ubuntu上的环境搭建](https://www.cnblogs.com/vipstone/p/9184314.html)
2. [深入了解RabbitMQ工作原理及简单使用](https://www.cnblogs.com/vipstone/p/9275256.html)
3. [RabbitMQ交换器Exchange介绍与实践](https://www.cnblogs.com/vipstone/p/9295625.html)
4. [RabbitMQ事务和Confirm发送方消息确认——深入解读](https://www.cnblogs.com/vipstone/p/9350075.html)
5. [使用Docker部署RabbitMQ集群](https://www.cnblogs.com/vipstone/p/9362388.html)
6. [你不知道的RabbitMQ集群架构全解](https://www.cnblogs.com/vipstone/p/9368106.html)

## 引言

根据前面的知识（[深入了解RabbitMQ工作原理及简单使用](https://www.cnblogs.com/vipstone/p/9275256.html)、[Rabbit的几种工作模式介绍与实践](https://www.cnblogs.com/vipstone/p/9295625.html)）我们知道，如果要保证消息的可靠性，需要对消息进行持久化处理，然而消息持久化除了需要代码的设置之外，还有一个重要步骤是至关重要的，那就是保证你的消息顺利进入Broker（代理服务器），如图所示：

![img](http://icdn.apigo.cn/blog/rabbitmq-flow3.png)

正常情况下，如果消息经过交换器进入队列就可以完成消息的持久化，但如果消息在没有到达broker之前出现意外，那就造成消息丢失，有没有办法可以解决这个问题？

RabbitMQ有两种方式来解决这个问题：

1. 通过AMQP提供的事务机制实现；
2. 使用发送者确认模式实现；

## 一、事务使用

事务的实现主要是对信道（Channel）的设置，主要的方法有三个：

1. channel.txSelect()声明启动事务模式；
2. channel.txComment()提交事务；
3. channel.txRollback()回滚事务；

从上面的可以看出事务都是以tx开头的，tx应该是transaction extend（事务扩展模块）的缩写，如果有准确的解释欢迎在博客下留言。

我们来看具体的代码实现：

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);	
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(_queueName, true, false, false, null);
String message = String.format("时间 => %s", new Date().getTime());
try {
	channel.txSelect(); // 声明事务
	// 发送消息
	channel.basicPublish("", _queueName, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes("UTF-8"));
	channel.txCommit(); // 提交事务
} catch (Exception e) {
	channel.txRollback();
} finally {
	channel.close();
	conn.close();
}
```

注意：用户需把config.xx配置成自己Rabbit的信息。

从上面的代码我们可以看出，在发送消息之前的代码和之前介绍的都是一样的，只是在发送消息之前，需要声明channel为事务模式，提交或者回滚事务即可。

了解了事务的实现之后，那么事务究竟是怎么执行的，让我们来使用wireshark抓个包看看，如图所示：

![img](D:\学习\面试资料\java-interview\rabbitmq\images\rabbitmq-trsaction-wr.png)

输入ip.addr==rabbitip && amqp查看客户端和rabbit之间的通讯，可以看到交互流程：

- 客户端发送给服务器Tx.Select(开启事务模式)
- 服务器端返回Tx.Select-Ok（开启事务模式ok）
- 推送消息
- 客户端发送给事务提交Tx.Commit
- 服务器端返回Tx.Commit-Ok

以上就完成了事务的交互流程，如果其中任意一个环节出现问题，就会抛出IoException移除，这样用户就可以拦截异常进行事务回滚，或决定要不要重复消息。

那么，既然已经有事务了，没什么还要使用发送方确认模式呢，原因是因为事务的性能是非常差的。**事务性能测试**：

事务模式，结果如下：

- 事务模式，发送1w条数据，执行花费时间：14197s
- 事务模式，发送1w条数据，执行花费时间：13597s
- 事务模式，发送1w条数据，执行花费时间：14216s

非事务模式，结果如下：

- 非事务模式，发送1w条数据，执行花费时间：101s
- 非事务模式，发送1w条数据，执行花费时间：77s
- 非事务模式，发送1w条数据，执行花费时间：106s

从上面可以看出，非事务模式的性能是事务模式的性能高149倍，我的电脑测试是这样的结果，不同的电脑配置略有差异，但结论是一样的，事务模式的性能要差很多，那有没有既能保证消息的可靠性又能兼顾性能的解决方案呢？那就是接下来要讲的Confirm发送方确认模式。

### 扩展知识

我们知道，消费者可以使用消息自动或手动发送来确认消费消息，那如果我们在消费者模式中使用事务（当然如果使用了手动确认消息，完全用不到事务的），会发生什么呢？

**消费者模式使用事务**

假设消费者模式中使用了事务，并且在消息确认之后进行了事务回滚，那么RabbitMQ会产生什么样的变化？

结果分为两种情况：

1. autoAck=false手动应对的时候是支持事务的，也就是说即使你已经手动确认了消息已经收到了，但在确认消息会等事务的返回解决之后，在做决定是确认消息还是重新放回队列，如果你手动确认现在之后，又回滚了事务，那么已事务回滚为主，此条消息会重新放回队列；
2. autoAck=true如果自定确认为true的情况是不支持事务的，也就是说你即使在收到消息之后在回滚事务也是于事无补的，队列已经把消息移除了；

## 二、Confirm发送方确认模式

Confirm发送方确认模式使用和事务类似，也是通过设置Channel进行发送方确认的。

**Confirm的三种实现方式：**

方式一：channel.waitForConfirms()普通发送方确认模式；

方式二：channel.waitForConfirmsOrDie()批量确认模式；

方式三：channel.addConfirmListener()异步监听发送方确认模式；

### 方式一：普通Confirm模式

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
String message = String.format("时间 => %s", new Date().getTime());
channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
if (channel.waitForConfirms()) {
	System.out.println("消息发送成功" );
}
```

看代码可以知道，我们只需要在推送消息之前，channel.confirmSelect()声明开启发送方确认模式，再使用channel.waitForConfirms()等待消息被服务器确认即可。

### 方式二：批量Confirm模式

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
for (int i = 0; i < 10; i++) {
	String message = String.format("时间 => %s", new Date().getTime());
	channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
}
channel.waitForConfirmsOrDie(); //直到所有信息都发布，只要有一个未确认就会IOException
System.out.println("全部执行完成");
```

以上代码可以看出来channel.waitForConfirmsOrDie()，使用同步方式等所有的消息发送之后才会执行后面代码，只要有一个消息未被确认就会抛出IOException异常。

### 方式三：异步Confirm模式

```java
// 创建连接
ConnectionFactory factory = new ConnectionFactory();
factory.setUsername(config.UserName);
factory.setPassword(config.Password);
factory.setVirtualHost(config.VHost);
factory.setHost(config.Host);
factory.setPort(config.Port);
Connection conn = factory.newConnection();
// 创建信道
Channel channel = conn.createChannel();
// 声明队列
channel.queueDeclare(config.QueueName, false, false, false, null);
// 开启发送方确认模式
channel.confirmSelect();
for (int i = 0; i < 10; i++) {
	String message = String.format("时间 => %s", new Date().getTime());
	channel.basicPublish("", config.QueueName, null, message.getBytes("UTF-8"));
}
//异步监听确认和未确认的消息
channel.addConfirmListener(new ConfirmListener() {
	@Override
	public void handleNack(long deliveryTag, boolean multiple) throws IOException {
		System.out.println("未确认消息，标识：" + deliveryTag);
	}
	@Override
	public void handleAck(long deliveryTag, boolean multiple) throws IOException {
		System.out.println(String.format("已确认消息，标识：%d，多个消息：%b", deliveryTag, multiple));
	}
});
```

异步模式的优点，就是执行效率高，不需要等待消息执行完，只需要监听消息即可，以上异步返回的信息如下：

![img](D:\学习\面试资料\java-interview\rabbitmq\images\rabbitmq-confirm-async-result.png)

可以看出，代码是异步执行的，消息确认有可能是批量确认的，是否批量确认在于返回的multiple的参数，此参数为bool值，如果true表示批量执行了deliveryTag这个值以前的所有消息，如果为false的话表示单条确认。

**Confirm性能测试**

测试前提：与事务一样，我们发送1w条消息。

方式一：Confirm普通模式

- 执行花费时间：2253s
- 执行花费时间：2018s
- 执行花费时间：2043s

方式二：Confirm批量模式

- 执行花费时间：1576s
- 执行花费时间：1400s
- 执行花费时间：1374s

方式三：Confirm异步监听方式

- 执行花费时间：1498s
- 执行花费时间：1368s
- 执行花费时间：1363s

### 总结

综合总体测试情况来看：Confirm批量确定和Confirm异步模式性能相差不大，Confirm模式要比事务快10倍左右。