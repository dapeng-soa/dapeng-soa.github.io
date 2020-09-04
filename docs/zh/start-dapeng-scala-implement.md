---
layout: doc
title: 快速开始:使用Scala开始
---

### 使用Scala 开发测试基于大鹏框架的服务

#### 1. 环境准备
- [x] java8 或以上版本
- [x] scala 2.12.2 或以上版本
- [x] sbt 环境 (sbt版本建议使用1.1.6 或以上)
        由于sbt国内环境不理想，建议使用代理 https://github.com/Centaur/repox
- [ ] docker环境(可选)

<hr/>

#### 2. 创建dapeng scala项目
> dapeng提供了一个scala版本项目的gitter8模板项目，只需要一个指令，即可创建demo项目

```bash
> sbt new dapeng-soa/dapeng-soa-v2.g8

[info] Set current project to github (in build file:/****/dev/github/)
[info] Set current project to github (in build file:/***/dev/github/)

this is a template to genreate dapeng api service

name [hello]:
version [0.1-SNAPSHOT]:
scalaVersion [2.12.2]:
organization [com.github.dapeng-soa]:
resources [resources]:
scala [scala]:
api [hello-api]:
service [hello-service]:
servicePackage [com.github.dapeng]:
dapengVersion [2.1.0]:

Template applied in ./hello
```
> demo项目，一路回车键即可，后续根据需求可以自行修改对应的值

#### 3. 编译并运行项目
> 注意，大鹏框架容器依赖了zookeeper, 所以在启动dapeng服务前，请启动zookeeper服务
`可以使用docker启动zookeeper, 也可以下载zookeeper直接启动，这里不做过多赘述`

```
> cd yourTemplateProject   (本例项目名为 hello)
> sbt runContainer

......
------------------------- Initialize serviceCache......
--------------------Container: com.github.dapeng.impl.container.DapengContainer@1e8de580
--------------------Applications: [com.github.dapeng.impl.container.DapengApplication@68575702]
--------------------Filters: [com.github.dapeng.impl.filters.LogFilter@7de8c951, com.github.dapeng.impl.filters.slow.service.SlowServiceCheckFilter@656e2ba1]
......
api-doc server started at port: 8192
```
> 上述日志最后一行可以看到服务已经启动，端口为: 8192


#### 4. 测试项目

##### 4.1 使用客户端测试
> gitter8模板提供了一个现成的单元测试类. 直接启动main方法即可测试服务

```scala
package com.github.dapeng.test

import com.github.dapeng.hello.scala.HelloServiceClient

object HelloServiceImplTest {

  def main(args: Array[String]): Unit = {
    val response = new HelloServiceClient().sayHello("hello world! ")
    println(response)
  }
}
```

```
//运行结果
...
11-20 17:39:40 026 main DEBUG [866013b77415db71] - Loaded default ResourceLeakDetector: io.netty.util.ResourceLeakDetector@36ebc363
11-20 17:39:40 049 main DEBUG [866013b77415db71] - NettyClient::send, timeout:1000, seqId:0,  to: /172.16.19.183:9095
11-20 17:39:40 227 main INFO [866013b77415db71] - LogFilter::onExit,response[seqId:0, respCode:0000, server: 172.16.19.183:9095]:service[com.github.dapeng.hello.service.HelloService]:version[1.0.0]:method[sayHello] cost[total:220, calleeTime1:156, calleeTime2:156, calleeIp: 172.16.19.183
has received msg:  hello world! 
```

##### 4.2 使用框架提供的站点测试
dapeng框架提供配套的在线测试站点，我们可以可视化的测试我们的接口，如：
> 打开 http://localhost:8192/api/test/HelloService/1.0.0/sayHello.htm

![helloTest](https://raw.githubusercontent.com/dapeng-soa/documents/master/images/dapeng-thrift/apiDocHelloTest.jpg)

## 附:常见的一些坑
#### windows下的编码问题
在执行sbt命令时，如果是windows，会有可恨的编码问题，建议按照如下方式解决：
1. 使用IDEA打开项目
2. 执行sbt命令使用 Run/Debug Configuations内的sbt Task来运行sbt命令，并在配置时VM参数上加-Dfile.encoding=utf8 如下图：
![](http://www.struy.top/18-8-26/25456507.jpg)
