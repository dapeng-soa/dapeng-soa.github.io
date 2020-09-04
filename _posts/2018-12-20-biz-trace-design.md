---
layout: post
title:  "服务监控之业务系统数据埋点设计"
date:   2018-12-20 14:00:00 +0800
author: Ever
categories: monitor
---
业务系统的一些数据指标，对于开发人员或者业务部门来说非常重要，如何上报指标并把数据展现出来，意义重大。

## 1. 背景
目前基于dapeng-soa框架的服务，内置有服务接口的流量(包括调用次数以及数据流量、成功失败次数)、耗时两个基本监控。

对于业务的一些定制监控，例如下面这个：
>支付通道请求响应时间:
支付通道接口请求延迟数据监控，
例如：
1分钟内的请求有多少次。
超过1分钟的请求有多少次。
超过2分钟的请求有多少次。
超过3分钟的请求有多少次。

我们需要提供一个通用的数据埋点接口，以满足这些定制业务的需求。

## 2. 接口的设计目标
1. 简单 - 接口简单易用
2. 高效 - 接口应高效，尽量使用批量上报
3. 可靠 - 尽可能不丢失数据，尤其是系统重启的时候
4. 数据落地到时序数据库influxdb，并通过grafana去展示数据

对每个监控项， 可配置两个图：
1. 柱状图， 用于查看数量
2. 饼图， 用于查看占比

![架构图](https://raw.githubusercontent.com/dapeng-soa/documents/master/images/others/traceApi.png)

## 3. 接口设计
接口重用CounterService， 并随counterService api包一起发布。
同时， 该接口在本地队列满之后，需要把队列清空，并把清空的数据记录到单独的日志文件。

日志格式:
[Trace-\${monitorId}],payload:${payload}

>日志系统可不处理该日志文件

### 3.1 接口设计
```
public class JTraceClient {
    private CounterServiceAsync counterClient = new CounterServiceAsyncClient();

   /**
    * 单点提交
    * 该数据有多个tag，多个value
    *  monitorId建议为:serviceName.bizName
    * 注意， 提交后只是放入本地一个队列，并没有保存至influxdb
    **/
  void trace(String monitorId, Map<String, String> tags, Map<String, Int> values)
   
   /**
    * 单点提交
    * 注意， 提交后只是放入本地一个队列，并没有保存至influxdb
    **/
  void tracePoint(DataPoint dataPoint)

   /**
    * 批量提交
    * 注意， 提交后只是放入本地一个队列，并没有保存至influxdb
    **/
  void tracePoints(List<DataPoint> dataPoints)

   /**
    * 刷新缓存队列
    * 把本地队列所有数据保存至influxdb。 一般通过jmx或者服务重启的时候使用。
    **/
  void flush()
}
```

对背景一节中订单服务支付通道监控，我们可根据监控需求， 把数据点按耗时分类(tag)，
```
costTime < 1minute : LT1M
costTime < 2minute : LT2M
costTime < 3minute : LT3M
costTime > 3minute : GT3M
```
该业务场景monitorId为：`orderService.paymentCost`

那么假设某个支付请求耗时8秒， 那么埋点方式为:
```java
  traceClient.trace("orderService.paymentCost",  Map("costTime"->"LT1M"), Map("times"->1));
```
>实际上， 就算这么简单的监控需求， 我们也应该尽量兼容多个需求。 例如支付渠道统计，支付成功率统计。这时候我们需要用如下方式埋点:
```
traceClient.trace("orderService.paymentCost",
                           Map("channel"->"ali", "costTime"->"LT1M", "respCode"->"succeed"),
                           Map("times"->1, "cost"->3500))
```

### 3.2 本地缓存策略
数据点在用户提交后，都存放在本地队列中。

队列建议采用有界队列，在提交的时候如果队列大小大于警戒值:`QUEUE_ALERM`（默认8000， 可配置）, 那么直接把当前队列的数据清空并发送出去。

接口会通过一个定时器线程定时刷新(每10秒刷新一次)本地队列到influxdb(通过调用counterService.submitPoints接口)。

伪代码如下:
```
val queueLock = new Integer(0);
var dataQueue = new LinkedList();
def trace(monitorId : String, tags : Map[String, String],  values: Map[String, Integer]) : boolean = {
   if (points.size >= QUEUE_ALERM) {
       flush()
   }

  val point = Point(tags::("monitorId"->monitorId), values, xx)
  spinLock
  dataQueue.add(point)
  resetSpinLock
}

def flush {
  spinLock
  if (dataQueue.size >= QUEUE_ALERM)  {
        val pointsCopy = dataQueue.copy
        dataQueue = new LinkedList()
  }
 resetSpinLock
 counterService.submitPoints(pointsCopy).execptionally ( ex => {
        log(pointsCopy)
    })
}

 //自旋锁
def spinLock() {
  while(!queueLock.compareAndSet(0, 1))
}

// 恢复自旋锁
def resetSpinLock() {
  queueLock.set(0)
}
```
>同样， 定时器在处理dataQueue的时候，也需要添加自旋锁

### 3.3 数据的可靠性报障
#### 3.3.1 异常数据留存
当数据刷新失败的时候，需要重新把数据放回队列(或者刷新成功后再清理队列)
见3.2
#### 3.3.2 优雅退出机制
定时器线程需注册JVM的ShutdownHook，当收到进程停止的信号的时候， 立马把本地缓存队列的数据刷新到influxdb。

>注意，常用的tag，例如serviceName,  nodeIp, nodePort, reqUri(针对前端， 例如Eywa或者dapengMesh)，可统一加上去。
```java
public class TraceTagProxy {
   /**
    * 通用的tag在此方法中设置。
    **/
  public Map<String, String> customTags() {
    return Map("serviceName"->SoaSystemProperties.SERVICE_NAME， “nodeIp”->SoaSystemProperties.HOST_IP);
  }
}
```

### 3.4 数据点结构

```
public class DataPoint{
        
            /**
            * 业务类型, 在时序数据库中也叫metric/measurement,
            * 相当于关系型数据库的数据表
            * 例如：
            * 流量数据:dapeng_node_flow
            * 调用统计、耗时、成功率:dapeng_service_process
            **/
            public String bizTag ;
          
            /**
            * tag表示数据的类别， 类似关系型数据库的索引。tag的类型只能是字符串， 
            * 在查询的时候， 只能针对tag去过滤数据。
           **/
            public java.util.Map<String, String> tags = new java.util.HashMap<>();

            /**
            * value 数据值，用于展示, 支持的类型floats，integers，strings，booleans
            **/
            public java.util.Map<String, Long> values = new java.util.HashMap<>();
          
            /**
            * 时间戳 事件发生时候的时间戳
            **/
            public long timestamp ;
          
            /**
            * influxdb的库， 默认为:dapengState
            **/
            public String database ;
```

## 4. 业务场景举例
### 4.1. 支付通道监控:

支付通道接口响应时间监控。例如：
```
  1分钟内的请求有多少次 => LT1M
  超过1分钟的请求有多少次 => LT2M
  超过2分钟的请求有多少次 => LT3M
  超过3分钟的请求有多少次 => GT3M
```

```
trace("orderService.paymentChannelCost", Map("costTime"->"LT1M"), Map("times"->1))
```

OR
如果还要监控成功失败的情况，那么支付通道接口请求失败数:
即调用接口返回失败的数量。每日请求失败的占比是多少，失败返回码有哪些 比如获取MAC值失败等。
```
trace("orderService.paymentChannelCost", 
     Map("channel"->"ali", "respCode"->"xxx", "costTime"->"LT1M"), 
     Map("times"->1, "cost"->25ms))
```

### 4.2. 线上app支付的回调次数：
支付宝回调次数，微信回调次数。 分别占比多少。 每次回调处理时间是多少。
```
trace("orderService.paymentChannelCallback", 
      Map("channel"->"ali"), 
      Map("times"->1, "cost"->25ms))
```

### 4.3. 每次支付所用的时间：
每次发起支付，到最后得到结果的时间 可分析出支付时间占比
```
  1-15秒支付成功的有多少笔 => LT15S
  16-30秒支付成功的有多少笔 => LT30S
  31-45秒支付成功的有多少笔 => LT45S
  45秒以上支付成功的有多少笔 => GT45S
  分别占比是多少？
```

```java
trace("orderService.paymentCost", Map("costTime"->"LT35S", "respCode"->"succeed"), Map("times"->1))
```




