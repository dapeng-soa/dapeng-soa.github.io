---
layout: post
title:  "微服务准实时分布式日志告警与分析系统"
date:   2019-02-19 14:00:00 +0800
author: Ever
categories: biz
---
>微服务时代，一个请求可能跨越好几个节点的不同服务，使得日志查看非常麻烦。
同时，系统出现问题也没法快速定位出现问题的节点、异常的上下文等。

本文旨在通过自研的分布式日志告警与分析系统， 通过kafka流式处理技术实现线上业务的准实时告警。
同时，结合es、钉钉等工具，构建成一整套完整的业务告警、问题定位、日志查询系统。

## 系统架构
![系统架构](https://github.com/dapeng-soa/documents/blob/master/images/dapeng-log/log-system.png?raw=true)

## 配置
[详细配置](https://www.jianshu.com/p/ce30c31111ca)

其中， [fluent-bit](https://github.com/dapeng-soa/fluent-bit)我们做了功能上的增强:

1.实现主从host的转发（forward）功能。原fluent-bit只
支持向一个host进行转发，进行修改后，添加了一个从host（hoststandby)，当主host宕掉以后会通过hoststandby建立连接进行转发，当主host恢复后再
重新通过主连接进行转发。

2.对input插件中的tail进行了改造：在没有配置DB的情况下，修改为默认从文件末尾进行读取；在配置了DB的情况下，对配置文件添加db_count字段，通
过该字段可以设置将文件偏移量offset写入DB的频率。原生将offset写入DB的策略为：读取一次缓存块就进行一次写入，在这种策略下，会引起高频率的IO操作。

>fluent-bit可以选择把日志文件的读指针(offset)写入到db中(通常是一个文件)
