# 一、概述

## 1. MQ概述

### 1.1 概念

MQ（message queue），本质是个队列，符合FIFO先进先出原则，只不过队列中存放的内容是message而已；还是一种跨进程的通信机制，用于上下游传递消息。

### 1.2 作用

- 流量削峰

   MQ可以将系统的超量请求**暂存**其中，以便系统后期可以慢慢进行处理，从而避免了请求的丢失或系统被压垮。

- 应用解耦

  发送者和接收者之间通过消息中间件进行通信，不再直接依赖彼此。

- 异步处理

  将不需要立即响应的操作，异步发送给后台处理，提高主流程响应速度。

### 1.3 分类

| 队列       | 高吞吐 | 可靠性 | 易用性 | 支持持久化           | 事务支持         | 推荐使用场景           |
| ---------- | ------ | ------ | ------ | -------------------- | ---------------- | ---------------------- |
| Kafka      | ★★★★★  | ★★★★☆  | ★★☆☆☆  | 是                   | 否（支持幂等性） | 大数据日志、流式处理   |
| RabbitMQ   | ★★★☆☆  | ★★★★★  | ★★★★☆  | 是                   | 是               | 企业系统、微服务解耦   |
| RocketMQ   | ★★★★☆  | ★★★★★  | ★★★☆☆  | 是                   | 是               | 金融、电商交易系统     |
| ActiveMQ   | ★★☆☆☆  | ★★★★☆  | ★★★☆☆  | 是                   | 是               | 老牌Java系统           |
| Redis      | ★★★☆☆  | ★★★☆☆  | ★★★★★  | 否（可持久化但不强） | 否               | 小型异步任务、实时提醒 |
| NSQ        | ★★★★☆  | ★★★☆☆  | ★★★★☆  | 否                   | 否               | 日志、实时数据收集     |
| ZeroMQ     | ★★★★★  | ★★★☆☆  | ★★☆☆☆  | 否                   | 否               | 嵌入式通信、低延迟通信 |
| Celery     | ★★☆☆☆  | ★★★☆☆  | ★★★★★  | 依赖后端             | 部分支持         | Python分布式任务       |
| Amazon SQS | ★★★★☆  | ★★★★★  | ★★★★★  | 是                   | 部分支持         | 云原生、无需运维       |



## 2. RocketMQ概述

RocketMQ是阿里巴巴完全用Java语言编写的消息队列。于2016年捐赠给Apache基金。



# 二、快速入门

消息队列存储在RockerMQ的Broker服务端，然后消费者和生产者通过调用RockerMQ客户端来进行消息的消费和生产，但两个客户端并不直接交互，而是以Broker作为中间着进行交互。而Broker的启动依赖Nameserver（注册中心），需要在Nameserver中注册。

## 1. 启动Broker

```
docker run -d --name rmqbroker \
 -p 10911:10911 -p 10909:10909 \
 -e "NAMESRV_ADDR=host.docker.internal:9876" \
 apache/rocketmq:latest sh mqbroker \
 -n host.docker.internal:9876 -c /opt/rocketmq-4.9.0/conf/broker.conf
```

## 2. 编写客户端代码

引入依赖

```
<!-- https://mvnrepository.com/artifact/org.apache.rocketmq/rocketmq-client -->
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>5.3.1</version>
</dependency>
```



生产者代码

```java
DefaultMQProducer producer = new DefaultMQProducer("ProducerGroup");
producer.setNamesrvAddr("localhost:9876");
producer.start();

Message msg = new Message("TopicTest", "TagA", "Hello MQ".getBytes());
SendResult result = producer.send(msg);

System.out.println(result);
producer.shutdown();
```



消费者代码

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("ConsumerGroup");
consumer.setNamesrvAddr("localhost:9876");
consumer.subscribe("TopicTest", "*");

consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
    for (MessageExt msg : msgs) {
        System.out.println("Received: " + new String(msg.getBody()));
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});

consumer.start();
```



# 三、基本概念

## 1. 主题（Topic）

### 1.1 定义

主题是 Apache RocketMQ 中消息传输和存储的顶层容器，用于标识同一类业务逻辑的消息。 主题的作用主要如下：

- **定义数据的分类隔离：** 在 Apache RocketMQ 的方案设计中，建议将不同业务类型的数据拆分到不同的主题中管理，通过主题实现存储的隔离性和订阅隔离性。
- **定义数据的身份和权限：** Apache RocketMQ 的消息本身是匿名无身份的，同一分类的消息使用相同的主题来做身份识别和权限管理。

### 1.2 模型关系

在整个 Apache RocketMQ 的领域模型中，主题所处的流程和位置如下：

![rocketmq_xioans](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_xioans.png)

主题是 Apache RocketMQ 的顶层存储，所有消息资源的定义都在主题内部完成，但主题是一个逻辑概念，并不是实际的消息容器。

主题内部由多个队列组成，消息的存储和水平扩展能力最终是由队列实现的；并且针对主题的所有约束和属性设置，最终也是通过主题内部的队列来实现。

### 1.3 内部属性

**主题名称**

- 定义：主题的名称，用于标识主题，主题名称集群内全局唯一。
- 取值：由用户创建主题时定义。

**队列列表**

- 定义：队列作为主题的组成单元，是消息存储的实际容器，一个主题内包含一个或多个队列，消息实际存储在主题的各队列内。
- 取值：系统根据队列数量给主题分配队列，队列数量创建主题时定义。
- 约束：一个主题内至少包含一个队列。

**消息类型**

- 定义：主题所支持的消息类型。
- 取值：创建主题时选择消息类型。Apache RocketMQ 支持的主题类型如下：
  - Normal：普通消息，消息本身无特殊语义，消息之间也没有任何关联。
  - FIFO：顺序消息，Apache RocketMQ 通过消息分组MessageGroup标记一组特定消息的先后顺序，可以保证消息的投递顺序严格按照消息发送时的顺序。
  - Delay：定时/延时消息，通过指定延时时间控制消息生产后不要立即投递，而是在延时间隔后才对消费者可见。
  - Transaction：事务消息，Apache RocketMQ 支持分布式事务消息，支持应用数据库更新和消息调用的事务一致性保障。
- 约束：Apache RocketMQ 从5.0版本开始，支持强制校验消息类型，即每个主题只允许发送一种消息类型的消息，这样可以更好的运维和管理生产系统，避免混乱。为保证向下兼容4.x版本行为，强制校验功能默认开启。

### 1.4 行为约束

**消息类型强制校验**

Apache RocketMQ 5.x版本支持将消息类型拆分到主题中进行独立运维和处理，因此系统会对发送的消息类型和主题定的消息类型进行强制校验，若校验不通过，则消息发送请求会被拒绝，并返回类型不匹配异常。校验原则如下：

- 消息类型必须一致：发送的消息的类型，必须和目标主题定义的消息类型一致。
- 主题类型必须单一：每个主题只支持一种消息类型，不允许将多种类型的消息发送到同一个主题中。

> 为保证向下兼容4.x版本行为，上述强制校验功能默认开启。

**常见错误使用场景**

- 发送的消息类型不匹配。例如：创建主题时消息类型定义为顺序消息，发送消息时发送事务消息到该主题中，此时消息发送请求会被拒绝，并返回类型不匹配异常。
- 单一消息主题混用。例如：创建主题时消息类型定义为普通消息，发送消息时同时发送普通消息和顺序消息到该主题中，则顺序消息的发送请求会被拒绝，并返回类型不匹配异常。



## 2. 消息队列（MessageQueue）

### 2.1 定义

队列是 Apache RocketMQ 中消息存储和传输的实际容器，也是 Apache RocketMQ 消息的最小存储单元。 Apache RocketMQ 的所有主题都是由多个队列组成，以此实现队列数量的水平拆分和队列内部的流式存储。

队列的主要作用如下：

- 存储顺序性

  队列天然具备顺序性，即消息按照进入队列的顺序写入存储，同一队列间的消息天然存在顺序关系，队列头部为最早写入的消息，队列尾部为最新写入的消息。消息在队列中的位置和消息之间的顺序通过位点（Offset）进行标记管理。

- 流式操作语义

  Apache RocketMQ 基于队列的存储模型可确保消息从任意位点读取任意数量的消息，以此实现类似聚合读取、回溯读取等特性，这些特性是RabbitMQ、ActiveMQ等非队列存储模型不具备的。

### 2.2 模型关系

在整个 Apache RocketMQ 的领域模型中，队列所处的流程和位置如下：

![rocketmq_opsanc](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_opsanc.png)

Apache RocketMQ 默认提供消息可靠存储机制，**所有发送成功的消息都被持久化存储到队列中**，配合生产者和消费者客户端的调用可实现**至少投递一次的可靠性语义**。

Apache RocketMQ 队列模型和Kafka的分区（Partition）模型类似。在 Apache RocketMQ 消息收发模型中，队列属于主题的一部分，虽然所有的消息资源以主题粒度管理，但实际的操作实现是面向队列。例如，生产者指定某个主题，向主题内发送消息，但实际消息发送到该主题下的某个队列中。

Apache RocketMQ 中通过修改队列数量，以此实现横向的水平扩容和缩容。

### 2.3 内部属性

读写权限

- 定义：当前队列是否可以读写数据。
- 取值：由服务端定义，枚举值如下
  - 6：读写状态，当前队列允许读取消息和写入消息。
  - 4：只读状态，当前队列只允许读取消息，不允许写入消息。
  - 2：只写状态，当前队列只允许写入消息，不允许读取消息。
  - 0：不可读写状态，当前队列不允许读取消息和写入消息。

- 约束：队列的读写权限属于运维侧操作，不建议频繁修改。

### 2.4 行为约束

每个主题下会由一到多个队列来存储消息，每个主题对应的队列数与消息类型以及实例所处地域（Region）相关。



## 3. 消息（Message）

### 3.1 定义

消息是 Apache RocketMQ 中的**最小数据传输单元**。生产者将业务数据的负载和拓展属性包装成消息发送到 Apache RocketMQ 服务端，服务端按照相关语义将消息投递到消费端进行消费。

Apache RocketMQ 的消息模型具备如下特点：

- **消息不可变性**

  消息本质上是已经产生并确定的事件，一旦产生后，消息的内容不会发生改变。即使经过传输链路的控制也不会发生变化，消费端获取的消息都是只读消息视图。

- **消息持久化**

  Apache RocketMQ 会默认对消息进行持久化，即将接收到的消息存储到 Apache RocketMQ 服务端的存储文件中，保证消息的可回溯性和系统故障场景下的可恢复性。

### 3.2 模型关系

在整个 Apache RocketMQ 的领域模型中，消息所处的流程和位置如下：

![rocketmq_opsanc](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_opsanc.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 3.3 消息内部属性

**系统保留属性**

**主题名称**

- 定义：当前消息所属的主题的名称。集群内全局唯一。
- 取值：从客户端SDK接口获取。

**消息类型**

- 定义：当前消息的类型。
- 取值：从客户端SDK接口获取。Apache RocketMQ 支持的消息类型如下：
  - Normal：普通消息，消息本身无特殊语义，消息之间也没有任何关联。
  - FIFO：顺序消息，Apache RocketMQ 通过消息分组MessageGroup标记一组特定消息的先后顺序，可以保证消息的投递顺序严格按照消息发送时的顺序。
  - Delay：定时/延时消息，通过指定延时时间控制消息生产后不要立即投递，而是在延时间隔后才对消费者可见。
  - Transaction：事务消息，Apache RocketMQ 支持分布式事务消息，支持应用数据库更新和消息调用的事务一致性保障。

**消息队列**

- 定义：实际存储当前消息的队列。
- 取值：由服务端指定并填充。

**消息位点**

- 定义：当前消息存储在队列中的位置。
- 取值：由服务端指定并填充。取值范围：0~long.Max。

**消息ID**

- 定义：消息的唯一标识，集群内每条消息的ID全局唯一。
- 取值：生产者客户端系统自动生成。固定为数字和大写字母组成的32位字符串。

**索引Key列表（可选）**

- 定义：消息的索引键，可通过设置不同的Key区分消息和快速查找消息。
- 取值：由生产者客户端定义。

**过滤标签Tag（可选）**

- 定义：消息的过滤标签。消费者可通过Tag对消息进行过滤，仅接收指定标签的消息。
- 取值：由生产者客户端定义。
- 约束：一条消息仅支持设置一个标签。

**定时时间（可选）**

- 定义：定时场景下，消息触发延时投递的毫秒级时间戳。
- 取值：由消息生产者定义。
- 约束：最大可设置定时时长为40天。

**消息发送时间**

- 定义：消息发送时，生产者客户端系统的本地毫秒级时间戳。
- 取值：由生产者客户端系统填充。
- 说明：客户端系统时钟和服务端系统时钟可能存在偏差，消息发送时间是以客户端系统时钟为准。

**消息保存时间戳**

- 定义：消息在Apache RocketMQ 服务端完成存储时，服务端系统的本地毫秒级时间戳。 对于定时消息和事务消息，消息保存时间指的是消息生效对消费方可见的服务端系统时间。

- 取值：由服务端系统填充。
- 说明：客户端系统时钟和服务端系统时钟可能存在偏差，消息保留时间是以服务端系统时钟为准。

**消费重试次数**

- 定义：消息消费失败后，Apache RocketMQ 服务端重新投递的次数。每次重试后，重试次数加1。更多信息，请参见[消费重试](https://rocketmq.apache.org/zh/docs/featureBehavior/10consumerretrypolicy)。
- 取值：由服务端系统标记。首次消费，重试次数为0；消费失败首次重试时，重试次数为1。

**业务自定义属性**

- 定义：生产者可以自定义设置的扩展信息。
- 取值：由消息生产者自定义，按照字符串键值对设置。

**消息负载**

- 定义：业务消息的实际报文数据。
- 取值：由生产者负责序列化编码，按照二进制字节传输。

### 3.4 行为约束

消息大小不得超过其类型所对应的限制，否则消息会发送失败。

系统默认的消息最大限制如下：

- 普通和顺序消息：4 MB
- 事务和定时或延时消息：64 KB



## 4. 生产者（Producer）

### 4.1 定义

生产者是 Apache RocketMQ 系统中用来构建并传输消息到服务端的运行实体。

生产者通常被集成在业务系统中，将业务消息按照要求封装成 Apache RocketMQ 的消息（Message）并发送至服务端。

在消息生产者中，可以定义如下传输行为：

- 发送方式：生产者可通过API接口设置消息发送的方式。Apache RocketMQ 支持同步传输和异步传输。
- 批量发送：生产者可通过API接口设置消息批量传输的方式。例如，批量发送的消息条数或消息大小。
- 事务行为：Apache RocketMQ 支持事务消息，对于事务消息需要生产者配合进行事务检查等行为保障事务的最终一致性。

生产者和主题的关系为多对多关系，即同一个生产者可以向多个主题发送消息，对于平台类场景如果需要发送消息到多个主题，并不需要创建多个生产者；同一个主题也可以接收多个生产者的消息，以此可以实现生产者性能的水平扩展和容灾。 

![rocketmq_zajnbad](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_zajnbad.png)

### 4.2 模型关系

在 Apache RocketMQ 的领域模型中，生产者的位置和流程如下：

![rocketmq_vkdsno](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_vkdsno.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到主题的指定队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 4.3 内部属性

**客户端ID**

- 定义：生产者客户端的标识，用于区分不同的生产者。集群内全局唯一。
- 取值：客户端ID由Apache RocketMQ 的SDK自动生成，主要用于日志查看、问题定位等运维场景，不支持修改。

**通信参数**

- 接入点信息 **（必选）** ：连接服务端的接入地址，用于识别服务端集群。 接入点必须按格式配置，建议使用域名，避免使用IP地址，防止节点变更无法进行热点迁移。
- 身份认证信息 **（可选）** ：客户端用于身份验证的凭证信息。 仅在服务端开启身份识别和认证时需要传输。
- 请求超时时间 **（可选）** ：客户端网络请求调用的超时时间。

**预绑定主题列表**

- 定义：Apache RocketMQ 的生产者需要将消息发送到的目标主题列表，主要作用如下：

  - 事务消息 **（必须设置）** ：事务消息场景下，生产者在故障、重启恢复时，需要检查事务消息的主题中是否有未提交的事务消息。避免生产者发送新消息后，主题中的旧事务消息一直处于未提交状态，造成业务延迟。

  - 非事务消息 **（建议设置）** ：服务端会在生产者初始化时根据预绑定主题列表，检查目标主题的访问权限和合法性，而不需要等到应用启动后再检查。

    若未设置，或后续消息发送的目标主题动态变更， Apache RocketMQ 会对目标主题进行动态补充检验。

- 约束：对于事务消息，预绑定列表必须设置，且需要和事务检查器一起配合使用。

**事务检查器**

- 定义：Apache RocketMQ 的事务消息机制中，为保证异常场景下事务的最终一致性，生产者需要主动实现事务检查器的接口。
- 发送事务消息时，事务检查器必须设置，且需要和预绑定主题列表一起配合使用。

**发送重试策略**：

- 定义: 生产者在消息发送失败时的重试策略。



## 5. 消费者分组（Consumer Group）

### 5.1 定义

消费者分组是 Apache RocketMQ 系统中承载多个消费行为一致的消费者的负载均衡分组。

和消费者不同，消费者分组并不是运行实体，而是一个逻辑资源。在 Apache RocketMQ 中，通过消费者分组内初始化多个消费者实现消费性能的水平扩展以及高可用容灾。

在消费者分组中，统一定义以下消费行为，同一分组下的多个消费者将按照分组内统一的消费行为和负载均衡策略消费消息。

- 订阅关系：Apache RocketMQ 以消费者分组的粒度管理订阅关系，实现订阅关系的管理和追溯。
- 投递顺序性：Apache RocketMQ 的服务端将消息投递给消费者消费时，支持顺序投递和并发投递，投递方式在消费者分组中统一配置。
- 消费重试策略： 消费者消费消息失败时的重试策略，包括重试次数、死信队列设置等。
- 消费规则：一个消费者组只能订阅并消费一个Topic的消息，但是一个Topic能够被多个消费者组同时订阅消费。一个消费者组内的不同消费者不能消费同一个消息队列，但是不同消费者组的消费者可以消费同一个消息队列。

### 5.2 模型关系

在 Apache RocketMQ 的领域模型中，消费者分组的位置和流程如下：

![rocketmq_qldfsab](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_qldfsab.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到主题的指定队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 5.3 内部属性

**消费者分组名称**

- 定义：消费者分组的名称，用于区分不同的消费者分组。集群内全局唯一。
- 取值：消费者分组由用户设置并创建。

**投递顺序性**

- 定义：消费者消费消息时，Apache RocketMQ 向消费者客户端投递消息的顺序。

  根据不同的消费场景，Apache RocketMQ 提供顺序投递和并发投递两种方式。

- 取值：默认投递方式为并发投递。

**消费重试策略**

- 定义：消费者消费消息失败时，系统的重试策略。消费者消费消息失败时，系统会按照重试策略，将指定消息投递给消费者重新消费。
- 取值：重试策略包括：
  - 最大重试次数：表示消息可以重新被投递的最大次数，超过最大重试次数还没被成功消费，消息将被投递至死信队列或丢弃。
  - 重试间隔：Apache RocketMQ 服务端重新投递消息的间隔时间。
- 约束：重试间隔仅在PushConsumer消费类型下有效。

**订阅关系**

- 定义：当前消费者分组关联的订阅关系集合。包括消费者订阅的主题，以及消息的过滤规则等。订阅关系由消费者动态注册到消费者分组中，Apache RocketMQ 服务端会持久化订阅关系并匹配消息的消费进度。

### 5.4 行为约束

在 Apache RocketMQ 领域模型中，消费者的管理通过消费者分组实现，同一分组内的消费者共同分摊消息进行消费。因此，为了保证分组内消息的正常负载和消费，

Apache RocketMQ 要求同一分组下的所有消费者以下消费行为保持一致：

- **投递顺序**
- **消费重试策略**



## 6. 消费者（Consumer）

### 6.1 定义

消费者是 Apache RocketMQ 中用来接收并处理消息的运行实体。 消费者通常被集成在业务系统中，从 Apache RocketMQ 服务端获取消息，并将消息转化成业务可理解的信息，供业务逻辑处理。

在消息消费端，可以定义如下传输行为：

- 消费者身份：消费者必须关联一个指定的消费者分组，以获取分组内统一定义的行为配置和消费状态。
- 消费者类型：Apache RocketMQ 面向不同的开发场景提供了多样的消费者类型，包括PushConsumer类型、SimpleConsumer类型、PullConsumer类型（仅推荐流处理场景使用）等。
- 消费者本地运行配置：消费者根据不同的消费者类型，控制消费者客户端本地的运行配置。例如消费者客户端的线程数，消费并发度等，实现不同的传输效果。
- 消费规则：一个消费者组只能订阅一个Topic，因此一个消费者也只能消费一个Topic的消息，但一个消费者可以消费一个Topic中的多个消息队列，但一个消息队列只能被一个消费者消费。

### 6.2 模型关系

在 Apache RocketMQ 的领域模型中，消费者的位置和流程如下：

![rocketmq_lcnasdsa](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_lcnasdsa.png)

1. 消息由生产者初始化并发送到Apache RocketMQ 服务端。
2. 消息按照到达Apache RocketMQ 服务端的顺序存储到主题的指定队列中。
3. 消费者按照指定的订阅关系从Apache RocketMQ 服务端中获取消息并消费。

### 6.3 内部属性

**消费者分组名称**

- 定义：当前消费者关联的消费者分组名称，消费者必须关联到指定的消费者分组，通过消费者分组获取消费行为。
- 取值：消费者分组为Apache RocketMQ 的逻辑资源，需要您提前通过控制台或OpenAPI创建。

**客户端ID**

- 定义：消费者客户端的标识，用于区分不同的消费者。集群内全局唯一。
- 取值：客户端ID由Apache RocketMQ 的SDK自动生成，主要用于日志查看、问题定位等运维场景，不支持修改。

**通信参数**

- 接入点信息 **（必选）** ：连接服务端的接入地址，用于识别服务端集群。 接入点必须按格式配置，建议使用域名，避免使用IP地址，防止节点变更无法进行热点迁移。
- 身份认证信息 **（可选）** ：客户端用于身份验证的凭证信息。 仅在服务端开启身份识别和认证时需要传输。
- 请求超时时间 **（可选）** ：客户端网络请求调用的超时时间。

**预绑定订阅关系列表**

- 定义：指定消费者的订阅关系列表。 Apache RocketMQ 服务端可在消费者初始化阶段，根据预绑定的订阅关系列表对目标主题进行权限及合法性校验，无需等到应用启动后才能校验。

- 取值：建议在消费者初始化阶段明确订阅关系即要订阅的主题列表，若未设置，或订阅的主题动态变更，Apache RocketMQ 会对目标主题进行动态补充校验。

**消费监听器**

- 定义：Apache RocketMQ 服务端将消息推送给消费者后，消费者调用消息消费逻辑的监听器。
- 取值：由消费者客户端本地配置。
- 约束：使用PushConsumer类型的消费者消费消息时，消费者客户端必须设置消费监听器。

### 6.4 行为约束

在 Apache RocketMQ 领域模型中，消费者的管理通过消费者分组实现，同一分组内的消费者共同分摊消息进行消费。因此，为了保证分组内消息的正常负载和消费，

Apache RocketMQ 要求同一分组下的所有消费者以下消费行为保持一致：

- **投递顺序**
- **消费重试策略**



## 7. 消息标识

RocketMQ中每个消息拥有唯一的Messageld，且可以携带具有业务标识的Key，以方便对消息的查询。不过需要注意的是，MessageId有两个: 在生产者send0消息时会自动生成一个MessageId (msgId)， 当消息到达Broker后，Broker也会自动生成一个Messageld（offsetMsgId）。msgId、offsetMsgId与key都称为消息标识。

- msgId：由producer端生成， 其生成规则为：

  producerIp + 进程pid + MessageClientIDSetter类的ClassLoader的hashcode + 当前时间 + AutomicInteger自增计数器

- offsetMsgId：由broker端生成，其生成规则为：

  brokerIp + 物理分区的offset（Queue中的偏移量）

- key：由用户指定的业务相关的唯一标识



# 四、系统架构

![rocketmq_ocsnajoc](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_ocsnajoc.png)

## 1. 核心架构组件

RocketMQ 是一个分布式消息中间件，主要包含以下核心组件：

### 1.1 生产者（Producer）

负责发送消息到指定的 Topic。



### 1.2 消费者（Consumer）

负责订阅并消费指定 Topic 的消息。



### 1.3 代理服务器（Broker）

#### 1.3.1 功能介绍

Broker充当着消息中转角色，负责存储消息、转发消息。Broker在RocketMQ系统中负责接收并存储从生产者发送来的消息，同时为消费者的拉取请求作准备。Broker同时也存储着消息相关的元数据，包括**消费者组消费进度偏移offset**、**主题**、**队列**等。

> Kafka 0.8版本之后，offset是存放在Broker中的，之前版本是存放在Zookeeper中的。

#### 1.3.2 模块构成

Brocker Server的功能模块示意图：

![rocketmq_ocnajsnc](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_ocnajsnc.png)

- Remoting Module：整个Broker的实体，负责处理来自clients端的请求。而这个Broker实体则由以下模块构成。
- Client Manager：客户端管理器。负责接收、解析客户端(Producer/Consumer)请求， 管理客户端。例如，维护Consumer的Topic订阅信息
- Store Service：存储服务。提供方便简单的API接口，处理消息存储到物理硬盘和消息查询功能。
- HA Service：高可用服务。提供Master Broker和Slave Broker之间的数据同步功能。
- Index Service：索引服务。根据特定的Message key，对投递到Broker的消息进行索引服务，同时也提供根据Message Key对消息进行快速宣询的功能。

#### 1.3.3 集群部署

 为了增强Broker性能与吞吐量，Broker一般都是以集群形式出现的。各集群节点中可能存放着相同Topic的不同Queue。不过，这里有个问题，如果某Broker节点宕机，如何保证数据不丢失呢？其解决方案是，将每个Broker集群节点进行横向扩展，即将Broker节点再建为一个HA集群，解决单点问题。

Broker节点集群是一个主从集群，即集群中具有Master 与Slave两种角色。一个Master可以包含多个Slave，但一个Slave只能隶属于一个Master。Master与Slave的对应关系是通过指定相同的BrokerName、不同的Brokerld 来确定的。Brokerld为0表示Master，非0表示Slave。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer。

Master负责处理读写操作请求，而Slave负责对Master中的数据进行备份。当Master挂掉了，Slave则会自动切换为Master去工作。所以Broker集群是主备集群。



### 1.4 名称服务（NameServer）

#### 1.4.1 功能介绍

NameServer是一个Broker与Topic路由的注册中心，支持Broker的动态注册与发现。

RocketMQ的思想来自于Kafka，而Kafka是依赖了Zookeeper的。所以，在RocketMQ的早期版本，即在MetaQ v1.0与v2.0版本中，也是依赖于Zookeeper的。 从MetaQ v3.0,即RocketMQ开始去掉了Zookeeper依赖，使用了自己的NameServer.

主要包括两个功能:

- Broker管理：接受Broker集群的注册信息并且保存下来作为路由信息的基本数据；提供心跳检测机制，检查Broker是否还存活。
- 路由信息管理：每个NameServer中都保存着Broker集群的整个路由信息和用于客户端查询的队列信息。Producer和Conumser通过NameServer可以获取整个Broker集群的路由信息，从而进行消息的投递和消费。

#### 1.4.2 路由注册

NameServer通常也是以集群的方式部署，不过，NameServer是无状态的，即NameServer集群中的各个节点间是无差异的，**各节点间相互不进行信息通讯**。那各节点中的数据是如何进行数据同步的呢？在Broker节点启动时，轮训NameServer列表， 与每个NameServer节点建立长连接，发起注册请求。在NameServer内部维护着一个Broker列表， 用来动态存储Broker的信息。

> NameServer与其他像zk、Eureka、Nacos等注册中心不同。
>
> NameServer无状态方式的优缺点：
>
> **优点：**NameServer集群搭建简单。
>
> **缺点：**对于Broker，必须明确指出所有NameServer地址。否则未指出的将不会去注册。也正因为如此，NameServer并不能随便扩容。因为，若Broker不重新配置，新增的NameServer对于Broker来说是不可见的，其不会向这个NameServer进行注册。

Broker节点为了证明自己是活着的，为了维护与NameServer间的长连接，会将最新的信息以心跳包的方式上报给NameServer，每30秒发送一次心跳。 心跳包中包含Brokerld、Broker地址（IP+Port）、Broker名称、Broker所属集群名称等等。NameServer在接收到心跳包后， 会更新心跳时间戳，记录这个Broker的最新存活时间。

#### 1.4.3 路由剔除

由于Broker关机、宕机或网络抖动等原因，NameServer没有收到Broker的心跳，NameServer可能会将其从Broker列表中剔除。

NameServer中有一个定时任务，每隔10秒就会扫描一次Broker表，查看每一个Broker的最新心跳时间戳距离当前时间是否超过120秒，如果超过，则会判定Broker失效，然后将其从Broker列表中剔除。

> 扩展：对于RoketMQ日常运维工作，例如Broker升级，需要停掉Broker的工作，运维工程师需要怎么做？
>
> 运维工程师需要将Broker的读写权限禁掉。一旦client（Consumer或Producer）向broker发送请求，都会收到broker的NO_PERMISSION响应，然后client会进行其他Broker的重试。
>
> 当运维工程师观察到这个Broker没有流量后，再关闭它，实现Broker从NameServer的移除。

#### 1.4.4 路由发现

RocketMQ的路由发现采用的是Pull模型。当Topic路由信息出现变化时，NameServer不会主动推送给客户端，而是客户端定时拉取主题最新的路由。默认客户端每30秒会拉取一次最新的路由。

> 扩展：
>
> 1. Push模型：推送模型。实时性较好，是一个“订阅-发布”模型，需要维护一个长连接，而长连接的维护是需要资源成本的。该模型适合于的场景：
>    - **实时性要求较高**
>    - Client数量不多，Server数据变化较频繁
> 2. Pull模型：拉取模型。存在的问题是，实时性较差。
> 3. Long Polling模型：长轮询模型。

#### 1.4.5 客户端NameServer选择策略

> 这里的客户端指的是Producer和Consumer。

客户端首先选择**随机策略**进行的选择，失败后采用的是**轮询策略**。

**随机策略**

客户端在配置时必须要写上NameServer集群的地址，客户端首先会生产个随机数，然后再与NameServer节点数量取模，此时得到的就是所要连接的节点索引，然后就会进行连接。

**轮询策略**

逐个NameServer进行连接尝试。 



## 2. 工作流程

1. 启动NameServer, NameServer启动后开始监听端口，等待Broker、 Producer、 Consumer连接。
2. 启动Broker时，Broker会与所有的NameServer建立并保持长连接，然后每30秒向NameServer定时发送心跳包。
3. 收发消息前，可以先创建Topic，**创建Topic时需要指定该Topic要存储在哪些Broker上**，当然，在创建Topic时也会将Topic与Broker的关系写入到NameServer中。不过，这步是可选的，也可以在发送消息时自动创建Topic。
4. Producer发送消息，启动时先跟NameServer集群中的其中一台建立长连接，并从NameServer中获取路由信息，即当前发送的Topic的Queue与Broker的地址（IP+Port）的映射关系。然后根据算法策略从队选择一个Queue，与队列所在的Broker建立长连接从而向Broker发消息。当然，在获取到路由信息后，Producer会首先将路由信息缓存到本地，再每30秒从NameServer更新一次路由信息。
5. Consumer跟Producer类似， 跟其中一台NameServer建立长连接， 获取其所订阅Topic的路由信息，然后根据算法策略从路由信息中获取到其所要消费的Queue，然后直接跟Broker建立长连接，开始消费其中的消息。Consumer在获取到路由信息后，同样也会每30秒从NameServer更新一次路由信息。 不过不同于Producer的是，Consumer还会向Broker发送心跳， 以确保Broker的存活状态。

## 3. 架构特点

### 3.1 NameServer 特点

- **轻量无状态**：NameServer 是无状态的服务节点，彼此之间没有数据同步。
- **集群部署简单**：直接启动多个节点即可形成集群，Producer 和 Consumer 可连接任意一个 NameServer。

### 3.2 Broker 架构

- **主从架构（Master-Slave）**：
  - 一个 Broker 可以作为 Master，也可以作为 Slave。
  - 每个 Slave 只能对应一个 Master，但一个 Master 可以拥有多个 Slave。
  - Master 与 Slave 之间通过配置中的 `BrokerName` 相同、`BrokerId` 不同来绑定，其中 `BrokerId=0` 表示 Master，非 0 表示 Slave。
- **Topic 路由注册**：
  - 每个 Broker 启动后，会与所有 NameServer 节点建立长连接，并定时将自己所管理的 Topic 信息上报给 NameServer。
- **读写分离**：
  - 当前版本的 RocketMQ 支持从 Slave 拉取消息，但仅当 `BrokerId=1` 的从节点才参与消费负载。

### 3.3 Producer 特点

- **无状态、可横向扩展**：Producer 不存储任何状态信息，可以无限横向扩展。
- **连接机制**：
  - 与任意一个 NameServer 建立长连接，定期拉取 Topic 的路由信息；
  - 再与相关 Broker（一般是 Master）建立长连接；
  - 定时向 Broker 发送心跳包保持连接活跃。

### 3.4 Consumer 特点

- **灵活消费模型**：支持 Push 和 Pull 两种消息消费模式。
- **连接机制**：
  - 与 NameServer 建立长连接获取路由信息；
  - 再与 Broker（包括 Master 和 Slave）建立连接；
  - 根据消息偏移量、从节点可用性等因素智能选择消费源（Master 或 Slave）。



## 3. 集群搭建方式

### 3.1 NameServer 集群部署

- 每个 NameServer 节点独立部署，无需额外配置。
- 只需启动多个 NameServer 进程，Producer 和 Consumer 会随机连接其中之一。
- 建议部署至少两个以上节点以提高可用性。

### 3.2 Broker 集群部署

- 部署前应先配置好主从关系及相应参数：
  - `brokerClusterName`：所属集群名称。
  - `brokerName`：逻辑 Broker 名称（Master 与 Slave 相同）。
  - `brokerId`：`0` 表示 Master，非 0 表示 Slave。
- 启动后，Broker 会主动向所有 NameServer 注册。
- 可以为每个 Topic 配置写入和读取的 Broker 数量。
- 注意：自动创建 Topic 的功能需开启 `autoCreateTopicEnable=true`。

### 3.3 Producer 集群部署

- Producer 集群间相互独立，无需协同工作。
- 启动时连接 NameServer，拉取 Topic 的队列信息；
- 根据负载均衡策略轮询选择消息队列，将消息发送至对应的 Broker。

### 3.4 Consumer 集群部署

- Consumer 集群中的实例之间无状态，可独立消费消息。
- 支持两种消费模式：
  - **集群模式（Clustering）**：多实例分摊消费负载；
  - **广播模式（Broadcasting）**：所有实例都消费相同的消息。
- 消费者根据负载与状态决定从 Master 还是 Slave 拉取消息，以实现高吞吐与容灾。



## 4. 集群模式

### 4.1 单Master模式
这种方式风险较大，一旦Broker重启或者宕机时，会导致整个服务不可用。不建议线上环境使用，可以用于本地测试。
### 4.2 多Master模式
一个集群无slave，全是Master，例如2个Master或者3个Master，这种模式的优缺点如下：

- 优点：配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢(异步刷盘丢失少量消息，同步刷盘一条不丢)，性能最高。
- 缺点：单台机器宕机期间，这台机器上末被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。

### 4.3 Master多Slave模式(异步)

每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟(毫秒级)， 这种模式的优缺点如下：

- 优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，同时Master宕机后，消费者仍然可以从Slave消费,而且此过程对应用透明，不需要人工干预，性能同多Master模式几乎一样。
- 缺点：Master宕机，磁盘损坏情况下会丢失少量消息。

### 4.4 多Master多Slave模式(同步)

每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，即只有主备都写成功，才向应用返回成功，这种模式的优缺点如下：

- 优点：数据与服务都无单点故障，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高。
- 缺点：性能比异步复制模式略低(大约低10%左右) ，发送单个消息的RT会略高，且目前版本在主节点宕机后，备机不能自动切换为主机。



# 五、源码剖析

## 1. 整体结构

![rocketmq_lcsanmn](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_lcsanmn.png)

### 1.1 核心模块

| 模块                         | 功能说明                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| `broker [rocketmq-broker]`   | 消息中转站，负责接收生产者发送的消息并存储、处理消费者的拉取请求等，是消息传输的核心组件。 |
| `client [rocketmq-client]`   | 包含生产者（Producer）和消费者（Consumer）的客户端 SDK，是用户与 RocketMQ 交互的主要入口。 |
| `namesrv [rocketmq-namesrv]` | NameServer模块，负责路由注册与发现，为客户端提供 Topic 与 Broker 的映射关系。 |



### 1.2 基础与公共模块

| 模块                                  | 功能说明                                                     |
| ------------------------------------- | ------------------------------------------------------------ |
| `common [rocketmq-common]`            | 公共工具类和常量定义，例如配置、枚举、异常等。               |
| `remoting [rocketmq-remoting]`        | RocketMQ 自定义的 RPC 通信框架，底层使用 Netty，处理网络传输相关逻辑。 |
| `store [rocketmq-store]`              | 消息存储模块，管理消息的持久化，包括 CommitLog、ConsumeQueue 等。 |
| `tieredstore [rocketmq-tiered-store]` | 分层存储模块，用于支持冷热数据分层管理，例如内存、SSD、HDD 甚至远程存储。 |
| `srvutil [rocketmq-srvutil]`          | 服务端通用工具类，例如日志、配置解析、JVM 启动工具等。       |



### 1.3 扩展模块

| 模块                               | 功能说明                                                     |
| ---------------------------------- | ------------------------------------------------------------ |
| `auth [rocketmq-auth]`             | 权限认证模块，支持访问控制，如 ACL（访问控制列表）。         |
| `filter [rocketmq-filter]`         | 消息过滤器模块，实现消息属性、标签等过滤逻辑。               |
| `proxy [rocketmq-proxy]`           | 提供 HTTP/gRPC 等代理访问方式，用于异构系统接入 RocketMQ。   |
| `container [rocketmq-container]`   | 类似于 Spring Boot 的整合启动方式，快速以容器形式启动 RocketMQ 实例。 |
| `controller [rocketmq-controller]` | 新引入的集群控制模块，用于协调 Broker 的主从切换与控制逻辑。 |



### 1.4 工具与示例

| 模块                         | 功能说明                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| `tools [rocketmq-tools]`     | 命令行工具，如 `mqadmin`，提供 Topic 创建、Broker 管理等功能。 |
| `example [rocketmq-example]` | 示例代码模块，演示 Producer、Consumer 使用方式。             |



### 1.5 构建与配置文件

| 文件/文件夹                                | 说明                                                   |
| ------------------------------------------ | ------------------------------------------------------ |
| `pom.xml`                                  | Maven 的主构建配置文件，管理各模块依赖。               |
| `.bazel*` / `BUILD.bazel` / `MODULE.bazel` | Bazel 构建支持配置（RocketMQ 也支持 Bazel 构建方式）。 |
| `.gitignore`, `LICENSE`, `NOTICE` 等       | 项目元数据和开源协议信息。                             |



### 1.6 其他模块

| 模块                                     | 说明                                                        |
| ---------------------------------------- | ----------------------------------------------------------- |
| `distribution [rocketmq-distribution]`   | RocketMQ 的整体分发打包配置模块，包含启动脚本、配置模板等。 |
| `openmessaging [rocketmq-openmessaging]` | OpenMessaging 标准接口实现（云厂商通用接口标准）。          |
| `dev`, `style`, `docs`                   | 分别是开发工具、代码风格配置、文档相关资源。                |



## 2. Broker模块

### 2.1 整体结构

![rocketmq_mksajioc](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_mksajioc.png)

#### 2.1.1 核心类说明

| 类名                     | 功能简介                                                     |
| ------------------------ | ------------------------------------------------------------ |
| `BrokerStartup`          | **Broker 启动类**，启动流程入口，配置加载 -> 控制器初始化 -> 启动服务。 |
| `BrokerController`       | **Broker 核心控制器**，集中管理各类服务、调度线程、配置加载等，Broker 的大脑。 |
| `BrokerPathConfigHelper` | Broker 路径相关配置辅助类。                                  |
| `BrokerPreOnlineService` | Broker 预上线服务，如注册前的准备工作、环境检查等。          |
| `RocksDBConfigManager`   | 如果使用 RocksDB 作为存储引擎或索引引擎，此类用于管理其配置。 |
| `ShutdownHook`           | Broker 优雅关闭时执行的钩子逻辑，释放资源、刷盘等。          |

#### 2.1.2 包结构功能说明

| 包名           | 功能简介                                                     |
| -------------- | ------------------------------------------------------------ |
| `auth`         | 处理 Broker 端的权限认证相关逻辑，配合 ACL 使用。            |
| `client`       | 处理客户端连接与管理，如生产者、消费者与 Broker 的连接状态。 |
| `coldctr`      | 冷热数据控制，可能用于分层存储控制（Cold Data Controller）。 |
| `config`       | 配置管理，管理 Broker 的启动参数、Topic 配置、消费者组配置等。 |
| `controller`   | 与集群控制器交互的逻辑，如 Broker 主从切换、控制器通知处理等（配合 controller 模块使用）。 |
| `dledger`      | DLedger 是一种 Raft 实现，用于主从同步、选举（主从高可用）。 |
| `failover`     | Broker 故障转移相关逻辑。                                    |
| `filter`       | 消息过滤机制，如 TAG、SQL 表达式过滤等。                     |
| `latency`      | 延迟故障机制，例如消息发送延迟的处理策略。                   |
| `loadbalance`  | 负载均衡相关逻辑，通常用于 Broker 或消息队列的分配策略。     |
| `longpolling`  | 消费端长轮询机制的支持，用于提高拉消息的实时性。             |
| `metrics`      | RocketMQ 监控指标采集实现。                                  |
| `mqtrace`      | 消息轨迹追踪相关逻辑（比如消息是否被消费、在哪个 Broker 上消费等）。 |
| `offset`       | 消费位点管理，负责记录和维护每个消费者的消费位置（offset）。 |
| `out`          | 一些消息转发或出站处理逻辑。                                 |
| `pagecache`    | 页面缓存管理（与 mmap 机制相关的缓存技术）。                 |
| `plugin`       | 插件机制，支持对 Broker 功能进行扩展。                       |
| `pop`          | POP 模式的消息支持，是类似 Ack 的一种轻量消费机制。          |
| `processor`    | Broker 各种请求的处理器，如发送消息、拉取消息、心跳等请求。  |
| `schedule`     | 定时消息投递支持。                                           |
| `slave`        | 与从节点（Slave）相关的逻辑，如同步、复制等。                |
| `subscription` | 订阅关系的处理与校验。                                       |
| `topic`        | Topic 的元数据管理和操作逻辑。                               |
| `transaction`  | 消息事务处理，支持事务消息的半提交、回查等机制。             |
| `util`         | 工具类集合，如路径、配置辅助、日志等通用功能。               |



### 2.2 BrokerStartup

#### 2.2.1 属性

##### 2.2.1.1 log

```java
public static Logger log;
```

日志，静态变量，记录运行信息。

##### 2.2.1.2 CONFIG_FILE_HELPER

```java
public static final SystemConfigFileHelper CONFIG_FILE_HELPER = new SystemConfigFileHelper();
```

配置文件加载器，内部类常量，SystemConfigFileHelper为内部类。



#### 2.2.2 内部类

##### 2.2.2.1 SystemConfigFileHelper

**作用**

用来加载配置文件信息，传入配置文件路径，即可读取配置信息。

**属性**

- ```java
  private static final Logger LOGGER = LoggerFactory.getLogger(SystemConfigFileHelper.class);
  ```

  日志，静态常量，记录运行信息。

- ```java
  private String file;
  ```

  文件路径，私有变量，用来存储对象当前要读取配置信息的配置文件的路径。

**方法**

- ```java
  public void setFile(String file) {
      this.file = file;
  }
  ```

  设置配置文件路径属性file。

- ```java
  public Properties loadConfig() throws Exception {
      InputStream in = new BufferedInputStream(Files.newInputStream(Paths.get(file)));
      Properties properties = new Properties();
      properties.load(in);
      in.close();
      return properties;
  }
  ```

  功能：根据file属性记录的配置文件路径，通过IO流和Properties类来加载配置信息，并返回。

- ```java
  public void update(Properties properties) throws Exception {
      LOGGER.error("[SystemConfigFileHelper] update no thing.");
  }
  ```

  功能：默认是在日志中写入报错信息，可以通过覆写来实现具体的更新配置文件的逻辑。

- ```java
  public String getFile() {
      return file;
  }
  ```

  获取当前file属性的值。



#### 2.2.3 方法

##### 2.2.3.1 main

```java
public static void main(String[] args) {
    start(createBrokerController(args));
}
```

功能：整个Broker的Maven项目的程序入口，创建一个BrokerController，并调用start方法启动Broker。

##### 2.2.3.2 createBrokerController

```java
public static BrokerController createBrokerController(String[] args) {
    try {
        BrokerController controller = buildBrokerController(args);
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }
        Runtime.getRuntime().addShutdownHook(new Thread(buildShutdownHook(controller)));
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }
    return null;
}
```

功能：调用buildBrokerController方法来创建一个BrokerController对象：

- 创建成功则返回。
- 创建失败则直接终止程序，若报错则打印报错信息并终止程序。

##### 2.2.3.3 buildBrokerController

```java
public static BrokerController buildBrokerController(String[] args) throws Exception {

    // 【1】设置远程通信版本（Netty Remoting 的版本控制）
    System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));

    // 【2】初始化核心配置类对象（配置信息默认为默认值）
    final BrokerConfig brokerConfig = new BrokerConfig();                  // Broker基本配置
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();  // Netty服务端配置（监听端口等）
    final NettyClientConfig nettyClientConfig = new NettyClientConfig();  // Netty客户端配置
    final MessageStoreConfig messageStoreConfig = new MessageStoreConfig(); // 消息存储配置
    final AuthConfig authConfig = new AuthConfig();                       // 认证授权配置
    nettyServerConfig.setListenPort(10911);        // 默认监听端口
    messageStoreConfig.setHaListenPort(0);         // 高可用默认端口初始化为 0

    // 【3】构建命令行参数解析对象（Options）并解析命令行参数
    Options options = ServerUtil.buildCommandlineOptions(new Options());
    CommandLine commandLine = ServerUtil.parseCmdLine(
        "mqbroker", args, buildCommandlineOptions(options), new DefaultParser());
    if (null == commandLine) {
        System.exit(-1); // 如果命令行解析失败，直接退出
    }

    // 【4】如果通过 -c 指定了配置文件路径，则读取该文件并解析为 Properties 对象
    Properties properties = null;
    if (commandLine.hasOption('c')) {
        String file = commandLine.getOptionValue('c');
        if (file != null) {
            CONFIG_FILE_HELPER.setFile(file);                   // 设置配置文件路径
            BrokerPathConfigHelper.setBrokerConfigPath(file);   // 供其他模块使用
            properties = CONFIG_FILE_HELPER.loadConfig();       // 加载配置文件为 Properties
        }
    }

    // 【5】将配置文件中的配置项注入到对应配置对象中
    if (properties != null) {
        properties2SystemEnv(properties);                           // 配置项写入系统环境变量
        MixAll.properties2Object(properties, brokerConfig);         // 更新 Broker 配置
        MixAll.properties2Object(properties, nettyServerConfig);    // 更新 Netty 服务端配置
        MixAll.properties2Object(properties, nettyClientConfig);    // 更新 Netty 客户端配置
        MixAll.properties2Object(properties, messageStoreConfig);   // 更新存储配置
        MixAll.properties2Object(properties, authConfig);           // 更新认证配置
    }

    // 【6】命令行参数覆盖配置文件（命令行优先级更高）
    MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), brokerConfig);

    // 【7】校验环境变量是否设置 RocketMQ_HOME，否则退出
    if (null == brokerConfig.getRocketmqHome()) {
        System.out.printf("Please set the %s variable in your environment " +
            "to match the location of the RocketMQ installation", MixAll.ROCKETMQ_HOME_ENV);
        System.exit(-2);
    }

    // 【8】校验 NameServer 地址合法性（支持多个地址，使用“;”分隔）
    String namesrvAddr = brokerConfig.getNamesrvAddr();
    if (StringUtils.isNotBlank(namesrvAddr)) {
        try {
            String[] addrArray = namesrvAddr.split(";");
            for (String addr : addrArray) {
                NetworkUtil.string2SocketAddress(addr); // 若非法会抛出异常
            }
        } catch (Exception e) {
            System.out.printf("The Name Server Address[%s] illegal, please set it as follows, " +
                    "\"127.0.0.1:9876;192.168.0.1:9876\"%n", namesrvAddr);
            System.exit(-3);
        }
    }

    // 【9】若当前角色为 SLAVE，则下调访问内存中消息比例，减轻系统负担
    if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
        int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
        messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
    }

    // 【10】根据 brokerRole 设置 brokerId（只有主节点是 0）
    if (!brokerConfig.isEnableControllerMode()) {
        switch (messageStoreConfig.getBrokerRole()) {
            case ASYNC_MASTER:
            case SYNC_MASTER:
                brokerConfig.setBrokerId(MixAll.MASTER_ID);
                break;
            case SLAVE:
                if (brokerConfig.getBrokerId() <= MixAll.MASTER_ID) {
                    System.out.printf("Slave's brokerId must be > 0%n");
                    System.exit(-3);
                }
                break;
            default:
                break;
        }
    }

    // 【11】若启用了 DLeger 日志（分布式日志复制），则 brokerId 为 -1
    if (messageStoreConfig.isEnableDLegerCommitLog()) {
        brokerConfig.setBrokerId(-1);
    }

    // 【12】DLeger 和 控制器模式不可同时启用
    if (brokerConfig.isEnableControllerMode() && messageStoreConfig.isEnableDLegerCommitLog()) {
        System.out.printf("The config enableControllerMode and enableDLegerCommitLog cannot both be true.%n");
        System.exit(-4);
    }

    // 【13】若没有显式设置 HA 监听端口，则默认监听端口 + 1
    if (messageStoreConfig.getHaListenPort() <= 0) {
        messageStoreConfig.setHaListenPort(nettyServerConfig.getListenPort() + 1);
    }

    // 【14】设置 broker 是否运行在容器环境中（false 代表不是）
    brokerConfig.setInBrokerContainer(false);

    // 【15】设置 broker 的日志目录（是否隔离日志目录）
    System.setProperty("brokerLogDir", "");
    if (brokerConfig.isIsolateLogEnable()) {
        System.setProperty("brokerLogDir", brokerConfig.getBrokerName() + "_" + brokerConfig.getBrokerId());
    }
    if (brokerConfig.isIsolateLogEnable() && messageStoreConfig.isEnableDLegerCommitLog()) {
        System.setProperty("brokerLogDir", brokerConfig.getBrokerName() + "_" + messageStoreConfig.getdLegerSelfId());
    }

    // 【16】打印配置（-p 打印配置项，-m 打印含注释的默认项），然后退出
    if (commandLine.hasOption('p')) {
        Logger console = LoggerFactory.getLogger(LoggerName.BROKER_CONSOLE_NAME);
        MixAll.printObjectProperties(console, brokerConfig);
        MixAll.printObjectProperties(console, nettyServerConfig);
        MixAll.printObjectProperties(console, nettyClientConfig);
        MixAll.printObjectProperties(console, messageStoreConfig);
        System.exit(0);
    } else if (commandLine.hasOption('m')) {
        Logger console = LoggerFactory.getLogger(LoggerName.BROKER_CONSOLE_NAME);
        MixAll.printObjectProperties(console, brokerConfig, true);
        MixAll.printObjectProperties(console, nettyServerConfig, true);
        MixAll.printObjectProperties(console, nettyClientConfig, true);
        MixAll.printObjectProperties(console, messageStoreConfig, true);
        System.exit(0);
    }

    // 【17】正式记录配置信息到 broker 日志
    log = LoggerFactory.getLogger(LoggerName.BROKER_LOGGER_NAME);
    MixAll.printObjectProperties(log, brokerConfig);
    MixAll.printObjectProperties(log, nettyServerConfig);
    MixAll.printObjectProperties(log, nettyClientConfig);
    MixAll.printObjectProperties(log, messageStoreConfig);

    // 【18】设置认证配置路径
    authConfig.setConfigName(brokerConfig.getBrokerName());
    authConfig.setClusterName(brokerConfig.getBrokerClusterName());
    authConfig.setAuthConfigPath(messageStoreConfig.getStorePathRootDir() + File.separator + "config");

    // 【19】创建 BrokerController 实例
    final BrokerController controller = new BrokerController(
        brokerConfig, nettyServerConfig, nettyClientConfig, messageStoreConfig, authConfig);

    // 【20】注册配置项到 Configuration，用于后续管理与回滚
    controller.getConfiguration().registerConfig(properties);

    // 【21】返回构建完成的控制器对象
    return controller;
}
```



##### 2.2.3.4 start

```java
public static BrokerController start(BrokerController controller) {
    try {
        controller.start();

        String tip = String.format("The broker[%s, %s] boot success. serializeType=%s",
            controller.getBrokerConfig().getBrokerName(), controller.getBrokerAddr(),
            RemotingCommand.getSerializeTypeConfigInThisServer());

        if (null != controller.getBrokerConfig().getNamesrvAddr()) {
            tip += " and name server is " + controller.getBrokerConfig().getNamesrvAddr();
        }

        log.info(tip);
        System.out.printf("%s%n", tip);
        return controller;
    } catch (Throwable e) {
        e.printStackTrace();
        System.exit(-1);
    }

    return null;
}
```

功能：通过调用BrokerController对象的start方法来启动Broker：

- 启动成功则记录到日志中。
- 启动失败则打印错误信息并终止程序。



##### 2.2.3.5 



### 2.3 BrokerController

#### 2.3.1 属性



#### 2.3.2 方法





## 3. Common模块

### 3.1 整体结构

#### 2.1.1 核心类说明

![rocketmq_lsqmns](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_lsqmns.png)

| 类名                          | 功能简介                                         |
| :---------------------------- | :----------------------------------------------- |
| `BrokerConfig`                | Broker配置类，用于存储Broker的配置信息。         |
| `MixAll`                      | 工具类，提供了一系列静态方法和常量。             |
|                               |                                                  |
| `AbstractBrokerRunnable`      | 抽象类，用于定义Broker的可运行任务。             |
| `BoundaryType`                | 枚举类型，定义了消息发送的边界类型。             |
| `BrokerConfigSingleton`       | 单例类，用于获取Broker配置实例。                 |
| `BrokerIdentity`              | 用于标识Broker身份的信息类。                     |
| `ConfigManager`               | 配置管理器，用于管理配置的加载和更新。           |
| `ControllerConfig`            | 控制器配置类，用于存储控制器的配置信息。         |
| `CountDownLatch2`             | 计数器锁，用于多线程同步。                       |
| `JraftConfig`                 | Jraft配置类，用于存储Jraft的配置信息。           |
| `KeyBuilder`                  | 用于构建键值的构建器类。                         |
| `LifecycleAwareServiceThread` | 生命周期感知服务线程，用于管理服务的启动和停止。 |
| `LockCallback`                | 锁回调接口，用于定义锁操作的回调逻辑。           |
| `MQVersion`                   | 消息队列版本类，用于管理版本信息。               |
| `ObjectCreator`               | 对象创建器，用于创建指定类型的对象。             |
| `Pair`                        | 键值对类，用于存储成对的数据。                   |
| `PopAckConstants`             | 常量类，定义了POP确认相关的常量。                |
| `ServiceState`                | 服务状态类，用于表示服务的当前状态。             |
| `ServiceThread`               | 服务线程类，用于执行后台服务任务。               |
| `SubscriptionGroupAttributes` | 订阅组属性类，用于存储订阅组的配置信息。         |
| `SystemClock`                 | 系统时钟类，提供获取当前时间的方法。             |
| `ThreadFactoryImpl`           | 线程工厂实现类，用于创建线程。                   |
| `TopicAttributes`             | 主题属性类，用于存储主题的配置信息。             |
| `TopicConfig`                 | 主题配置类，用于存储主题的配置信息。             |
| `TopicFilterType`             | 枚举类型，定义了主题过滤的类型。                 |
| `TopicQueueId`                | 主题队列ID类，用于标识主题队列。                 |
| `UnlockCallback`              | 解锁回调接口，用于定义解锁操作的回调逻辑。       |
| `UtilAll`                     | 工具类，提供了一系列通用的工具方法。             |

#### 2.1.2 包结构功能说明

![rocketmq_vdnajs](E:\各种资料\Java开发笔记\我的笔记\images\rocketmq_vdnajs.png)

| 包名             | 功能简介                                               |
| :--------------- | :----------------------------------------------------- |
| `action`         | 定义了消息队列中的动作，如发送、消费等。               |
| `annotation`     | 包含注解，用于提供元数据和代码标记。                   |
| `attribute`      | 定义了消息和系统的属性。                               |
| `chain`          | 可能用于处理消息的链式调用逻辑。                       |
| `coldctr`        | 冷热数据控制，可能用于分层存储控制。                   |
| `compression`    | 提供消息压缩和解压缩的功能。                           |
| `config`         | 配置管理，包括系统配置和主题配置等。                   |
| `consistenthash` | 实现一致性哈希算法，用于负载均衡。                     |
| `constant`       | 定义了系统中使用的常量。                               |
| `consumer`       | 包含消费者相关的类和接口。                             |
| `fastjson`       | 集成FastJSON库，用于JSON的序列化和反序列化。           |
| `filter`         | 定义了消息过滤的逻辑。                                 |
| `future`         | 处理异步操作的结果。                                   |
| `hook`           | 提供钩子功能，允许在消息处理的不同阶段插入自定义逻辑。 |
| `logging`        | 包含日志记录相关的类。                                 |
| `message`        | 定义了消息的模型和相关操作。                           |
| `metrics`        | 提供度量和监控功能。                                   |
| `namesrv`        | 与命名服务相关的类，用于服务发现。                     |
| `producer`       | 包含生产者相关的类和接口。                             |
| `queue`          | 定义了消息队列的结构和操作。                           |
| `resource`       | 管理资源，如文件和网络连接。                           |
| `running`        | 可能与运行时环境相关。                                 |
| `state`          | 管理服务的状态信息。                                   |
| `statistics`     | 收集和分析统计数据。                                   |
| `stats`          | 可能是统计信息的简写。                                 |
| `sysflag`        | 定义了系统标志，用于控制消息的路由和处理。             |
| `thread`         | 包含线程管理相关的类。                                 |
| `topic`          | 定义了主题模型和相关操作。                             |
| `utils`          | 提供各种工具类和辅助方法。                             |



### 3.2 BrokerConfig

#### 3.2.1 属性



#### 3.2.2 方法



### 3.3 MixAll

#### 3.3.1 属性

全部属性均为静态常量属性，用来记录配置变量和各种信息。

##### 3.3.1.1 环境变量与系统属性

用于从环境变量或 Java 启动参数中获取配置项：

```java
public static final String ROCKETMQ_HOME_ENV = "ROCKETMQ_HOME"; // 环境变量名，RocketMQ 安装目录
public static final String ROCKETMQ_HOME_PROPERTY = "rocketmq.home.dir"; // Java 系统属性名，RocketMQ 安装目录
public static final String NAMESRV_ADDR_ENV = "NAMESRV_ADDR"; // 环境变量名，NameServer 地址
public static final String NAMESRV_ADDR_PROPERTY = "rocketmq.namesrv.addr"; // Java 系统属性名，NameServer 地址
public static final String MESSAGE_COMPRESS_TYPE = "rocketmq.message.compressType"; // 消息压缩类型
public static final String MESSAGE_COMPRESS_LEVEL = "rocketmq.message.compressLevel"; // 消息压缩级别
```

##### 3.3.1.2 NameServer 地址与域名

RocketMQ 使用 NameServer 做服务注册，这些是相关配置：

```java
public static final String DEFAULT_NAMESRV_ADDR_LOOKUP = "jmenv.tbsite.net"; // 默认的域名查找地址
public static final String WS_DOMAIN_NAME = System.getProperty("rocketmq.namesrv.domain", DEFAULT_NAMESRV_ADDR_LOOKUP); // 可配置域名
public static final String WS_DOMAIN_SUBGROUP = System.getProperty("rocketmq.namesrv.domain.subgroup", "nsaddr"); // 子域名分组
```

##### 3.3.1.3 默认生产者/消费者组名

这些组名用作 RocketMQ 系统的默认通信组：

```java
public static final String DEFAULT_PRODUCER_GROUP = "DEFAULT_PRODUCER";
public static final String DEFAULT_CONSUMER_GROUP = "DEFAULT_CONSUMER";
public static final String TOOLS_CONSUMER_GROUP = "TOOLS_CONSUMER";
public static final String SCHEDULE_CONSUMER_GROUP = "SCHEDULE_CONSUMER";
public static final String FILTERSRV_CONSUMER_GROUP = "FILTERSRV_CONSUMER";
public static final String MONITOR_CONSUMER_GROUP = "__MONITOR_CONSUMER";
public static final String CLIENT_INNER_PRODUCER_GROUP = "CLIENT_INNER_PRODUCER";
public static final String SELF_TEST_PRODUCER_GROUP = "SELF_TEST_P_GROUP";
public static final String SELF_TEST_CONSUMER_GROUP = "SELF_TEST_C_GROUP";
public static final String ONS_HTTP_PROXY_GROUP = "CID_ONS-HTTP-PROXY";
public static final String CID_ONSAPI_PERMISSION_GROUP = "CID_ONSAPI_PERMISSION";
public static final String CID_ONSAPI_OWNER_GROUP = "CID_ONSAPI_OWNER";
public static final String CID_ONSAPI_PULL_GROUP = "CID_ONSAPI_PULL";
public static final String CID_RMQ_SYS_PREFIX = "CID_RMQ_SYS_";
```

##### 3.3.1.4 心跳和订阅控制参数

控制客户端与 Broker 的心跳、订阅变更等：

```java
public static final String IS_SUPPORT_HEART_BEAT_V2 = "IS_SUPPORT_HEART_BEAT_V2";
public static final String IS_SUB_CHANGE = "IS_SUB_CHANGE";
```

##### 3.3.1.5 本地地址与主机名

获取本地网络配置（IP、主机名）：

```java
public static final List<String> LOCAL_INET_ADDRESS = getLocalInetAddress(); // 本地IP列表
public static final String LOCALHOST = localhost(); // 本地主机名或IP
```

##### 3.3.1.6 字符集与系统相关

字符集、OS类型、进程ID等：

```java
public static final String DEFAULT_CHARSET = "UTF-8";
public static final long CURRENT_JVM_PID = getPID(); // 当前JVM进程ID
private static final String OS = System.getProperty("os.name").toLowerCase(); // 当前操作系统名
```

##### 3.3.1.7 Broker 和消息配置

和主/从节点、逻辑队列、压缩、追踪、重试、DLQ 等有关的常量：

```java
public static final long MASTER_ID = 0L;
public static final long FIRST_SLAVE_ID = 1L;
public static final long FIRST_BROKER_CONTROLLER_ID = 1L;

public final static int UNIT_PRE_SIZE_FOR_MSG = 28;
public final static int ALL_ACK_IN_SYNC_STATE_SET = -1;

public static final String RETRY_GROUP_TOPIC_PREFIX = "%RETRY%";
public static final String DLQ_GROUP_TOPIC_PREFIX = "%DLQ%";
public static final String REPLY_TOPIC_POSTFIX = "REPLY_TOPIC";
public static final String UNIQUE_MSG_QUERY_FLAG = "_UNIQUE_KEY_QUERY";
```

##### 3.3.1.8 Trace、RPC、ACL 等系统字段

追踪消息、RPC 请求头、权限配置等：

```java
public static final String DEFAULT_TRACE_REGION_ID = "DefaultRegion";
public static final String CONSUME_CONTEXT_TYPE = "ConsumeContextType";
public static final String CID_SYS_RMQ_TRANS = "CID_RMQ_SYS_TRANS";
public static final String ACL_CONF_TOOLS_FILE = "/conf/tools.yml";
public static final String REPLY_MESSAGE_FLAG = "reply";

public final static String RPC_REQUEST_HEADER_NAMESPACED_FIELD = "nsd";
public final static String RPC_REQUEST_HEADER_NAMESPACE_FIELD = "ns";
```

##### 3.3.1.9 LMQ（逻辑消息队列）支持

逻辑队列相关配置项：

```java
public static final String LMQ_PREFIX = "%LMQ%";
public static final int LMQ_QUEUE_ID = 0;
public static final String LMQ_DISPATCH_SEPARATOR = ",";
```

##### 3.3.1.10 分区与区域配置

用于多区域部署时：

```java
public static final String ROCKETMQ_ZONE_ENV = "ROCKETMQ_ZONE";
public static final String ROCKETMQ_ZONE_PROPERTY = "rocketmq.zone";
public static final String ROCKETMQ_ZONE_MODE_ENV = "ROCKETMQ_ZONE_MODE";
public static final String ROCKETMQ_ZONE_MODE_PROPERTY = "rocketmq.zone.mode";
public static final String ZONE_NAME = "__ZONE_NAME";
public static final String ZONE_MODE = "__ZONE_MODE";
```

##### 3.3.1.11 日志与元数据

和日志或元信息模拟有关的内容：

```
java复制编辑private static final Logger log = LoggerFactory.getLogger(LoggerName.COMMON_LOGGER_NAME);
public static final String LOGICAL_QUEUE_MOCK_BROKER_PREFIX = "__syslo__";
public static final String LOGICAL_QUEUE_MOCK_BROKER_NAME_NOT_EXIST = "__syslo__none__";
public static final String METADATA_SCOPE_GLOBAL = "__global__";
```

##### 3.3.1.12 多路径分隔符

当 Broker 存储使用多个路径时：

```java
public static final String MULTI_PATH_SPLITTER = System.getProperty("rocketmq.broker.multiPathSplitter", ",");
```



#### 3.3.2 方法

全部方法均为静态方法，提供各种工具，比如判断操作系统类别、String和File类型转换、获取进程ID等等。

##### 3.3.2.1 操作系统判断相关

```java
public static boolean isWindows()
public static boolean isMac()
public static boolean isUnix()
public static boolean isSolaris()
```

- 用于判断当前操作系统类型，便于做平台差异化处理。

##### 3.3.2.2 RocketMQ 地址相关

```java
public static String getWSAddr()
```

- 生成 RocketMQ 的 nameserver web socket 地址（可考虑域名含端口与否的情况）。

##### 3.3.2.3 Topic 与 Consumer Group 工具方法

```java
public static String getRetryTopic(final String consumerGroup)
public static String getReplyTopic(final String clusterName)
public static String getDLQTopic(final String consumerGroup)
public static boolean isSysConsumerGroup(final String consumerGroup)
public static boolean isSysConsumerGroupAndEnableCreate(final String consumerGroup, final boolean isEnableCreateSysGroup)
public static boolean isPredefinedGroup(final String consumerGroup)
public static boolean isSysConsumerGroupPullMessage(String consumerGroup)
```

- 管理重试、回复、死信队列（DLQ）topic 的命名规则；
- 判断消费者组是否是系统组、预定义组；
- 判断是否允许拉取消息等。

##### 3.3.2.4 Broker 地址处理

```java
public static String brokerVIPChannel(final boolean isChange, final String brokerAddr)
```

- 若开启 VIP 通道，将原 broker 的端口减 2 得到新的地址。

##### 3.3.2.5 进程信息

```java
public static long getPID()
```

- 获取当前 Java 进程的 PID（通过 RuntimeMXBean 解析）。

##### 3.3.2.6 文件读写操作

```java
public static void string2File(...)
public static void string2FileNotSafe(...)
public static String file2String(...)
```

- 实现字符串和文件互转，同时支持备份和 URL 文件读取。

##### 3.3.2.7 Properties 工具方法

```java
public static void printObjectProperties(...)
public static Properties object2Properties(...)
```

- 可打印带有注解的字段，或者将对象字段转为 Properties。

```java
public static void properties2Object(final Properties p, final Object object)
```

- 用来将配置信息赋值给object的同名属性。通过反射机制，获取object中的set方法，从set方法的方法名中读取属性名，然后根据属性名去Properties对象中找响应的配置信息，若能找到，且属性的类型为基本数据类型，则将配置信息转类型赋值给属性变量。

```java
public static String properties2String(...)
public static Properties string2Properties(...)
public static boolean isPropertiesEqual(...)
public static boolean isPropertyValid(...)
```

- 处理 `Properties` 对象与字符串的互转、排序、验证等。

##### 3.3.2.9 网络信息

```java
public static List<String> getLocalInetAddress()
public static String getLocalhostByNetworkInterface()
```

- 获取本机所有 IP 地址；
- 获取非 loopback 本地地址，优先返回 IPv4。

##### 3.3.2.10 原子变量更新

```java
public static boolean compareAndIncreaseOnly(final AtomicLong target, final long value)
```

- 如果新值大于当前值则更新，常用于高并发场景中的最大值更新。

##### 3.3.2.11 人类可读的字节大小

```java
public static String humanReadableByteCount(long bytes, boolean si)
```

- 将字节转换为如 1.2MB、500KiB 这样的单位字符串。

##### 3.3.2.12 基本类型比较

```java
public static int compareInteger(int x, int y)
public static int compareLong(long x, long y)
```

- 包装了 `Integer.compare()` 和 `Long.compare()` 方法。

##### 3.3.2.13 LMQ相关

```java
public static boolean isLmq(String lmqMetaData)
```

- 判断是否为 LMQ 类型。

```java
public static boolean topicAllowsLMQ(String topic)
```

- 判断给定的 `topic` 是否允许使用 LMQ。

##### 3.3.2.14 文件路径处理

```java
public static String dealFilePath(String aclFilePath)
```

- 使用 `Path.normalize()` 来规范化路径格式（处理冗余的 `.`、`..` 等）。



