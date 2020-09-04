---
layout: doc
title:  "KafkaMsgAgent 开发者配置指南"
---

kafka-msg-agent  以下简称 agent

## Project Git Url

`kafka-msg-agent` 主要功能在 `eventbus` 和 `openapi` 中实现，本工程只是作了整合和配置，工程 `git` 地址如下

工程地址: https://github.com/dapeng-soa/kafka-msg-agent


## Dependency

1.`kafka-msg-agent` 依赖 `eventbus` 和 `openapi` 两个 类库

```xml
<!--eventbus-->
<dependency>
      <groupId>com.today</groupId>
      <artifactId>event-bus_2.12</artifactId>
      <version>${eventbus.version}</version>
</dependency>
<!--open-api -->
<dependency>
      <groupId>com.github.dapeng-soa</groupId>
      <artifactId>dapeng-open-api</artifactId>
      <version>${dapeng.version}</version>
</dependency>
```
2. `agent` 和异构系统之间是通过 `http` 进行传输和请求的。`kafka-msg-agent` 提供了三种 `http` 请求类库。
如果没有显示指定，默认采用 `OKHttp`。

- [OKHttp](http://square.github.io/okhttp/)
- [HttpClient](http://hc.apache.org/httpclient-3.x/)
- [AsyncHttpClient](https://github.com/AsyncHttpClient/async-http-client)

分别加入依赖如下
```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.5</version>
</dependency>

<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>3.11.0</version>
</dependency>

<dependency>
      <groupId>org.asynchttpclient</groupId>
      <artifactId>async-http-client</artifactId>
      <version>2.6.0</version>
</dependency>
```

## Docker Compose

```yml
kafkaMsgAgent:
    container_name: kafkaMsgAgent
    image: docker.today36524.com.cn:5000/basic/kafka-msg-agent:2.1.1
    restart: on-failure:3
    environment:
      - LANG=zh_CN.UTF-8
      - TZ=CST-8
      - soa_zookeeper_host=127.0.0.1:2181
      - soa_kafka_host=127.0.0.1:9092
    volumes:
       # kafka-msg-agent 日志映射到宿主机配置
       - "/data/logs/kafka-msg-agent:/kafka-msg-agent/logs"
       # kafka-agent 消费者配置文件(核心配置)
       - "/data/var/config/msgAgent/rest-consumer.xml:/kafka-msg-agent/conf/rest-consumer.xml"
```
需要配置项：

- agent 连接的 `zookeeper` 集群地址。使用环境变量 `soa_zookeeper_host` 注入
- agent 连接的 `kafka` 集群地址。使用环境变量 `soa_kafka_host` 注入
- agent 消费者配置信息，使用 `rest-consumer.xml` 文件配置


## rest-consumer.xml

```xml
<consumer-groups>
    <consumer-group id="member">
        <group-id>phpEventGroup</group-id>
        <topic>member_demo</topic>
        <kafka-host>soa_kafka_host</kafka-host>
        <service>com.demo.service.DemoService</service>
        <version>1.0.0</version>
        <thread-count>1</thread-count>
        <consumers>
            <consumer>
                <event-type>com.event.scala.demo.RegisterEvent</event-type>
                <event>com.event.demo.RegisterEvent</event>
                <destination-url>http://127.0.0.1:8080/subscribe/</destination-url>
            </consumer>
        </consumers>
    </consumer-group>

    <consumer-group id="order">
        <group-id>orderGroup</group-id>
        <topic>order_demo</topic>
        <kafka-host>soa_kafka_host</kafka-host>
        <service>com.demo.service.OrderService</service>
        <version>1.0.0</version>
        <thread-count>1</thread-count>
        <consumers>
            <consumer>
                <event-type>com.event.scala.demo.RegisterEvent</event-type>
                <event>com.event.demo.RegisterEvent</event>
                <destination-url>http://127.0.0.1:8080/subscribe/</destination-url>
            </consumer>
        </consumers>
    </consumer-group>

    <consumer-group id="app">
        <group-id>appGroup</group-id>
        <topic>order_demo</topic>
        <kafka-host>soa_kafka_host</kafka-host>
        <service>com.demo.service.AppService</service>
        <version>1.0.0</version>
        <thread-count>1</thread-count>
        <!-- eventbus async 分支功能，可以指定使用哪种 http client -->
        <http-type>3</http-type>
        <consumers>
            <consumer>
                <event-type>com.event.scala.demo.RegisterEvent</event-type>
                <event>com.event.demo.RegisterEvent</event>
                <destination-url>http://127.0.0.1:8080/subscribe/</destination-url>
            </consumer>
        </consumers>
    </consumer-group>
</consumer-groups>
```

**`consumer-group` 下面可以定义多个消费者，这些消费者有如下信息**

- 消费者组需要根据 `groupId` 和 `topic` 两者组成唯一。不同的消费者组之间这两个组合必须不同。
- 在上面的基础上，每一个消费者组 `process` 的消息应该来自一个 `service` 服务，如 `xml` 中定义的 `service`
- `thread-count` 定义当前消费者组会起多少个消费者实例线程进行消费。
    - 指导意见，可根据当前订阅 `topic` 的分区数来决定使用多少线程实例。
    - 例如 `order_demo` 主题存在 `16` 个分区，我们部署两个 `agent` 节点。那么上述订阅 `order_demo` 的消费者组的 `thread-count` 可以设置为 `8` 条。
- `http-type` 指定使用哪种 `http` 请求客户端。不设置默认为 `OKHttp`.
    - 设置 1 对应 `OKHttp`
    - 设置 2 对应 `HttpClient`
    - 设置 3 对应 `AsyncHttpClient`


**`consumers` 下面可以定义多个根据 eventType 划分的消费者**

前置知识，在一个 `topic` 中，会存在各种各样的 `eventType` 的事件，这些事件类型不一样。所以我们将每一个不同 `eventType` 的事件使用 一个 逻辑上的 `Consumer` 来定义。这里 `consumers` 下面包含了很多 `consumer` 的定义。换言之，一个 topic 下面有多种不同事件，这些事件定义配置的集合组成 `Consumers`。

**`consumer` 定义事件消费单元**
一个具体的业务上的事件类型构成了一个消费者逻辑。虽然在物理上，可以使用 KafkaConsumer 订阅一个 topic ，然后获取到多个事件类型的消息，这时候，根据获取到的事件类型去调用封装好的Consumer 逻辑。
这样，一个消费者可以处理多个 consumer 单元的消息了。

- `event-type`：这个是事件消息定义类的全限定名，一般为代码自动生成，采用 scala 的生成的事件类。
 - `event`： 这个是 thrift 中定义的事件的全名 (namespace+事件名)，便于通过此配置从服务元数据中获取到当前事件的元数据信息(编解码信息)。
- `destination-url`：这条事件处理逻辑，即通过 http 调用异构系统的 url

## Summary

- 1.一个 `ConsumerGroup` 下面的消费者定义，应该处于同一个消费者组，并且订阅相同的 `Topic`，并且元数据 `service` 是同一个。
- 2.如果订阅同一个 `topic` 的事件的 元数据服务不是同一个，需要单独定义新的 `ConsumerGroup`
- 3.`ConsumerGroup` 根据 `topic` 和 `groupId` 唯一













