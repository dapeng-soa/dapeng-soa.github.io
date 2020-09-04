---
layout: doc
title:  "dapeng-event-bus 详细指南系列 --- 消费者详解"
---

上一篇我们详细介绍了 eventbus 生产者 基于 Kafka 来实现消息发布的原理和使用方法。本章我们将从另一个角度，消费者一端来详聊。
我们将在本章从零学习 基于 eventbus 如何来创建消费者并消费来自 Kafka 服务器上的消息。

![bj1IHuqdtWg.jpg](https://upload-images.jianshu.io/upload_images/6393906-c0493e38bff49870.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 1. Step1: 消费者项目依赖

#### 1). eventbus 组件依赖

- sbt 工程

```
"com.today" % "event-bus_2.12" % "2.1.1"
```

- maven 工程

```xml
<dependency>
    <groupId>com.today</groupId>
    <artifactId>event-bus_2.12</artifactId>
    <version>2.1.1</version>
</dependency>
```

<br/>

#### 2). 事件结构体 `API` 的依赖
`dapeng` 的 `eventbus` 发送的消息是通过 `thrift` 序列化为 `byte` 数组的，所以消费者再获取到消息后，需要根据事件对应的序列化器/反序列化器 将消息解码为消息对象。这个对象就是消息结构体。我们再之前的生产者中定义了，所以这里就要依赖生产者的`API`。

例子，我们依赖发送端 `demo` 的`api`

- sbt project

```
"com.today" % "event-bus_2.12" % "2.1.1",
"com.today" % "user-api_2.12" % "1.0.0"
```

- maven

```xml
<dependency>
    <groupId>com.today</groupId>
    <artifactId>event-bus_2.12</artifactId>
    <version>2.1.1</version>
</dependency>
<!--事件发送方api-->
<dependency>
    <groupId>com.today</groupId>
    <artifactId>user-api_2.12</artifactId>
    <version>1.0.0</version>
</dependency>
```

---

<br/>

### 2. Step2-1: 定义消费者类和监听方法
> 我们的业务服务采用 `Spring` 作为容器，采用 `Spring` 的声明式事务管理。因此，`eventbus` 消费者类设计为一个 `Spring` 类，可以使用到 `Spring` 的事务管理和其他很多特性。

#### 1). 前提和注意事项 (划重点)

- 第一点，消费者类必须是一个 `Spring bean` ，所以要么使用 `xml` 定义该`bean`，要么利用注解加扫包的逻辑，最终该类需要被 `Spring` 管理。
- 第二点，该类的类上需要加上 `eventbus` 中定义的注解 `@KafkaConsumer`，并且配置相应信息。相关例子如下:

```java
@KafkaConsumer(groupId = "demo_subscribe", topic = "member_test",sessionTimeout = 60000)
@Transactional(rollbackFor = Array(classOf[Throwable]))
class DemoConsumer {
 
}
```

#### 2). **`@KafkaConsumer`** 注解配置详解

![config.png](https://upload-images.jianshu.io/upload_images/6393906-99ec36d3e00f17e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

- 一个消费者类上必须定义 `@KafkaConsumer`。然后配置该消费者的 `groupId`，需要定义的 `topic` 和 消费者与 `kafka` 的会话超时时间（此项可选，大多数情况下不用填，默认即可。）

#### 3). 定义消费者方法
> 我们通过 `@KafkaConsumer` 来标明一个消费者 `Bean` 之后，就可以使用
`@KafkaListener` 注解来 `subscribe` 订阅和消费指定的事件了。

```scala
//订阅事件1
@KafkaListener(serializer = classOf[RegisteredEventSerializer])
def subscribeRegisteredEvent(event: RegisteredEvent): Unit = {
    println(s"收到消息 RegisteredEvent  ${event.id} ${event.userId}")
}
//订阅事件2
@KafkaListener(serializer = classOf[ActivedEventSerializer])
def subscribeRegisteredEvent(event: ActivedEvent): Unit = {
  println(s"收到消息 ActivedEvent  ${event.id} ${event.userId} ") 
}
```

<br/>

#### 4). 消费者类和方法注意事项

对于一个 `topic` ，例如 `member_test`，它是 `member` 微服务所有事件的 `Topic`(主题)，也就是说，这个 `topic` 中存了很多种类型的事件，因此。此时我们就通过 `@KafkaListener` 来订阅具体事件类型的消息。

如上面的例子中，在 `member_test` 这个主题中，有两种事件 `RegisteredEvent` 和 `ActivedEvent`。我们需要在 `KafkaListener` 定义该事件的序列化器。

最后，我们需要注意,**监听方法中 `序列化器` 和 `事件类型` 都要采用生成的 `Scala` 版本的代码。**

--- 

<br/>

### 3. Step2-2: 定义消费者类和监听方法 (接收消息元数据信息)

> 在第2点中我们介绍了，怎么定义消费者类和方法来接收消息。此时，我们可以对监听方法新增一个 `ConsumerContext` 参数，来让业务得到当前接受到的消息的元信息。比如消息所属的 `topic`，消息的 `offset` 和 `partition` ，以及消息的创建时间等等。

#### 1). 自定义带有 `ConsumerContext` 方法参数的监听方法
在自定义的消费者方法的参数中加入 `ConsumerContext` 参数，**注意此参数需要放在第一个位置**。

```scala
@KafkaListener(serializer = classOf[RegisteredEventSerializer])
def subscribeRegisteredEvent(context:ConsumerContext, event: RegisteredEvent): Unit = {
    println(s"消息元信息 $context.toString")
    println(s"收到消息 RegisteredEvent  ${event.id} ${event.userId}")
}
```

通过上面这种方式，我们就能过获取到当前消费者订阅的事件的信息。
**建议在日常开发中使用这种方式**，我们的代码只需要增加这个参数即可。关于数据的填充等 **`eventbus`** 内部会自动完成，无需我们自己操心。

#### 2). 关于 ConsumerContext 元信息的内容如下:

```java
public class ConsumerContext {
    //消息的key，可能为消息id，或者消息业务bizTag经过hash后的值。
    //取决于消息发送端有没有使用根据业务tag进行指定发送分区的策略。如果没有，则此key即为消息唯一id。
    private final long key;
    //消息所属 topic
    private final String topic;
    //消息的 offset
    private final long offset;
    //消息的分区信息
    private final int partition;
    //消息产生的时间戳
    private final long timestamp;
    //格式化后的消息创建时间 2018-11-26 12:21:21 321
    private final String timeFormat;
    //创建时间 or 更新时间，一般为创建时间
    private final String timestampType;
  
    //省略构造函数、getter setter 等
```

---

<br/>

### 4. Step3: 配置注解支持组件
> 使用 `eventbus` 组件来定义一个一个消费者很容易。只需要借助于几个注解。我们得益于基于 `Spring IOC` 强大的能力来做到这一点。为了能够让 `Spring` 来发现我们定义的消费者，我们还需要进行一些配置。

#### 1). 消费者类Bean 的发现与创建原理

注意，我们刚刚使用了 `@KafkaConsumer` 来标志一个消费者类。但是原生的Spring容器它是没办法知道你这个类就是消费者类的。所以我们还要向Spring 注册一个 `Bean`，该 `Bean` 实现了 `Spring` 的 `BeanPostProcessor` 接口，是Spring的一个扩展接口。我们注册这个bean的作用就是，让它去发现刚才用我们自定义的注解`@KafkaConsumer` 标注的类，让它成为一个消费者类。

#### 2). 配置自定义组件 MsgAnnotationBeanPostProcessor

`xml` 注册只需如下配置，注意该 `bean` 只能在 `Spring` 中注册一次。请不要注册多次(曾经发生过注册多次的现象，导致消费者不生效)。

```xml
<bean class="com.today.eventbus.spring.MsgAnnotationBeanPostProcessor"/>
```

#### 3).注解支持总结(划重点)
- 1.需要注册 **`MsgAnnotationBeanPostProcessor`** 这个 `bean`
- 2. 这个 `bean` 只能注册一次，请勿在 `spring` 配置文件中注册多次( `id` 不同注册多次)

---

<br/>

### 5. Step4: 消费者 kafka 日志多余 debug 日志的屏蔽
> Apache Kafka Client 客户端在日志级别为 DEBUG 的情况下会产生很多日志，这样可能为干扰我们正常的业务日志。
因此如果业务开发日志级别是DEBUG的情况下，我们需要针对此，屏蔽这些多余的日志信息。


#### 1).屏蔽策略

- 1.eventbus 类库产生的日志都记录到 xxx-event-bus 的日志文件中，与业务主日志区分开来。
- 2.eventbus 和 kafka 部分包开头的日志采用 INFO 级别。
- 3.下面是一个 logback 的 配置示例，注意 日志文件中的 xxx 用当前项目的简名替换。


```xml
<!--将eventbus包下面的日志都放入单独的日志文件里 dapeng-eventbus.%d{yyyy-MM-dd}.log-->
<appender name="eventbus" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <prudent>true</prudent>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>${soa.base}/logs/detail-xxx-eventbus.%d{yyyy-MM-dd}.log</fileNamePattern>
        <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
        <pattern>%d{MM-dd HH:mm:ss SSS} %t %p [%X{sessionTid}] - %m%n</pattern>
    </encoder>
</appender>

<!-- eventbus开头的所有类的日志存到 detail-xxx-eventbus 的日志文件中--> 
<logger name="com.today.eventbus" level="INFO" additivity="true">
    <appender-ref ref="eventbus"/>
</logger>


<!-- 将 kafka clients 下 所有类产生的日志级别 改为 info -->
<logger name="org.apache.kafka.clients" level="INFO"/>

<!-- 关掉 druid sql -->
<logger name="druid.sql" level="OFF"/>

```

---

<br/>

### 6. 消费者总结 

