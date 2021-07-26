## RabbitMQ的三种交换机（exchange）



### 1. Direct Exchange – 处理路由键。

需要将一个队列绑定到交换机上，要求该消息与一个特定的路由键完全匹配。这是一个完整的匹配。如果一个队列绑定到该交换机上要求路由键 “dog”，则只有被标记为“dog”的消息才被转发，不会转发dog.puppy，也不会转发dog.guard，只会转发dog。

​    ![0](D:\学习\面试资料\java-interview\rabbitmq\images\1343) 

Java代码  

​    ![0](https://note.youdao.com/yws/public/resource/f04ff27f0e2268e4a1b0ffdd279c1947/xmlnote/6435B963C1D241BCB966351EBE36FA92/1339) 

~~~java
 Channel channel = connection.createChannel(); 
 channel.exchangeDeclare("exchangeName", "direct"); //direct fanout topic 
 channel.queueDeclare("queueName"); 
 channel.queueBind("queueName", "exchangeName", "routingKey"); 
  
 byte[] messageBodyBytes = "hello world".getBytes(); 
 //需要绑定路由键 
 channel.basicPublish("exchangeName", "routingKey",  MessageProperties.PERSISTENT_TEXT_PLAIN, messageBodyBytes); 
~~~





### 2. Fanout Exchange – 不处理路由键。

你只需要简单的将队列绑定到交换机上。一**个发送到交换机的消息都会被转发到与该交换机绑定的所有队列上**。很像子网广播，每台子网内的主机都获得了一份复制的消息。Fanout交换机转发消息是最快的。

​    ![0](D:\学习\面试资料\java-interview\rabbitmq\images\1341) 

Java代码：

~~~java
 Channel channel = connection.createChannel(); 
 channel.exchangeDeclare("exchangeName", "fanout"); //direct fanout topic 
 channel.queueDeclare("queueName"); 
 channel.queueBind("queueName", "exchangeName", "routingKey"); 
  
 channel.queueDeclare("queueName1"); 
 channel.queueBind("queueName1", "exchangeName", "routingKey1"); 
  
 byte[] messageBodyBytes = "hello world".getBytes(); 
 //路由键需要设置为空 
 channel.basicPublish("exchangeName", "", MessageProperties.PERSISTENT_TEXT_PLAIN, messageBodyBytes); 
~~~



### 3.**Topic Exchange** – 将路由键和某模式进行匹配。

此时队列需要绑定要一个模式上。符号“#”匹配一个或多个词，符号“*”匹配不多不少一个词。因此“audit.#”能够匹配 到“audit.irs.corporate”，但是“audit.*” 只会匹配到“audit.irs”。我在RedHat的朋友做了一张不错的图，来表明topic交换机是如何工作的：

​    ![0](D:\学习\面试资料\java-interview\rabbitmq\images\1342) 

Java代码  

~~~java
 Channel channel = connection.createChannel(); 
 channel.exchangeDeclare("exchangeName", "topic"); //direct fanout topic 
 channel.queueDeclare("queueName"); 
 channel.queueBind("queueName", "exchangeName", "routingKey.*"); 
 
 byte[] messageBodyBytes = "hello world".getBytes(); 
 channel.basicPublish("exchangeName", "routingKey.one",  MessageProperties.PERSISTENT_TEXT_PLAIN, messageBodyBytes);  
~~~

