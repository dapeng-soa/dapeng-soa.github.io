---
layout: doc
title:  "dapeng-event-bus 详细指南系列 --- 生产者详解"
---

dapeng eventbus 是基于dapeng 微服务下的各个服务之间的消息总线组件。它能够灵活的解耦各个业务之间的关系。使用起来非常方便，只需进行部分配置，用户即可以很方便的使用。
这篇文章我们将会讲解，eventbus 生产者一方的详细使用过程和注意事项。

![-x-Brii2QaM.jpg](https://upload-images.jianshu.io/upload_images/6393906-64d03671c762606c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)


## EventBus 生产者
> 事件消息产生的一方微服务，例如订单和采购服务。门店订货、`POS` 订单完成时，需要触发一个事件，将此事件发送给 `Kafka`，后续消息的处理订阅等生产者方的服务不用再关心，只需发送消息即可。

### 1.引入依赖
> 在 `build.sbt` 中引入 事件消息 `lib` 依赖，如果项目是 `maven` 工程，按照 `maven` 的格式在 `pom` 中引入即可。

- sbt

```
"com.today" % "event-bus_2.12" % "2.1.1"
```

- maven

```xml
<dependency>
    <groupId>com.today</groupId>
    <artifactId>event-bus_2.12</artifactId>
    <version>2.1.1</version>
</dependency>
```

---

### 2.生产者消息原理

> 为了保证发送的消息的一致性以及事务性质，我们为消息建立了一个临时的中转站，将序列化的消息保存到数据库中。借助于数据库事务，我们能够将消息的存储和业务逻辑置于一个事务下，可以一起提交和回滚。我们可以模拟一个步骤。

#### 1). 时刻A

业务逻辑处理一半，因为某些功能完成，于是调用 `EventBus.fireEvent` 触发事件。此时事件消息会序列化为二进制，并存储在我们后面要讲的事件表中。
<br/>

#### 2). 时刻B

业务逻辑继续往下执行。这里会有两种情况：
- 1.整个流程处理成功，没有异常，事务提交。随着事务的提交消息也存储到事件表中成功。并会打印如下日志。

```log
11-26 19:24:58 801 dapeng-container-biz-pool-9 INFO [ac180007c16816b1] - 
save message unique id: 12233961, 
eventType: com.today.api.order.scala.events.StoreOrderEndEventNew successful
```

该日志存储在某个业务项目的 eventbus日志下。
格式为:`detail-xxx-eventbus.2018-11-26.log`。
如果出现这条日志，说明业务逻辑处理成功，并且成功触发了消息。**注意，这里触发了消息不一定意味着消息发送出去了**，第二个步骤我在下面讲。
- 2.业务流程在某一个环节出错 ，并抛了异常，此时整个事务会回滚，存储的消息也会回滚，整个过程下来，没有产生消息，就像一切未发生一样。
<br/>

#### 3). 时刻C

> 这个过程与上面的两个过程 **时刻A** 和 **时刻B** 是解耦的关系。换句话说，这个过程是一个独立的过程，因为上面的步骤已经将序列化后的消息**存储**到了**事件表**中。上一个步骤不用再关心后续逻辑。

有需要生产消息的微服务需要配置一个**定时消息轮询组件**，传入数据源等信息。

```xml
 <!--messageScheduled 单独事务管理,不需要敏感性数据源-->
<bean id="messageTask" class="com.today.eventbus.scheduler.MsgPublishTask" init-method="startScheduled">
        <constructor-arg name="topic" value="${kafka_topic}"/>
        <constructor-arg name="kafkaHost" value="${kafka_producer_host}"/>
        <constructor-arg name="tidPrefix" value="${kafka_tid_prefix}"/>
        <constructor-arg name="dataSource" ref="order_dataSource"/>
        <property name="serviceName" value="com.today.api.order.service.OrderService2"/>
</bean>
```

**上述配置解析**
该配置为一个 `Spring Bean` ，初始化方法指定 `startScheduled`，该方法会启动一个定时器，由 `ScheduledExecutorService` 实现，默认的定时触发时间为 `300ms`，我们可以通过设置环境遍历来修改此值。
单位为毫秒

```properties
soa.eventbus.publish.period = 500
```

定时器组件主要的作用是去轮询查找刚才**时刻AB**成功后存储到事件表中的事件消息。每次会最多查询50条记录，然后将这些消息直接发送到kafka中去，如果发送成功了，会打印如下日志。然后发送成功的消息会被删除。

**所以正常情况下**，事件表（`dp_common_event`）表是一个空表，在事件产生的高峰期可能会有一定的堆积。所以，这一个环节，事件表的堆积情况可以反应生产者消息产生的速度，某些情况下需要对其进行监控。

```log
11-26 19:24:59 072 dapeng-eventbus--scheduler-0 INFO [] - 
bizProducer:批量发送消息 id:(List(12233961, 12233962, 12233963)),size:[3] 
 to kafka broker successful
```

如果我们看到上面这个日志，说明消息是真正的发送 `kafka` 上面了。此时，发送端圆满完成任务，后续过程会将由消费端去进行处理了。
<br/>

#### 4). 生产者原理总结

- **步骤1**: 业务逻辑处理成功，调用 `EventBus.fireEvent()`(需要用户手动调用)
- **步骤2**: 定时器自动轮询消息并发送(配置好即可，对用户不可见)
- **细节**：注意查看两个步骤的 **日志** 来定位消息是否发送成功。

---

### 3.数据库消息表准备

> 这里我们需要两个表。存储业务触发事件的事件表（**`dp_common_event`**）和针对微服务多个实例定时器的**分布式锁**，采用数据库**悲观锁**实现。这个专用事件锁的表为（**`dp_event_lock`**）。

#### 1). dp_common_event 表 schema

```sql
CREATE TABLE `dp_common_event` (
  `id` bigint(20) NOT NULL COMMENT '事件id，全局唯一, 可用于幂等操作',
  `event_type` varchar(255) DEFAULT NULL COMMENT '事件类型',
  `event_biz` varchar(255) DEFAULT NULL COMMENT '事件biz内容(分区落地)',
  `event_binary` blob COMMENT '事件内容',
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='事件存储表';
```

我们详细来说明事件表各个字段的作用：

- **`id`**:  此为每一条事件消息的唯一id，由分布式取号服务生成，全局唯一。
- **`event_type`**:  此为当前消息的类型。因为每一个微服务可能存在多个事件类型，所以一个topic产生的消息会有多种类型。该字段保存的是事件类的全限定名。
- **`event_biz`**:  **可选项**，如果消费者需要对事件发送按顺序消费，即将相同的 `biztag` 业务内容的消息发送到一个分区去，避免了业务消费方并行消费多个分区相同业务 `bizTag` 时导致数据库竞争死锁等等一系列异常。如果发送事件时，没有带 `bizTag`，则此处存储为 `null`。
- **`event_binary`:** 为事件消息序列化为二进制后存储字段。
- **`updated_at `:** 消息创建时间。

需要注意的是，`blob` 最大存储字节为 `65k`，如果我们一次产生的消息过大，超过了这个大小，将会导致消息存储失败，进而影响整个业务逻辑。所以如果有这种情况，我们可以将 `event_binary` 字段的类型改为 `MediumBlob` ，它支持 `16M` 的字节大小。

#### 2). dp_event_lock 表 schema
```sql
CREATE TABLE `dp_event_lock` (
  `id` int(11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='事件锁Lock表';
```

**注意**：事件锁表在创建之后需要新增一条数据，用这条数据来作为锁。利用了 mysql 的读行级锁的特性。

```sql
insert into `dp_event_lock` set id = 1, name = 'event_lock';
```

`eventbus` 定时轮询生产者内部会使用这条记录来锁住当前操作，相关源码如下：

```scala
withTransaction(dataSource)(conn => {
  //如果生产部署多实例，这里同时有多个定时器会去处理事件，所以利用下面这句话来在
  //同一时间有且仅有一个定时器组件进来并做后续消息的 fetch 和 send 操作
  val lockRow = row[Row](conn, sql"SELECT * FROM dp_event_lock WHERE id = 1 FOR UPDATE")
  // 做后面消息的的 fetch 和 send 逻辑        
})
```

--- 

### 4.事件 IDL 定义
> 事件类型和业务内容需要定义为结构体。待后续序列化和反序列化。

#### 1). 消息定义格式注意事项
1.以事件双方约定的消息内容定义IDL结构体
2. 规定必须为每个事件定义事件ID，以便消费者做消息幂等

`==> events.thrift`

```thrift
namespace java com.github.dapeng.user.events

/**
* 注册成功事件, 由于需要消费者做幂等,故加上事件Id
**/
struct RegisteredEvent {
    /**
    * 事件Id
    **/
    1: i64 id,
    /**
    * 用户id
    **/
    2: i64 userId
}
```
#### 2). IDL服务接口事件声明(是谁触发当前事件)
- 接口可能会触发一个或多个事件
- 事件必须在触发接口进行声明，否则事件解码器不会生成

`== >user_service.thrift`

```thrfit
namespace java com.github.dapeng.user.service

include "user_domain.thrift"
include "events.thrift"

/**
* 事件发送端业务服务
**/
service UserService{
/**
# 用户注册
## 事件
    注册成功事件，激活事件
**/
    string register(user_domain.User user)
    (events="events.RegisteredEvent,events.ActivedEvent")

}(group="EventTest")
```

#### 3). 在spring的配置文件`spring/services.xml`进行定义，注意`init-method`指定`startScheduled`
> 上面我们在讲解生产者原理时，讲解到了定时器组件，作用是查询事件表中的消息，并将此消息发送到 `Kafka` 上。

```xml
<!--messageScheduled 定时发送消息bean-->
<bean id="messageTask" class="com.today.eventbus.scheduler.MsgPublishTask" init-method="startScheduled">
    <constructor-arg name="topic" value="${kafka_topic}"/>
    <constructor-arg name="kafkaHost" value="${kafka_producer_host}"/>
    <constructor-arg name="tidPrefix" value="${kafka_tid_prefix}"/>
    <constructor-arg name="dataSource" ref="demo_dataSource"/>
    <!--可选项 >>1  -->
    <property name="serviceName" value="com.today.api.order.service.OrderService2"/>
</bean>
```

#### 4). 各个配置含义详解

- **`topic`**
Kafka 消息 topic，每一个微服务区分(建议:**服务简名_版本号_event**)。
例如 `order`，使用 topic 为 `order_1.0.0_event`

-  **`kafkaHost`**
kafka集群地址，例如：
**`192.168.100.1:9092,192.168.100.2:9092,192.168.100.3:9092`**

-  **`tidPrefix`** 
kafka事务生产者的前缀，我们规定按服务名来界定。例如
**`kafka_tid_prefix=order_1.0.0`**

- **`dataSource`** 
使用业务的 `dataSource` ,这里不需要使用事务敏感的 `datasource`，因为事务由定时器组件自己管理。该 `datasource` `ref `在 `spring` 中配置的 `datasource`。

####  5). 一个完整的配置如下
> **`config_user_service.properties`**

```properties
# event config
kafka_topic=user_1.0.0_event
kafka_producer_host=127.0.0.1:9092
kafka_tid_prefix=user_1.0.0
```

定时器轮询间隔默认为 `100ms`，上面已经有提到。我们可以通过环境变量修改默认的轮询时间，如果是在开发环境下，或者在测试环境下，我们可以将轮询时间设置长一点，比如2s:

```
soa.eventbus.publish.period=2000 
```

代表轮询数据库消息库时间，如果对消息及时性很高，请将此配置调低，建议最低为100ms，默认配置是300ms。

---

### 5.生产者消息事件管理器EventBus
> 在做事件触发前,你需要实现 `AbstractEventBus` ,并将其交由spring托管，来做自定义的本地监听分发。

#### 1). 定义消息管理器 EventBus
直接在你项目下定义如下类,注意 dispatchEvent 内的内容可根据具体业务要求实现，**如果没有本地订阅，直接为一个空方法即可**。

```scala
object EventBus extends AbstractEventBus {

  /**
    * 事件在触发后，可能存在本地的监听者，以及跨领域的订阅者
    * 本地监听者可以通过实现该方法进行分发
    * 同时,也会将事件发送到其他领域的事件消息订阅者
    * @param event
    */
  override def dispatchEvent(event: Any): Unit = {
    //此内容可为空
    event match {
      case e: RegisteredEvent => // do somthing 
      case _ =>  log.trace(" do nothing ")
    }
  }
  
  override def getInstance: EventBus.this.type = this
}
```

#### 2). 在 **`Spring`** 的配置文件 `services.xml` 中 注册这个 `EventBus` `Bean`
> spring/services.xml

```xml
<bean id="eventBus" class="com.github.dapeng.service.commons.EventBus" factory-method="getInstance">
    <property name="dataSource" ref="tx_demo_dataSource"/>
</bean>
```

**注意细节**
该 `eventbus` 需要传入业务的事务敏感性 dataSource ，这样可以保证，eventbus 存储消息时，可以和业务逻辑使用同一个连接，这样就可以处于同一个事务之中。
如果是手动管理事务的情况，请参考后文。

#### 3). 业务事件发布(触发)
> 目前由于一些业务，需要对相同的 `BizTag` 分组，希望相同的 `Tag` 的消息能够发送到相同的 `Kafka` 分区中去。所以这里提供两种触发事件的方法，第一种是对没有此需求的普通生产者而言，第二种需要加入一个 bizTag，我们分别介绍。

在 `EventBus` 的父类 `AbstractEventBus` 中定义了两个触发事件的方法，如下

```scala
def fireEvent(event: Any): Unit = {
    dispatchEvent(event)
    persistenceEvent(None, event)
}
/**
* 顺序的触发事件，需要多传入一个业务内容。
* 然后会根据这个内容将相同的tag消息发送到一个分区。
*/
def fireEventOrdered(biz: String, event: Any): Unit = {
    dispatchEvent(event)
    persistenceEvent(Option.apply(biz), event)
}
```

#### 4). 不需要对消息做分区要求的触发事件方式

```scala
EventBus.fireEvent(RegisteredEvent(event_id,user.id))
```

#### 5). 需要对消息做分区要求的触发事件方式

我们这里要传入定义的具有业务意义的 `bizTag`，这个具体需由业务决定。
例如现在有一个订单的完成事件，我们希望相同门店的订单都发往一个分区，因此这里的 `BizTag` 可以选择订单号。相关发送消息如下：

```scala
val orderNo = "123456"
EventBus.fireEvent(orderNo,RegisteredEvent(event_id,user.id))
```
通过上述这种方式，就可以做到根据订单号进行 hash 然后指定到 Kafka 某一个分区，只要是此门店产生的消息都会路由到一个分区中去。关于 kafka 分区的概念，如果比较模糊，可以从网上查询相关资料了解。

---

### 6.记录日志屏蔽
> 生产方因为轮询数据库发布消息，如果间隔很短，会产生大量的日志，需要修改级别，在logback下进行如下配置。

**注意**：配置文件中会对 `eventbus` 的日志单独使用一个文件进行存储，该文件名需要用户根据自己的微服务进行自定义。例如下面配置中的 `detail-goods-eventbus`，中间的 `goods` 就代表当前微服务为 `goods` 服务。**所以注意不要照着下面配置完全复制到你的项目下的 `logback` 配置下**。

```xml
<!--将eventbus包下面的日志都放入单独的日志文件里 dapeng-eventbus.%d{yyyy-MM-dd}.log-->
<appender name="eventbus" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <prudent>true</prudent>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
					
        <fileNamePattern>${soa.base}/logs/detail-goods-eventbus.%d{yyyy-MM-dd}.log</fileNamePattern>
        <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
        <pattern>%d{MM-dd HH:mm:ss SSS} %t %p - %m%n</pattern>
    </encoder>
</appender>

    <!-- additivity:是否向上级(root)传递日志信息, -->
    <!--com.today.eventbus包下的日志都放在上面配置的单独的日志文件里-->
    <logger name="com.today.eventbus" level="INFO" additivity="false">
        <appender-ref ref="eventbus"/>
    </logger>

    <!--sql 日志显示级别-->
    <logger name="druid.sql" level="OFF"/>
    <logger name="wangzx.scala_commons.sql" level="INFO"/>
    <logger name="org.apache.kafka.clients" level="INFO"/>
    <logger name="org.springframework.jdbc.datasource.DataSourceUtils" level="INFO"/>
```

---



### END

到这里，消息总线生产者一方的所有使用方式讲解完毕。我们将会在下一章讲解 消费者一方的消息使用过程。

### 示例项目 Samples

- [生产者demo](https://github.com/leihuazhe/producer-demo)
- [消费订阅者demo](https://github.com/leihuazhe/consumer-demo)

- [dapeng eventBus](https://github.com/dapeng-soa/dapeng-event-bus)

