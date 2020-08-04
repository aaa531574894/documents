### 一、RabbitMQ工作模型

<img src="https://upload-images.jianshu.io/upload_images/17039633-b0adf1dfade2f122.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="rabbit架构" style="zoom:50%;" />



### 二、amqp协议 

[amqp 协议](https://www.rabbitmq.com/tutorials/amqp-concepts.html)，协议中规定了一些组件与交换模式。

##### vitural host

broker为了提供多个相互隔离的运行环境，提出了vituralhost的概念，可以在管理界面或者命令行中新建，同样的使用vh需要给用户赋权；用户使用时也需要在代码中显示声明。

##### exchange

可以理解为消息的交换机，有以下属性

- Name

- Durability (exchanges survive broker restart)

- Auto-delete (exchange is deleted when last queue is unbound from it)

- Arguments (optional, used by plugins and broker-specific features)

broker会为默认创建一个名字为空“” 的默认交换机

##### queue  

消息队列，具有如下属性：

* Name
* Durable (the queue will survive a broker restart)
* Exclusive (used by only one connection and the queue will be deleted when that connection closes)
* Auto-delete (queue that has had at least one consumer is deleted when last consumer unsubscribes)
* Arguments (optional; used by plugins and broker-specific features such as message TTL, queue length limit, etc)

##### dead letter 死信 

当一个消息发生以下三种情况时，可以被扔到死信队列中：

> * The message is negatively acknowledged by a consumer using basic.reject or basic.nack with requeue parameter set to false.
> * The message expires due to per-message TTL; 
> * The message is dropped because its queue exceeded a length limit

在生产者发布消息时，可以为消息设定x-dead-letter-exchange 参数，指定一个exchange。这个exchange和正常的exchange完全一样，只不过你特意创建了一个exchange用来接收死信，所以叫做死信exchange。

如果在发布消息时未对消息设置死信routingkey，消息被扔到死信exchange时，会使用消息原本的routingkey；

也可以通过为消息设置死信routingkey，通过x-dead-letter-routing-key参数实现。



  ##### bindings 

用来绑定队列与交换机的规则，告诉指定交互机应该讲消息发送给哪些队列。

*如果消息不能路由到任何队列（例如，交换机上没有声明任何与队列相关的绑定），则消息将被丢弃或返回给发布者，具体取决于发布者设置的消息属性。*

##### message

是指消息队列中的消息实体，一个消息具有以下属性：

- Content type
- Content encoding
- Routing key
- Delivery mode (是否持久化至磁盘)  发布非持久化消息至 持久化的队列或交换机中，重启后消息也会丢失。
- Message priority
- Message publishing timestamp
- Expiration period
- Publisher application id

##### consumer

消费者获取消息的方式有两种：

* 订阅队列，等待server推送。（推荐）
* 主动从队列中pull消息（别用）

##### massage acknowledgement

消费者消费消息时，应用可能会发生异常，网络也可能会有异常，那么broker应该合适从队列中删除消息呢？amqp中提供了两种确认模式：

- **自动确认模式**：broker将消息发送到应用程序后，基于tcp连接实现，确保消息发送后自动删除。
- **手动确认模式**：由应用程序自己决定何时通知broker，可能是在收到消息之后；也可能是在消息被成功处理之后；在这种模式下，如果消费者在收到消息后，尚未确认的情况下gg；broker会把消息再交给另一个消费者。
  * ack   告诉broker，消息我收到了，也成功处理了，你可以删除了。
  * nack   告诉broker，消息我收到了，处理没成功，需要删除消息或者重新入队给其他消费者处理
  * reject  同上一个方法，但是reject只能拒绝一条消息，nack可以拒绝多条消息。

##### message reject

当消费者消费消息时，对消息的处理可能不成功。应用程序可以通过拒绝消息来告诉broker消息处理失败了。拒绝消息时，应用程序可以告诉broker，是重新将消息入队，还是说将消息丢弃掉。如果队列只有一个消费者在使用，请在编程时不要使broker陷入死循环，一遍又一遍的对同一条消息出队，入队。

##### message prefetching （Qos）

在同一个队列被对个消费者同时消费时，可以设置prefetching值，设置每个消费者



##### 可靠消息投递机制

从生产者角度看：

>1. 标准的amqp协议中，要保证消息投递到了exchange中，需要开启事务，但是这样降级了250倍左右的吞吐量；所以rabbitmq引入了另一种机制confirm机制，[参考](https://blog.csdn.net/eluanshi12/article/details/88959856)
>2. 消息投递时的确认机制：粗略来讲就是生产者如果消息投递成功，broker会返回确认消息，类似tcp。

从消费者角度看看:

> 1. 一个消费者应用在完成对消息的所有操作之前（操作数据库等等。），不应该向broker返回ack确认；一旦ack确认由消费者返回给broker，消息就会被彻底删除。
> 2. 如果有网络波动，可能会存在消息重复投递的情况，消费者应该被设计为幂等性的，而不是重复执行任务。
> 3. 如果消费者无法消费消息，可以 reject 或 nack，或扔出dead-letter队列中。
> 4. 如果消费者消费过程中出了异常，重新将消息扔回了队列，消息再次投递时会带上 **redelivered**  标识。

从消息broker高可用角度看:

> 1. rabbitmq支持消息持久化，以应对服务重启，故障等极端情况。消息是否持久化是在生产者发送消息时设置的。
> 2. rabbitmq支持 镜像集群（不是docker的那个镜像，理解为集群的节点互为镜像，保持数据一直），所有的exchange，queue等内容会在集群的所有节点中同步复制。



### 三、常见的几种exchange类型

#### 1.direct change

首先，broker默认提供了一个default exchange，没有名称，即""。direct change 与workQueue模式都是使用的defalut exchange。

可以理解为点对点消息，消息发布与消费不经过exchanger；生产者直接将消息通过defaultExchanger，发布至指定队列；消费者直接从队列中读取消息。

<img src="./src/main/resources/images/direct_mode.jpg" alt="direct mode" style="zoom: 33%;" />

在这种模式下，exchange参数不填或填”“，使用默认的defalt exchanger.

#### 2.workQueue 

direct模式的升级版本，多个消费着从队列中读，有公平分发，多劳多得两种模式，要看消费端的具体实现。

<img src="./src/main/resources/images/work_queue.jpg" alt="work queue" style="zoom: 33%;" />

在这种模式下，exchange参数不填或填”“，使用默认的exchanger。

**注：**

> 一个队列多个消费者模式下，队列无法保证消息是完全按顺序执行的。

#### 3.fanout  广播

<img src="./src/main/resources/images/fanout_mode.jpg" alt="work queue" style="zoom: 50%;" />



在这种模式下，routingKey不填，没有意思，主要使用exchange。 



#### 4.routing   发布订阅/静态路由

 

<img src="./src/main/resources/images/routing_mode.jpg" alt="work queue" style="zoom: 50%;" />

#### 5.topic  订阅模式/动态路由

<img src="./src/main/resources/images/topic_mode.jpg" alt="work queue" style="zoom: 50%;" />

> *****代表匹配一个单词，如  *.log，可以匹配  order.log、product.log等routingkey；但history.order.log，log 这种非两个单词的routingkey是无法匹配的。
>
> **#**代表0到多个单词，如 #.log，可以匹配任意以log结尾的routingkey，例如：a.b.log   a.log   log；
>
> **注**：多个单词之间必须以 .  分割
>
> ***特殊的***：当exchange配合 routingkey = #号时，此时topic的模式等同于fanout模式；routingkey=中不包含* 或 #，即全量匹配时，topic模式等于routing模式

#### 6.header 

topic交换的升级版，不使用routingkey了，而是使用exchange+ headerMap的方式进行路由交换。通过binding声明queue与exchange的绑定关系，exchange将消息的header中满足条件的消息转发到对应的queue中。

#### 7.rpc模式

<img src="./src/main/resources/images/rpc.jpg" alt="work queue" style="zoom: 50%;" />

客户端发送请求时需要在消息中带两个参数：

> replyTo      ： 返回至哪个队列
>
> correlationId    唯一请求id，

#### 8.几种模式的对比

|         比较项         | direct change | workQueue | fanout | routing | topic  | header |
| :--------------------: | :-----------: | :-------: | :----: | :-----: | :----: | ------ |
|      exchange类型      |     默认      |   默认    | 自定义 | 自定义  | 自定义 | 自定义 |
| 是否支持routingkey匹配 |       √       |     √     |   ×    |    √    |   √    | ×      |
|   是否支持表达式匹配   |       ×       |     ×     |   ×    |    ×    |   √    | ×      |
| 是否支持消息header匹配 |       ×       |     ×     |   ×    |    ×    |   ×    | √      |



### 四、应用场景

#### 1、异步

* 注册机制，注册后发短信与邮件可以通过异步队列通知的方式。
* 

#### 2、应用解耦

#### 3、流量削峰







***


### 五、springboot中使用RabbitMQ

#### 1.引入maven依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
 这个starter依赖包含spring-amqp与spring-rabbit两个模块





#### 2.配置文件

```properties
spring.rabbitmq.host=192.168.85.4
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=secret
//也可以这样配置,使用的是amqp协议中的443端口
//spring.rabbitmq.addresses=amqp://admin:secret@localhost

//失败重试策略,默认是关闭的
spring.rabbitmq.template.retry.enabled=true
spring.rabbitmq.template.retry.initial-interval=2s

```
#### 3.**RabbitTemplate** or **AmqpTemplate**

amqpTemplate是基于amqp协议的，同样的还有amqpAdmin；虽然spring都会为我们自动装配这些对象；但是推荐基于amqp进行注入；这样做可以与amqp解耦，后续如果换掉AMQP底层实现，就不需重构代码了。

#### 4.组件抽象

spring amqp在*org.springframework.amqp.core*包中为我们提供了对amqp协议中各组件与对象的抽象。

* Message MessageBuilder

* MessageProperties MessagePropertiesBuilder

  ```java
  MessageProperties props = MessagePropertiesBuilder.newInstance()
      .setContentType(MessageProperties.CONTENT_TYPE_TEXT_PLAIN)
      .setMessageId("123")
      .setHeader("bar", "baz")
      .build();
  Message message = MessageBuilder.withBody("foo".getBytes())
      .andProperties(props)
      .build();
  ```

  

* Exchange  spring将其抽象为接口，具体有这几种实现类：

  > * DirectiveExchange
  > * FanoutExchange
  > * TopicExchange
  > * HeadersExchange

* Queue

  ```java
  public Queue(String name, boolean durable, boolean exclusive, boolean autoDelete,
  			@Nullable Map<String, Object> arguments)
  ```

* Binding    BindingBuilder  用来告诉broker，queue与exchange的绑定关系，即何时exchange应该把消息转发到这个队列中。

  ```java
  //DirectExchange  
  new Binding(someQueue, someDirectExchange, "foo.bar");
  //TopicExchange 
  new Binding(someQueue, someTopicExchange, "foo.*");
  //FanoutExchange 
  new Binding(someQueue, someFanoutExchange);
  
  Binding b = BindingBuilder.bind(someQueue).to(someTopicExchange).with("foo.*");
  ```

* Container 

  The container is an example of a “lifecycle” component. It provides methods for starting and stopping. When configuring the container, you essentially bridge the gap between an AMQP Queue and the MessageListener instance. 

  





#### 5.监听队列

介绍一下以注解方式，如何为方法声明需要监听的queue。

```java
@Component
@Slf4j        // 消费者1
public static class Comsumer1 {
    
    @RabbitListener(
        bindings = @QueueBinding(    //绑定队列
            value = @Queue,          //创建临时队列
            exchange = @Exchange(       //指定交换机,与交换模式
                value = RabbitConfig.FAN_OUT_MODE,
                type = "fanout")))
    public void recv(String str) {
        log.info("cosumer 1 ----" + str);
    }
}

@RabbitListner 要配置@EnableRabbit注解使用。
```



#### 注：

> * springboot中，exchange、queue这两个组件都只能是消费者声明创建，即：生产者可以随便向任何queue和exchange中发消息，但是如果没有消费者主动创建对应的组件，消息会被server忽略掉，管理界面也看不到任何组件的信息。
> * springboot 2.0开始，




****
### 六、RabbitMQ安装过程(基于centos7) [参考](https://www.cnblogs.com/fengyumeng/p/11133924.html)
> 此次安装的版本为 rabbitMQ-3.8.5; erlang 23.0
RabbitMq基于erlang开发，所以需要先安装erlang环境。[erlang官网](https://www.erlang.org/downloads)  

1. 安装erlang
```
wget http://erlang.org/download/otp_src_23.0.tar.gz
tar -xf otp_src_23.0.tar.gz
cd otp_src_23.0
./configure --prefix=/home/liuyf/rabbitmq/erlang 
make install
```
注:如果安装过程中遇到报错如下:可以忽略  
![报错](https://img2018.cnblogs.com/blog/1181622/201907/1181622-20190704173838160-1881201010.png)

2. 配置erlang环境变量
```
vi ~/.bashrc

追加:
ERLANG_HOME=/home/liuyf/rabbitmq/erlang
export PATH="$PATH:$ERLANG_HOME/bin"

source ~/.bashrc
```
3. 安装rabbitmq
```
//进入https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.8.5 找到安装包

xz -d rabbitmq-server-generic-unix-3.8.5.tar.xz   //解压  没有xz的话先安装 yum install -y xz
tar -xf rabbitmq-server-generic-unix-3.8.5.tar  //需要解压两次


//配置环境变量
vi ~/.bashrc
//追加
Rabbit_Home=/home/liuyf/rabbitmq/rabbitmq_server-3.8.5
export PATH="$PATH:$Rabbit_Home/sbin"

source ~/.bashrc

```
4. RabbitMQ相关指令
```
//启动
rabbitmq-server -detached
//停止
rabbitmqctl stop
//状态
rabbitmqctl status
//开启web插件
rabbitmq-plugins enable rabbitmq_management
//web地址默认为：http://192.168.85.4:15672/     默认账号密码：guest guest（这个账号只允许本机访问）

//增加用户
rabbitmqctl add_user liuyf 123456
//配置权限
rabbitmqctl set_permissions -p "/" liuyf ".*" ".*" ".*"
rabbitmqctl set_user_tags liuyf administrator

```





### 附：消息队列的消费者如何实现幂等性

##### 基于redis：

首先要为消息指定唯一id

```java
//客户端命令
SET 1893505609317740 1466849127 EX 300 NX
//java 客户端 lettuce
Boolean setIfAbsent(K key, V value, Duration timeout)
```

##### 基于数据库：

类似。。