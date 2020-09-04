---
layout: doc
title: dapeng-soa参数列表
---

#开发模式下使用".",容器模式下使用"_"。如host.ip在容器模式下为host_ip

|序号|参数|说明|备注|
|-|-|-|-|
|1|soa.container.ip|容器IP|(2.1.1版本后改用host.ip)|
|2|soa.container.port|容器端口(默认9090)|
|3|soa.shutdown.timeout|服务优雅关闭最大超时时间，单位毫秒，默认为100000毫秒(100s)|
|4|soa.apidoc.port|文档站点端口(默认8080)，开发环境（插件模式(plugin)运行服务）有效|
|5|soa.bytebuf.allocator|Netty ByteBuf ALLOCATOR 对象的创建方式(默认pooled)|
|6|soa.subPool.size|subPool 连接数， 单个客户端跟单个服务节点之间的连接数， 默认是1|
|7|soa.zookeeper.host|注册中心 (默默127.0.0.1:2181)|
|8|soa.max.process.time|默认最大处理时间， 超过即认为是慢服务（默认3000）|
|9|slow.service.check.enable|慢服务检查开关(默认true)|
|10|soa.container.usethreadpool|是否使用业务线程池开关(默认true)|
|11|soa.freq.limit.enable|服务限流开关(默认false)|
|12|soa.freq.shm.data|限流数据共享内存文件（默认是/data/shm.data）|
|13|soa.monitor.enable|服务监控开关 (默认false)|
|14|soa.service.timeout|服务超时时间(默认0)|
|15|soa.core.pool.size|业务线程池大小(默认为服务器处理器数*2)|
|16|soa.max.read.buffer.size|请求缓冲区大小(默认5M)|
|17|soa.instance.weight|服务实例权重(默认100)|
|18|soa.transactional.enable|全局事务 开关(默认true)|
|19|soa.filter.excludes|排除不需要的Filter|
|20|soa.filter.includes|加载需要的Filter|
|21|soa.kafka.host|kafka host(默认127.0.0.1:9092)|
|22|soa.zookeeper.master.host|可指定主从竞选的zk (需要时配置IP:2181)|
|23|soa.zookeeper.fallback.host|是否使用 灰度 zk (需要时配置IP:2181)|
|24|soa.local.host.name|设置本地主机名称|
