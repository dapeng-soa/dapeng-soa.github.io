---
layout: doc
title:  "dapeng-event-bus 详细指南系列 --- 消息代理 KafkaMsgAgent"
---

上一篇我们详细介绍了 `eventbus` 生产者 基于 `Kafka` 来实现消息发布的原理和使用方法。本章我们将从另一个角度，消费者一端来详聊。
我们将在本章从零学习 基于 `eventbus` 如何来创建消费者并消费来自 `Kafka` 服务器上的消息。

![image.png](https://upload-images.jianshu.io/upload_images/6393906-52e184e4040870f6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1. KafkaMsgAgent 为何诞生？

`KafkaMsgAgent` 主要应用于异构系统的消息消费场景。由于 `dapeng` 的事件总线目前将消息通过 `thrift` 序列化后存储在 `Kafka` 中的，所以，消费者也必须采用 thrift 机制将其解码。如果是 Java 系统的消费者还是可以直接对消息进行监听的。

如果监听者项目是非 `Java` 系统(比如 `PHP` 系统)，此时就没办法使用 
 `eventbus` 组件中的消费者模块来对消息进行消费了。

![image.png](https://upload-images.jianshu.io/upload_images/6393906-4416087bf81c35c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因此，我们提供了 `KafkaMsgAgent` 消息代理中间件。该中间件实质上是一个消费者工程。它会通过配置去监听对应的 `Topic` 和 事件类型。然后它会将二进制的消息解码为 `Json` 的格式。这时候，`PHP` 系统提供一个基于 `restful` 的 `http` 接口，`kafkaAgent` 会请求这个接口，然后将消息类型和解码后的消息内容传送给 `PHP` 提供的接口。这样间接的实现了 异构系统对 `dapeng` 事件的监听。

### 2. 消息的生产
> 整个系统由产生消息开始。从上图我们假定 **A服务** 是消息的产生方。

A服务可能会产生多条消息，这里我们假定A服务会产生 RegisteredEvent 和 ActivedEvent 两种事件，分别代表注册成功事件和激活成功事件。 相关消息结构如下：

#### 1).事件的 thrift 定义
```thrift
/**
* 注册成功事件,
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

/**
* 激活事件
**/
struct ActivedEvent {
    /**
    * 事件Id
    **/
    1: i64 id,
    /**
    * 用户ID
    **/
	2: i64 userId
}
```
#### 2). 根据 thrift 生成 scala 代码后的事件(Scala 样例类，非常简洁)

```java
// 注册成功事件
case class RegisteredEvent(id: Long, userId: Long)
//激活事件
case class ActivedEvent(id: Long,userId: Long)
```

#### 3). 在 service.thrift 中指明是某个方法触发事件。

```thrift
service UserService {
/**
 * 用户注册
 */
demo_response.RegisterUserResponse registerUser (1: demo_request.RegisterUserRequest request)
(events="com.today.user.events.RegisteredEvent,com.today.user.events.ActivedEvent")

}(group="demo")
```
由于 `registerUser` 方法会触发两个事件，所以我们会在此注明两个事件的 `thrift` 的全名，这里的全名是 thrift 中定义的 `namespace` 加上 事件的名称。  (**区别于此 `thrift` 生成 `scala` 代码后生成的代码的包名**)。

#### 4). 消息的发送
消息的发送流程在之前的文章中已有介绍。🔗: [消息发送](https://www.jianshu.com/p/be1c7f94dcb5)

### 3. KafkaMsgAgent 消息代理
> 这一个环节是整个消息代理的核心，我们将具体介绍如何通过配置来使用它。

#### 1). 配置消费者监听指定 topic 和 事件类型

# TODO

```xml
<consumer-groups>
    <consumer-group id="member">
        <group-id>phpEventGroup</group-id>
        <topic>member_1.0.0_event</topic>
        <kafka-host>soa_kafka_host</kafka-host>
        <service>com.today.api.member.service.MemberService</service>
        <version>1.0.0</version>
        <thread-count>1</thread-count>
        <consumers>
            <consumer>
                <event-type>com.today.api.member.scala.events.MemberRegisterByWechatOpenIdEvent</event-type>
                <event>com.today.api.member.events.MemberRegisterByWechatOpenIdEvent</event>
                <destination-url>https://wechat-lite.today36524.com/api/dapeng/subscribe/index</destination-url>
            </consumer>

            <consumer>
                <event-type>com.today.api.member.scala.events.ConsumeFullEvent</event-type>
                <event>com.today.api.member.events.ConsumeFullEvent</event>
                <destination-url>https://wechat-lite.today36524.com/api/dapeng/subscribe/index</destination-url>
            </consumer>

            <consumer>
                <event-type>com.today.api.member.scala.events.OrderCancelEvent</event-type>
                <event>com.today.api.member.events.OrderCancelEvent</event>
                <destination-url>https://wechat-lite.today36524.com/api/dapeng/subscribe/index</destination-url>
            </consumer>
        </consumers>
    </consumer-group>

    <consumer-group id="order">
        <group-id>phpEventGroup</group-id>
        <topic>order_1.0.0_event</topic>
        <kafka-host>soa_kafka_host</kafka-host>
        <service>com.today.api.order.service.OrderService2</service>
        <version>1.0.0</version>
        <thread-count>1</thread-count>
        <consumers>
            <consumer>
                <event-type>com.today.api.order.scala.events.StoreOrderEndEventNew</event-type>
                <event>com.today.api.order.events.StoreOrderEndEventNew</event>
                <destination-url>http://demo-app.today.cn/api/dapeng/subscribe/index</destination-url>
            </consumer>
        </consumers>
    </consumer-group>
    <consumer-group id="order2">
        <group-id>pdaEventGroup</group-id>
        <topic>order_1.0.0_event</topic>
        <kafka-host>soa_kafka_host</kafka-host>
        <service>com.today.api.order.service.AppOrderService</service>
        <version>1.0.0</version>
        <thread-count>1</thread-count>
        <consumers>
            <consumer>
                <event-type>com.today.api.order.scala.events.PdaMessageFireEventNew</event-type>
                <event>com.today.api.order.events.PdaMessageFireEventNew</event>
                <destination-url>http://demo-app.today.cn/api/dapeng/subscribe/index</destination-url>
            </consumer>
        </consumers>
    </consumer-group>
</consumer-groups>
```






































