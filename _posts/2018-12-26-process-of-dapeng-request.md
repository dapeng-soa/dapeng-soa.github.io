---
layout: post
title:  "一个dapeng请求的历程"
date:   2018-12-26 14:38:38 +0800
author: hui
categories: dapeng
---

目前可以通过多种方式来发起一个dapeng请求：
1. 通过javaClient或者scalaClient发起请求
2. 通过网关dapeng-mesh发起请求
3. 通过dapeng-cli发起请求

发起请求的方式也许不同，但是dapeng内部对其处理是殊途同归的，下文将会对一个dapeng请求在dapeng内部的历程进行一个简要的说明。



### 客户端

&ensp;&ensp;一个dapeng请求在客户端的历程可以通过下面的流程图来加以简单的说明：

![clientRequest](https://github.com/dapeng-soa/documents/blob/master/images/dapeng-request/clientRequest.jpg?raw=true)

&ensp;&ensp;在创建客户端时，首先会在本地查找相同服务名的客户端信息，如果本地不存在，则会新建客户端信息，并同时将zk上的信息同步到本地（包括运行实例信息、路由信息、配置信息和cookie规则信息），最后将客户端信息注册到本地。
<br>&ensp;&ensp;通过同步zk信息到本地，为下一步中getConnection提供了版本、路由、负载均衡等等信息。

1.getConnection :

当一个dapeng请求到来时，首先需要做的是对当前请求的服务版本号进行检查。并通过服务路由和负载均衡来选择服务实例，并建立连接。
    
2.filters:

通过dapeng的filter机制进行处理：
  - headerFilter：空的filter
  - shareFilters: 目前在客户端中该filters只包含logFilter，对与日志相关的信息进行处理，例如设置日志的级别
  - dispatchFilter: 在该filter中会将收到的请求封装成ByteBuf对象，并将通过构建的NettyClinet将其发送到服务端去处理。
    <br/>客户端将请求发送到服务端，dapeng采取了同步和异步两种方式来实现。
    <br/>在同步方式下，待服务端处理完成之后，该filter对返回的ByteBuf对象进行解析得到result。
    <br/>在异步方式下，该filter直接对服务端返回的CompletableFuture对象进行处理。

3.frameDecode:

粘包和断包处理


### 服务端

&ensp;&ensp;经过客户端处理后的请求，在服务端的历程可以使用一下流程图进行简单的说明：

![serverRequest](https://github.com/dapeng-soa/documents/blob/master/images/dapeng-request/serverRequest.jpg?raw=true)

1.decode：

服务端收到来自客户端的请求，首先需要做的是对该请求进行解码

2.freqControl：

当开启了限流设置时，判断该请求是否被限流

3.filters：

服务端也是通过dapeng的filter机制对请求进行处理：
  - headerFilter：空的filter
  - shareFilters: 目前在服务端中该filters包含logFilter和slowServiceCheckFiler，logFilter对与日志相关的信息进行处理，例如设置日志的级别，同时将容器的classLoad转换成业务的classLoad；slowServiceCheckFilter则对服务进行慢服务检测
  - dispatchFilter: 调用业务处理逻辑对请求进行处理
    <br/>服务端调用也业务端处理逻辑，dapeng也采取了同步和异步两种方式来实现。
    <br/>在同步方式下，待业务端处理完成之后，该filter对返回的结果进行处理。
    <br/>在异步方式下，该filter直接对业务端返回的CompletableFuture对象进行处理。

4.msgEncode:

经过一系列filter处理后，msgEncode对于返回的result进行编码，然后发送到客户端。