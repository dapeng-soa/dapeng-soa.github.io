---
layout: doc
title:  "使用 kafka dump tool 来解码和分析Kafka消息"
---

# Kafka Dump Tool  (以下简称 KDT)
> 目前我司 `Rpc` 框架 `dapeng-soa` 附属周边产品 `Eventbus` 发送和接收的消息是经过 `Thrift` 协议序列化后的。`kafka` 会将消息持久化到 `broker` 端。在开发过程中，可能我们需要知道发送的消息内容是什么。因此需要一个途径能够从 `broker` 已存储的消息中取出自己想要的消息内容。

## Kafka Dump Tool 需求点
1.根据消息 `topic` 和 `offset` 指定获取到具体的消息内容。
2.根据 `topic` 和 `offset` 获取一定范围内的消息内容。
3.根据消息发送时间模糊查询出这段时间内某个 `topic` 内的消息内容。

## 实现原理
1.消息通过特定的序列化器被序列化后存到了 `kafka` 内。所以要知道指定消息的元数据信息。元数据信息通过 `xml` 或者上传的方式以文件的方式告知给 `KDT`。`KDT` 解析元数据信息。放入内存。这样，前提解码器已经准备就绪。
2.通过指定 `offset` 和指定需要解析的消息条数设置拉取消息。
- 原理
`KDT` 会启动一个消费者，通过 `consumer.peek(offset)` 的形式将消费者拉取偏移量设置到指定设置处。这样可以将 `broker` 中的消息拉取下来。然后通过元数据拿到 `thrift` 消息解码器。将消息解码，展示给用户界面。当拉取了指定数量的消息后，停止消费者（或者暂停）。拉取结束。

3.通过指定时间的拉取方式。

## 注意点
- 关键点就是如何选取 `offset`，最好要提供多种方案。
比如：指定 `offset`，从最末尾开始，从某个时间点开始。
以及查询某个 `groupId` 的当前 `offset`。
输出格式包括 `JSON`, `pretty formatted JSON` 等，能够方便进行 `shell` 的管道操作。



## 使用 Dump 工具进行消息 Dump 
> KDT的功能已整合到 dapeng-cli中。我们可以使用 dapeng-cli来对消息进行 dump 操作。

### 打开 dapeng-cli 键入 dump 
> cli 控制台打印出了详细的使用方法，具体如图所示

![image.png](https://upload-images.jianshu.io/upload_images/6393906-4475c4417dc67e7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 正常消息 dump 后显示形式

![dump tool.png](https://upload-images.jianshu.io/upload_images/6393906-223ad221491ce20c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)


### 1.设置 zk
> 设置 zk 的目的是获取到zk中已注册的服务的元数据信息。需要提前设置好，默认是 `127.0.0.1:2181` ，如果为默认情况，不需要进行设置。

```sh
set -zkhost 192.168.10.2:2181
```


### 2. dump

#### 1.显示最新的一条消息，以及它的 offset 、partition
> 如果不指定 `-broker` 默认 127.0.0.1:9092

```sh
dump  -topic member_test 
dump  -broker 127.0.0.1:9092 -topic member_test 
```

初始显示目前消费者持有的当前的 `topic` 的所有分区信息。

![partition&offset.png](https://upload-images.jianshu.io/upload_images/6393906-72bfe73226466219.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上述命令没有指定具体分区 `partition` 以及分区的 `offset` 信息。默认 `dump tool` 会显示当前 `broker` 中最新的消息的 `offset`，并且没有可 `dump` 消息。如果这时候有新消息发送到 `broker`，这时候 `dump tool` 就会解码最新的消息。

#### 2. 指定 offset 不显示指定 partition

```sh
dump -broker 127.0.0.1:9092 -topic member_test -offset 200000
```

命令界面首先会显示用户刚设置的 `offset` 信息。但是可能由于用户设置的 `offset` 超过了目前分区最大的 `offset` 信息，因此kafka 会重新 `Resetting offset` 信息，3s后，显示的最新的信息才是当前最正确的 `offset` 信息。

![显示信息.png](https://upload-images.jianshu.io/upload_images/6393906-85aee5e50622c921.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3. 指定分区信息和 offset 信息

```sh
dump -broker 127.0.0.1:9092  -topic member_test  -partition 1 -offset 40
```

![指定.png](https://upload-images.jianshu.io/upload_images/6393906-d7a98997e9c93cd1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

指定分区和 `offset` 信息后，控制台显示当前分区 `1`， 和用户 `seek` 到的 `offset` 40，由于目前 `broker` 最新的偏移量为 `42`，因此这时会 `dump` offset 为 40 和 41两条消息。

#### 4.指定消费消息的范围
使用 `-limit` 来限制 从指定的 offset 开始 拉取的消息数量。如下表示，从 offset 20 开始 ，消费 10条消息。

```sh
dump -broker 127.0.0.1:9092 -topic member_test -offset 20 -limit 10
```

这时会 dump 分区 0 1 2 里 以 offset 20 起，一共 dump 20条消息结束。


#### 5.通过 -info 只显示消息元信息。

##### 使用如下命令来显示当前消息的元信息，包括创建信息。

```sh
dump -broker 127.0.0.1:9092 -topic member_test -partition 1  -offset 40 -info 
```

![image.png](https://upload-images.jianshu.io/upload_images/6393906-19512cb18d222941.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`info` 会显示该条消息的分区、`offset`、创建时间。不会显示当前 `offset` 消息的详细内容。

##### 多次执行，改变 offset 的值，查询 offset 对应的创建时间

```sh
dump -broker 127.0.0.1:9092 -topic member_test -partition 1  -offset 20 -info 
```

通过几次这样的操作后，可以锁定一定时间所对应的 `offset` 为多少。

如果消息存在多个分区中，可以在上述命令中指定 `-partition` 参数来更进一步查询。

### FAQ

### 1.消息 dump 失败，报如下错误信息

![image.png](https://upload-images.jianshu.io/upload_images/6393906-e26fdf30772be59d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)

#### 主要原因：
- 1. 当前事件所属服务元数据信息无法得到。
- 2.服务能获取到，消息的元数据信息获取不到。


#### 举例说明：

`StockEvent` 属于 `StockService` 服务。所以这条消息的元数据信息需要从 StockService 中去拿。
第一，判断 `StockService` 的是否存在于当前zk，是否是启动状态。
第二，如果服务是存在元数据的话，判断event的元数据信息是否暴露出去了。

如何暴露？[dapeng-event-bus 详细指南系列 --- 生产者详解](https://www.jianshu.com/p/be1c7f94dcb5) 文章也有提到，我们再详细说明一遍。

#### IDL服务接口事件声明
>需要将事件元信息定义到 `xxxservice.thrift` 的元信息中。

- 接口可能会触发一个或多个事件
- 事件必须在触发接口进行声明，否则事件解码器不会生成

`== >user_service.thrift`

```thrfit
namespace java com.github.dapeng.user.service

include "user_domain.thrift"
include "events.thrift"

string register(user_domain.User user
(events="events.RegisteredEvent,events.ActivedEvent")
}
```
