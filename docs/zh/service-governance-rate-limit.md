---
layout: doc
title: 服务限流
---
## 前言
服务限流的目的是通过对并发访问/请求进行限速或者一个时间窗口内的请求进行限速来保护系统，一旦达到限制速率则可以拒绝服务，或者等待，或者降级。实现服务限流的方法有很多，而dapeng实现了一种基于共享内存的性能优越的限流方式，具体的实现可以见下篇的“基于共享内存的服务限流”一章。本节主要讲解对服务限流的使用。

## 限流配置
dapeng的服务限流设计分别支持服务级别和方法级别的限流。对此，在每分钟、每小时和每天三个级别上进行了频数限制。具体的配置格式如下：

```
[rule1]
match_app = getName # 针对具体方法限流
rule_type = callerIp # 对每个请求端IP
min_interval = 60,600  # 每分钟请求数不多余600
mid_interval = 3600,10000 # 每小时请求数不超过1万
max_interval = 86400,80000 # 每天请求书不超过8万

[rule2]
match_app = * # 针对服务限流
rule_type = callerIp # 对每个请求端IP
min_interval = 60,600  # 每分钟请求数不多余600
mid_interval = 3600,10000 # 每小时请求数不超过1万
max_interval = 86400,80000 # 每天请求书不超过8万
```
上述分别对服务级别和方法级别给出了示例。下面对每个字段进行解释：

* match_app 可以设定服务级别，以“*”表示，或者设定具体的方法（methodName);

* rule_type 是限流的key类型，目前dapeng支持：
  * all  对这个服务限流
  * callerIp  按callerIp限流。
  * callerMid 按 callMid 限流。 由于callerMid 是字符串，实际按照其 hashCode 进行限流。（具有相同的hashCode的callerMid被归一处理）
  * userIp 按 userIp 进行限流
  * userId 按 userId 进行限流
  * 对每一个服务，可以配置多个rule，例如，可以按照 callerIp 进行限流，可以按照 callerMid 进行限流，也可以同时进行限流。
  
* min_interval, mid_interval, max_interval 三个字段表示对应match_app调用频率的统计周期和限制频率，以“,”分隔，前者表示统计周期，以秒表示，后者表示限制频率。上述示例中分别表示对每分钟、每小时、每天的调用频率进行统计。
* dapeng支持动态的更新限流规则。
* 当服务级别和方法级别规则同时存在时，优先匹配方法级别的规则, 且跟服务级别不叠加。
将与上述格式匹配的配置信息保存在ZK中对应的服务路径下：soa/freq/services/xxx.service即可生效。

## 设计以及实现
详见 [RateLimit](https://github.com/dapeng-soa/dapeng-soa/wiki/DapengFreqControl)