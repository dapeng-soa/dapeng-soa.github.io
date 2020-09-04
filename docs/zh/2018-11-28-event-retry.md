---
layout: doc
title:  "dapeng-event-bus 详细指南系列 --- 消费者消息重试机制"
---

上一篇我们介绍了基于 `eventbus` 如何开发消费者来监听事件。本章我们将系统研究消息的一致性和重试原理，并作出实践。


![消息重试.jpg](https://upload-images.jianshu.io/upload_images/6393906-32f1e87f0275561a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 消费者重试机制
> 消费者处理消息时，如果业务抛出异常的情况下，消息是会进行重试的。目前默认重试次数为`3`次，初始间隔时间为 `2000ms` 即 `2s` ，每次重试会以 `2` 的倍速增大重试间隔，直到重试结束。并且每次重试时间间隔会逐渐增大。

### 消息重试前提

- 1.监听方法需要主动抛出异常，如果没有异常抛出，那么**`eventbus`** 组件不会对消息进行重试。
- 2.监听方法如果抛出 **`SoaException`**，组件会认为这是单纯的业务 **`assert`** 异常，消息不会进行重试。

因此，在业务上，我们需要明确什么情况下消息会重试。如果业务在消费消息之前会做前置检查，此时前置检查不通过抛 **`SoaException`**，那么消息重试也没有意义，无论怎么重试，都是报一样的错。

---

### 消息重试原理
> 消息重试机制底层使用的是 `Spring Retriy`,有兴趣可以学习，相关 [学习资料](https://blog.csdn.net/u011116672/article/details/77823867)

下面是业务消息订阅消费使用的重试策略。默认使用 `DefaultRetryStrategy`，重试次数为 `3` 次。

```java
public class DefaultRetryStrategy extends RetryStrategy {
   /**
     * 重试次数...
     */
    private final int maxAttempts;
    /**
     * 重试间隔时间
     */
    private final int retryInterval;

    public DefaultRetryStrategy(int maxAttempts, int retryInterval) {
        this.maxAttempts = maxAttempts;
        this.retryInterval = retryInterval;
    }
    /**
     * 默认 SimpleRetryPolicy 策略
     * maxAttempts         最多重试次数
     * retryableExceptions 定义触发哪些异常进行重试
     */
    @Override
    protected RetryPolicy createRetryPolicy() {
        SimpleRetryPolicy simpleRetryPolicy = new SimpleRetryPolicy(maxAttempts, Collections.singletonMap(Exception.class, true));
        return simpleRetryPolicy;
    }

    /**
     * 指数退避策略，需设置参数sleeper、initialInterval、maxInterval和multiplier，
     * <p>
     * initialInterval 指定初始休眠时间，默认100毫秒，
     * multiplier      指定乘数，即下一次休眠时间为当前休眠时间*multiplier
     * maxInterval      最大重试间隔为 30s
     * <p>
     * 目标方法处理失败，马上重试，第二次会等待 initialInterval， 第三次等待  initialInterval * multiplier
     * 目前重试间隔 0s 4s 16s 30
     */

    @Override
    protected BackOffPolicy createBackOffPolicy() {
        ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(retryInterval);
        backOffPolicy.setMultiplier(2);
        return backOffPolicy;
    }
}
```

上述 `maxAttempts` 和 `retryInterval` 的默认值分别为 `3` 和 `2000`(ms)。
`setMultiplier` 消息每一次间隔增加倍数 为 `2`

根据上述类的配置，我们可以得到结论：

- 1.第一次重试与消息失败后立刻重试，几乎是马上就进行重试了。
- 2.消息第二次重试会间隔2s，第三次就是 2s * 2 = 4s
- 3.由于spring retry 最大的重试间隔时间为30s，所以间隔不会大于30s

由上面 `3` 个结论我们可以得出消息重试情况如下，一旦消息重试成功，马上可以`break` 出去，重试结束。


![重试次数.png](https://upload-images.jianshu.io/upload_images/6393906-8e65f34177b4f38d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


---


### 自定义重试次数和重试策略
> 如果上述 `DefaultRetryStrategy` 类无法满足业务需求，比如某些需求对事件消息有严格的要求，需要重试很多次，那么 `eventbus` 类库需要提供这个功能，可以由用户自己设定重试次数和重试间隔时间。
我们可以在消费者类注解 `@KafkaConsumer` 中显示的注明重试次数和间隔时间。如下:

```scala
@KafkaConsumer(groupId = "demo_subscribe", topic = "member_test", maxAttempts = 5, retryInterval = 6000)
@Transactional(rollbackFor = Array(classOf[Throwable]))
class DemoConsumer {
    @KafkaListener(serializer = classOf[RegisteredEventSerializer])
    def subscribeRegisteredEvent(event: RegisteredEvent): Unit = {
       println(s"收到消息 RegisteredEvent  ${event.id} ${event.userId}")
    }

    @KafkaListener(serializer = classOf[ActivedEventSerializer])
    def subscribeRegisteredEvent(context: ConsumerContext, event: ActivedEvent): Unit = {
        println(s"收到消息 ActivedEvent  ${event.id} ${event.userId} ")
    }
}
```
上面例子中，我们指定当前消费者类下面的所有订阅方法的重试次数次数为 `5`次，重试间隔时间为 `6s`，一次递增，最大重试间隔为 `30s`。


### 消费者重试机制总结和划重点
- 消息默认就开启了重试机制，默认 `3` 次，初始间隔 `2s`，然后以 `2` 的倍速增加，最大间隔 `30s`。
- 消息重试的前提是，业务需要抛出除 `SoaException` 以外的异常。
- 用户可以通过在消费者类上自己定义重试次数来覆盖默认重试次数，**建议修改重试次数，不要修改重试间隔时间**。




### END & 示例项目 Samples

<br/>

#### [生产者demo](https://github.com/leihuazhe/producer-demo)

<br/>

#### [消费订阅者demo](https://github.com/leihuazhe/consumer-demo)

<br/>

#### [dapeng eventBus](https://github.com/dapeng-soa/dapeng-event-bus)
