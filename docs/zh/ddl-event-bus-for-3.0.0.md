---
layout: doc
title: eventbus3.0.0 新特性和使用手册
---

## 根据具体的业务意义发送消息到同一分区

1.需要修改事件生产者系统对应的事件存储表 `dp_common_event`

>  具体需要增加一个字段,这个字段存储这条消息的业务key，例如storeId 等，没有则为空

```sql
alter table dp_common_event add event_biz varchar(255) DEFAULT NULL COMMENT '事件biz key' after event_type
```



2.发送事件方式

- old 如果不需要根据biz 来发送事件(对事件发送分区和顺序性没有要求)

```java
//fireEvent,内容为具体事件内容。事务由spring管理
EventBus.fireEvent(RegisteredEvent(GenIdUtil.getEventId().toInt, response.id))
//fireEvent，内容为事件内容，事务由自己手动管理，需要传入 conn (Connection)   
EventBus.fireEventManually(ActivedEvent(GenIdUtil.getEventId().toInt, response.id),conn )
```

- new（希望同一biz的内容发往同一个分区）

```java
//fireEventOrdered,第一个参数是biz,根据具体业务需求填，第二个为具体事件内容。事务由spring管理
EventBus.fireEventOrdered("11827901", RegisteredEvent(GenIdUtil.getEventId().toInt, response.id))
//fireEventOrderedManually，前两参数同上,第三个参数是需要传入 conn (Connection)手动管理事务
EventBus.fireEventOrderedManually("11827901",RegisteredEvent(GenIdUtil.getEventId().toInt, response.id),conn)
```



## 消费者订阅方法增加元数据信息

1.ConsumerContext 消费者消息上下文信息

> eventbus 3.0.0 增加消费者元数据上下文 ConsumerContext，该类包含内容如下:

```java
//消息key，可能为消息唯一id(不使用biz业务作为key)，可能为业务key经过hash后得到的数字(使用biz作为key)
private final Long key;
//消息来自的 topic
private final String topic;
//消息offset
private final Long offset;
//消息来自的分区
private final Integer partition;
//消息产生的时间-时间戳
private final Long timestamp;
//格式化显示的消息时间
private final String timeFormat;
//消息时间type，一般为 CreateTime
private final String timestampType;
//消息的类型 -> 事件类型的全限定名
private final String eventType;
```



2.在订阅消息的方法前面加上 `ConsumerContext` 参数，框架会利用反射自动为其赋值。

- old

```scala
@KafkaListener(serializer = classOf[RegisteredEventSerializer])
def subscribeRegisteredEvent(event: RegisteredEvent): Unit = {
   println(s"收到消息 RegisteredEvent: $event")
}
```



- new

```scala
@KafkaListener(serializer = classOf[RegisteredEventSerializer])
def subscribeRegisteredEvent(context: ConsumerContext, event: RegisteredEvent): Unit = {
    println(s"消息元信息ConsumerContext: $context")
    //业务自己选择如何处理这个元数据信息
    println(s"收到消息内容: $event ")
}
```



## 自定义消息重试次数

> 目前消费者消费事件失败后，默认重试3次，每次间隔为2s,4s,8s,16,30s,30s，即最长30s

```java
//使用并发阻塞同步队列
private LinkedBlockingQueue<ConsumerRecord<KEY, VALUE>> retryMsgQueue = new LinkedBlockingQueue<>();

//新增线程池定时重试
private void beginRetryMessage() {
    executor.execute(() -> {
        while (true) {
            try {
                ConsumerRecord<KEY, VALUE> record = retryMsgQueue.take();
                    for (ENDPOINT endpoint : bizConsumers) {
                         //将每一条重试逻辑放入新的线程中          
                        executor.execute(() -> {
                            retryStrategy.execute(() -> {
                                dealMessage(endpoint, record)));
                            }	
                        }
                    }
                    logger.info("retry result {} \r\n", record);
                } catch (InterruptedException e) {
                    logger.error("InterruptedException error", e);
                }
            }
        });
}
```

2.如果需要对消息消费保证高可用，希望消息能够重试更多次数，可以进行如下定义

```scala
//增加maxAttempts和retryInterval参数
@KafkaConsumer(groupId = "subscribe", topic = "member_test", maxAttempts = 5, retryInterval = 2000)
@Transactional(rollbackFor = Array(classOf[Throwable]))
class DemoConsumer {
  
  @KafkaListener(serializer = classOf[RegisteredEventSerializer])
  def subscribeRegisteredEvent(context: ConsumerContext, event: RegisteredEvent): Unit = {
    println(s"收到消息 RegisteredEvent:$event, context: $context")
  }


  @KafkaListener(serializer = classOf[ActivedEventSerializer])
  def subscribeRegisteredEvent(context: ConsumerContext, event: ActivedEvent): Unit = {
    println(s"收到消息 ActivedEvent  ${event.id} ${event.userId} ")
  }

}
```

在 `@KafkaConsumer` 注解中增加 `maxAttempts` （最大重试次数） 和 `retryInterval` 重试间隔.



## 消息回溯重新消费

### 通过应用程序指定分区offset
> 此功能是备选策略 (兜底策略)，如果通过之前配置的重试策略等，仍然有大批量消息消费失败。可以通过手动置顶 groupId 、topic 、 partition、offset 等，将消费者组的消费者某个分区重置到之前已经消费的某个offset

1.定义 application.conf ，系统从哪儿 load application.conf ，通过下面的环境变量指定

```sh
msg.back.tracking.path = /data/application.conf
```

2. `application.conf` 内容范例，前缀请使用 eventbus consumer

```properties
#固定格式
eventbus {
  #固定格式,里面定义的是一个数组。
  consumer = [
    {
      groupId = "demo_subscribe1",
      topic = "member_test"
      partition = 0,
      offset = 48700
    },
    {
      groupId = "demo_subscribe1",
      topic = "member_test"
      partition = 1,
      offset = 130
    }
  ]
}
```

- `application.conf`  定义内容前缀默认为 eventbus consumer
- 需要回溯的消息配置如上，定义成为一个数组的形式即可。

- 此配置文件放在指定目录，并通过环境变量 `msg.back.tracking.path` 来指定配置文件位置。
- 系统和消费者再重启之后会读取该配置文件，而且只在第一次启动后读取。读取完毕，消费者启动之后，**该配置文件马上会被删除**。
- 请注意备份配置文件，该配置文件会在第一次读取之后，即被删除。(为避免多次重启，多次重新读取配置文件，导致消息一直回溯，造成事故而设计)

### 通过指定kafka consumer-group分区offset

- 关停所有消费者，到kafka bin目录下执行
- ./kafka-consumer-groups.sh --bootstrap-server ${broker-list} --group ${group_id} --reset-offsets --topic ${topic}:${partition1},${partitions2} --to-offset ${offset} --execute

- 例如将消费组为test-group主题为test1分区为0，1，2 offset重置为100000
- ./kafka-consumer-groups.sh --bootstrap-server 172.16.17.38:9092 --group test-group --reset-offsets --topic test1:0,1,2 --to-offset 100000 --execute

---

## 升级步骤

- 1.修改对应生产者系统的 `dp_common_event` 表，增加 `event_biz` 字段
- 2.生产者系统依赖 `eventbus 3.0` 的依赖包
- 3.生产者系统需要使用分区策略的消息使用 `fireEventOrderedManually` 或者 `fireEventOrdered` 来触发事件
- 4.部署上线， 完成。




