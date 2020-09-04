---
layout: doc
title: DDD事件模型
---

## 服务事件模型
> Dapeng 服务都是基于领域驱动模型进行划分的，各个微服务之间各司其职，设计到多领域间的交互，一般以领域事件进行通讯。这里就诞生了领域事件这一概念。

### 事件总线
- 我们将发出事件通知的一方称为发送者 (`Publisher`) ,关心事件的一方称为订阅者 (`Subscriber`)。
- 关心一件事，便会收集这件事情相关的信息。而这些都将会转换为消息流，在订阅这件事情的领域间传播，一旦命中所要关心的事情，就由订阅者自行去处理接下来的事情。

![事件模型](http://www.struy.top/18-3-12/24835978.jpg)

上图为`eventbus`示意图大致流程是这样的：

- 服务接口触发事件
- eventbus 分发事件，如果存在领域内订阅者，直接分发到指定订阅者，再将事件消息存库定时发送至 kafka
- 如果不存在领域内订阅者，事件消息直接存库并定时发送 kafka
- 消息在发送成功以后会被清除，为了保证事务的一致性。建议事件db共享业务数据源
- 订阅者只需要订阅事件双方规约好的 topic 和事件类型就可以命中需要的事件消息


# 事件发布(生产者，Producer)


## 前置条件
### 1.需要发送消息的项目依赖jar包
- sbt项目在`build.sbt`里加入如下依赖
```xml
"com.today" % "event-bus_2.12" % "0.2-SNAPSHOT"
```
- maven项目在`pom.xml`中加入如下依赖：
```xml
<dependency>
    <groupId>com.today</groupId>
    <artifactId>event-bus_2.12</artifactId>
    <version>0.2-SNAPSHOT</version>
</dependency>
```


### 2.数据库存储支持，需在业务数据库中加入此表
```SET NAMES utf8;
SET FOREIGN_KEY_CHECKS = 0;

DROP TABLE IF EXISTS `dp_common_event`;
CREATE TABLE `dp_common_event` (
  `id` bigint(20) NOT NULL COMMENT '事件id，全局唯一, 可用于幂等操作',
  `event_type` varchar(255) DEFAULT NULL COMMENT '事件类型',
  `event_binary` blob DEFAULT NULL COMMENT '事件内容',
  `updated_at` timestamp NOT NULL DEFAULT current_timestamp() ON UPDATE current_timestamp() COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
--  Table structure for `event_lock`
-- ----------------------------
DROP TABLE IF EXISTS `dp_event_lock`;
CREATE TABLE `dp_event_lock` (
  `id` int(11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
--  Records of `event_lock`
-- ----------------------------
BEGIN;
INSERT INTO `dp_event_lock` VALUES ('1', 'event_lock');
COMMIT;

SET FOREIGN_KEY_CHECKS = 1;
```


### 3.IDL定义
- 以事件双方约定的消息内容定义IDL结构体
- 规定必须为每个事件定义事件ID，以便消费者做消息幂等

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
### 4.IDL服务接口事件声明(是谁触发当前事件)
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
### 在spring的配置文件`spring/services.xml`进行定义，注意`init-method`指定`startScheduled`
> 这里采用的是同步模式，当然`eventbus`也支持异步模式

```xml
<!--messageScheduled 定时发送消息bean-->
<bean id="messageTask" class="com.today.eventbus.scheduler.MsgPublishTask" init-method="startScheduled">
    <constructor-arg name="topic" value="${kafka_topic}"/>
    <constructor-arg name="kafkaHost" value="${kafka_producer_host}"/>
    <constructor-arg name="tidPrefix" value="${kafka_tid_prefix}"/>
    <constructor-arg name="dataSource" ref="tx_demo_dataSource"/>
</bean>
```
- topic kafka消息topic，领域区分(建议:领域_版本号_event)
- kafkaHost kafka集群地址(如:127.0.0.1:9091,127.0.0.1:9092)
- tidPrefix kafka事务id前缀，领域区分
- dataSource 使用业务的 dataSource

`==>config_user_service.properties`

```xml
# event config
kafka_topic=user_1.0.0_event
kafka_producer_host=127.0.0.1:9092
kafka_tid_prefix=user_1.0.0
```
## 重点： 配置轮询发布消息的时间间隔，以ms为单位，在dapeng.properties中配置
```xml
soa.eventbus.publish.period=500 //代表轮询数据库消息库时间，如果对消息及时性很高，请将此配置调低，建议最低为100ms，默认配置是1000ms
```

### 事件触发
- 在做事件触发前,你需要实现 `AbstractEventBus` ,并将其交由spring托管，来做自定义的本地监听分发

`==>commons/EventBus.scala`
```scala
object EventBus extends AbstractEventBus {

  /**
    * 事件在触发后，可能存在本地的监听者，以及跨领域的订阅者
    * 本地监听者可以通过实现该方法进行分发
    * 同时,也会将事件发送到其他领域的事件消息订阅者
    * @param event
    */
  override def dispatchEvent(event: Any): Unit = {
    event match {
      case e:RegisteredEvent =>
        // do somthing 
      case _ =>
        LOGGER.info(" nothing ")
    }
  }
  override def getInstance: EventBus.this.type = this
}
```
- 当本地无任何监听时==>
```scala
override def dispatchEvent(event: Any): Unit = {}
```
`==> spring/services.xml`
```xml
<bean id="eventBus" class="com.github.dapeng.service.commons.EventBus" factory-method="getInstance">
    <property name="dataSource" ref="tx_demo_dataSource"/>
</bean>
```
- 事件发布
```scala
EventBus.fireEvent(RegisteredEvent(event_id,user.id))
```
---

# 生产方因为轮询数据库发布消息，如果间隔很短，会产生大量的日志，需要修改级别，在logback下进行如下配置：

```xml
<!--将eventbus包下面的日志都放入单独的日志文件里 dapeng-eventbus.%d{yyyy-MM-dd}.log-->
<appender name="eventbus" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <prudent>true</prudent>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
					注意： 这里detail-  后面 加自己系统的名字。 例如这里的 goods
        <fileNamePattern>${soa.base}/logs/detail-goods-eventbus.%d{yyyy-MM-dd}.log</fileNamePattern>
        <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
        <pattern>%d{MM-dd HH:mm:ss SSS} %t %p - %m%n</pattern>
    </encoder>
</appender>

<!-- additivity:是否向上级(root)传递日志信息, -->
<!--com.today.eventbus包下的日志都放在上面配置的单独的日志文件里-->
<logger name="com.today.eventbus" level="DEBUG" additivity="false">
    <appender-ref ref="eventbus"/>
</logger>

<!--sql 日志显示级别-->
<logger name="druid.sql" level="OFF"/>
<logger name="wangzx.scala_commons.sql" level="DEBUG"/>
<logger name="org.apache.kafka.clients.consumer.KafkaConsumer" level="INFO"/>
<logger name="org.springframework.jdbc.datasource.DataSourceUtils" level="INFO"/>

```




---

# 事件订阅 (消费者 Consumer)
## 依赖
除需要向上面生产者一样依赖eventbus的jar包外,还需要依赖生产者端的api jar包
```xml
<!--事件发送方api-->
<dependency>
    <groupId>com.today</groupId>
    <artifactId>user-api_2.12</artifactId>
    <version>0.1-SNAPSHOT</version>
</dependency>

<!--if => sbt project--> 
"com.today" % "event-bus_2.12" % "0.2-SNAPSHOT",
"com.today" % "user-api_2.12" % "0.2-SNAPSHOT"
```
注解支持配置：

```xml
<bean id="postProcessor" class="com.today.eventbus.spring.MsgAnnotationBeanPostProcessor"/>
```
附(kafka日志级别调整)：
`==>logback.xml`

```xml
<logger name="org.apache.kafka.clients.consumer" level="INFO"/>
```
事件订阅者配置
```java
// java

@KafkaConsumer(groupId = "eventConsumer1", topic = "user_1.0.0_event",kafkaHostKey = "kafka.consumer.host"))
public class EventConsumer {
    @KafkaListener(serializer = RegisteredEventSerializer.class)
    public void subscribeRegisteredEvent(RegisteredEvent event){
        LOGGER.info("Subscribed RegisteredEvent ==> {}",event.toString());
    }
}
```
一定要在spring中上下文中注册消费者bean
==>spring.xml
```xml
<bean name="eventConsumer1" class="com.today.service.event.EventConsumer" />
```
## 注意： 订阅方在消费消息时，如果处理消息可能会抛出业务异常（就是业务有关的异常，如前置检查不通过，等等),在消费消息时，需要捕获业务系统。

```scala
@KafkaListener(serializer = classOf[ModifySkuBuyingPriceEventSerializer])
def modifySkuBuyingPriceEvent(event: ModifySkuBuyingPriceEvent): Unit = {
  // 重点
 try {
    logger.info(s"=====> ModifySkuBuyingPriceEvent")
    val ModifySkuBuyingPriceItemList = event.modifySkuBuyingPriceEventItems.map(
      x => build[ModifySkuBuyingPriceConsumer](x)()
    )
    val result = consumer.modifySkuBuyingPrice(ModifySkuBuyingPriceItemList) // 收到事件后调用业务接口示例
    logger.info(s"收到消息$event =>成功修改sku进价， ${result} ")
  } catch {
  //logger的写法自己定义
    case e: SoaException => logger.error("业务抛出的异常，消息不会重试", e)
  }

}
```

//scala
```scala
serializer = classOf[RegisteredEventSerializer]
```


#### @KafkaConsumer
- groupId 订阅者领域区分
- topic 订阅的 kafka 消息 topic 
- kafkaHostKey 可自行配置的kafka地址，默认值为`dapeng.kafka.consumer.host`。可以自定义以覆盖默认值
    - 用户只要负责把这些配置放到env或者properties里面
    - 如：`System.setProperty("kafka.consumer.host","127.0.0.1:9092");`

#### @KafkaListener
- serializer 事件消息解码器，由事件发送方提供.
## 既是消费者也是订阅者？
> 如果服务既有接口会触发事件，也存在订阅其他领域的事件情况。只要增加缺少的配置即可 



## 重点可以看如下发布者demo
```xml
https://github.com/leihuazhe/publish-demo
```




## 示例项目
- [事件发送端demo(sbt)](https://github.com/leihuazhe/publish-demo)
- [事件订阅端demo(sbt)](http://pms.today36524.com.cn:8083/basic-services/consumer-demo)
- [事件订阅端demo(maven)](https://github.com/StruggleYang/event-consumer)
- [eventbus](http://pms.today36524.com.cn:8083/basic-services/eventBus)