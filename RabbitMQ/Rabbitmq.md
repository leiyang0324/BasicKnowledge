# 简单队列

### 定义一个获取链接的工具类

![](J:\找工作\RabbitMQ\简单队列\获取MQ连接.png)

### 编写一个消息生产者

![](J:\找工作\RabbitMQ\简单队列\生产者发送消息.png)

### 编写一个消息消费者

过期的方法

![](J:\找工作\RabbitMQ\简单队列\消费者消费一个消息.png)

新的方法

![](J:\找工作\RabbitMQ\简单队列\消费者消费消息.png)

# 工作队列

​	1、轮询模式(Round-robin)

​	自动应答打开就行，就和简单队列没什么区别，最大的区别就是添加了一个消费者

​	2、公平分发(fair-dipatch)

​		在consumer和produce程序中都需要设置channel.basicQos(1),保证一次只发一个消息

​		在**channel.basicConsumer(QUEUE_NAME,autoack,consumer1)**关闭自动应答(antoack = false),在消费者程序中handledelvery函数中的finally块里面需要进行手动应答，channel.basicack(envelop.getDeliberyTag(),false);

手动应答需要关闭自动确认模式，这样就解决了当消费者刚拿到消息还没来得及处理，就出现问题，此时如果自动应答开启的话，只要消息队列将这个消息发送出来了就会立刻把这个消息删除，但是关闭了自动应答，开启了手动应答，只要消费者没有在自己的**handledelvery**函数中进行**channel.basicack(envelop.getDeliberyTag(),false)**消息队列就不会删除这一条消息。这一点可以保证及时消费者挂掉了消息也不会丢失。

但是另外一个问题又出现了，那就是如果消息队列挂掉了怎么办？

那就是持久化技术

# 

# 订阅模式

之前都是一个消息只能被一个消费者消费，那么一个消息需要被多个消费者消费该怎么办？

## 解读

一个生产者，多个消费者

每一个消费者都有自己的一个队列

生产者不直接将消息发送到队列，而是发送到交换机

每个队列都要绑定到交换机

生产者发送到消息经过交换机，到达队列，实现一个消息被多个消费者获取的目的

## fanout分发模式

在生产者端：

创建exchange:channel.exchangeDeclare(EXCHANGE_NAME,"fanout")

发送消息：channel.basicPublish(EXCHANGE_NAME,"".null,message.getBytes());,这里和以前没有exchange的不一样，以前是channel.basicPublish("",QUEUE_NAME.null,message.getBytes())

在消费者端：

queueDeclare一个队列，由于可能有多个消费者，每一个消费者都会有一个队列，所以将这些队列bind到ecxchange上面（channel.bind(QUEUE_NAME,EXCHANGE_NAME,"")）这里最后一个参数是为后面的路由模式，主题模式留的

channel.exchangeDeclare(EXCHANGE_NAME,"fanolut")

在生产者里面声明交换机，在每一个消费者程序里面声明一个队列并且将队列绑定到交换机，这样只要是绑定到交换机的消费者程序都可以接收到发送者发送来的消息。同样也是需要设置channel.basicQos(1),然后爱消费者里面手动应答，在监听代码中关闭自动应答。

## 路由模式（direct模式）

在fanout模式中，只要队列绑定到了exchanger上那么这个队列就会收到这个消息，但是我们需要绑定到这个交换机上的队列有的能收到有的不能受到，这个时候就需要路由键，我们把某一些消息通过路由键发送到交换机，然后由交换机决定将这些消息通过路由键推送到那些使用了对应的路由键绑定了交换机的队列上

channel.exchangeDeclare(EXCHANGE_NAME,"direct")

发送者

channel.basicPublish(EXCHANGE_NAME,"DELETE",null,message1.getBytes());

channel.basicPublish(EXCHANGE_NAME,"UPDATE",null,message2.getBytes());

channel.basicPublish(EXCHANGE_NAME,"ADD",null,message3.getBytes());

这里的意思就是每一条消息都绑定了一个路由键，只有在消费者端进行队列和exchange绑定的时候使用了该路由键的队列（也就是消费者）才能收到这条路由消息。

消费者1

channel.queueBind(QUEUE_NAME1, EXCHANGE_NAME, "DELETE");

channel.queueBind(QUEUE_NAME1, EXCHANGE_NAME, "UPDATE");

也就是说消费者1只能收到message1和message2

消费者2

channel.queueBind(QUEUE_NAME2, EXCHANGE_NAME, "DELETE");

channel.queueBind(QUEUE_NAME2, EXCHANGE_NAME, "UPDATE");

channel.queueBind(QUEUE_NAME2， EXCHANGE_NAME, "ADD");

消费者2能收到message1和message2，message3

## 主题模型

路由模式的缺点就是如果某一个队列需要接受很多种不同路由键的消息，那么就需要我们一条一条的在这个队列与exchange之间bind这些路由键，这样就会很麻烦，所哟主题模型提供了一种类似于通配符的模式让我们避免了一条一条的操作

相当于将之前的路由模式可以进行多个路由匹配，发送给主题转发器的消息不能是任意设置的选择键，必须是用小数点隔开的一系列的标识符。**这些标识符可以是随意，但是通常跟消息的某些特性相关联。**

生产者：

channel.exchangeDeclare(EXCHANGE_NAME, "topic");

channel.basicPublish(EXCHANGE_NAME, "item.delete", null, message.getBytes());

channel.basicPublish(EXCHANGE_NAME, "item.update", null, message.getBytes());

channel.basicPublish(EXCHANGE_NAME, "item.add", null, message.getBytes());

消费者：





*（星号）可以代替一个任意标识符 ；#（井号）可以代替零个或多个标识符。

# 消息确认机制（事物+confirm）

**解决的是生产者发送消息之后有没有正确到达消息队列**

持久化解决rabbitmq服务器异常问题

生产者将消息发送之后，消息有没有正确到达rabbitmq？rabbitmq不会反馈任何消息给生产者，也就是默认情况下我们是不知道消息有没有正确到达。

消息到达服务器之前丢失，那么持久化也不能解决此类问题。

## 事务机制

AMQP协议层面提供的解决方案，耗时减少了系统的吞吐量。

try {
**channel.txSelect();**
channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
int result = 1 / 0;
**channel.txCommit();**
} catch (Exception e) {
**channel.txRollback();**
System.out.println("----msg rollabck ");
}finally{

System.out.println("---------send msg over:" + msg);
}
channel.close();
connection.close();
}
}

## Confirm 模式

生产者将信道设置成 confirm 模式，一旦信道进入 confirm 模式，所有在该信道上面发布的消息都会被指派一个唯一的 ID( 从 1 开始 ))，一旦消息被投递到所有匹配的队列之后 broker 就会发送一个确认给生产者（包含消息的唯一ID 这就使得生产者知道消息已经正确到达目的队列了，如果消息和队列是可持久化的，那么确认消息会将消息写入磁盘之后发出， broker 回传给生产者的确认消息中 deliver tag 域包含了确认消息的序列号。

confirm 模式最大的好处在于**他是异步的**，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用便可以通过回调方法来处理该确认消息，如果RabbitMQ 因为自身内部错误导致消息丢失，**就会发送一条 nack 消息**，生产者应用程序同样可以在回调方法中处理该 nack 消息。

### 开启confirm 模式的方法

已经在transaction 事务模式的 channel 是不能再设置成 confirm 模式的，即这两种模式是不能共存的。
生产者通过调用 channel 的 confirmSelect 方法将 channel 设置为 confirm 模式
核心 代码
//生产者通过调用channel的confirmSelect方法将channel设置为confirm模式
channel.confirmSelect();

### 编程模式

​      1  普通confirm模式：每发送一条消息后，调用**waitForConfirms**()方法，等待服务器端confirm。实际上是一种串行confirm了。 

2. 批量confirm模式：每发送一批消息后，调用waitForConfirms()方法，等待服务器端confirm。 

3. 异步confirm模式：提供一个回调方法，服务端confirm了一条或者多条消息后Client端会回调这个方法。

   

   

#### 在发送端

if(!channel.waitForConfirms()){
System.out.println("send message failed.");
}else{
System.out.println(" send messgae ok ...");
}

#### 批量

批量 confirm 模式稍微复杂一点，客户端程序需要定期（每隔多少秒）或者定量（达到多少条）或者两则结合起来
publish 消息，然后等待服务器端 confirm, 相比普通 confirm 模式，批量极大提升 confirm 效率，但是问题在于一旦
出现 confirm 返回 false 或者超时的情况时，客户端需要将这一批次的消息全部重发，这会带来明显的重复消息数
量，并且，当消息经常丢失时，批量 confirm 性能应该是不升反降的。

for (int i = 0; i < 10; i++) {
channel.basicPublish("", QUEUE_NAME, null,msg.getBytes());
}
if(!channel.waitForConfirms()){
System.out.println("send message failed.");
}else{

System.out.println(" send messgae ok ...");
}

#### 异步 confirm 模式