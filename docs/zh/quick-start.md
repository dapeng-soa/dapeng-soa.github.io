---
layout: doc
title: 开发指南
---
## Dapeng是什么？

- Dapeng是一个支持SOA架构的开发框架，提供微服务的定义、开发、部署、监控、文档、测试、运维的一个完整工具链。Dapeng 提供的特色包括：
  - 开发友好
    - 基于 IDL 语言，支持复杂的数据结构，有良好的服务接口表达能力。
    - 提供开发有好的语言绑定。生成开发友好的Stub/Skelton代码（相比业内其他框架）。
    - 提供了基于 Case Class、immutble风格的Scala代码，支持函数式风格的业务开发。
    - maven/sbt插件支持，开发文档站点、WEB测试工具，支持REPL风格的服务开发，Edit-Save-Running，敏捷开发。
  - 测试友好
    - 基于IDL的 Mock Server 支持快速服务开发、联调。
    - 在线WEB测试。无需编写一行代码，即可使用WEB界面测试你的服务。
  - 面向高性能
    - 基于 Netty、NIO 底层技术，核心框架均高度调优，可支持单机25万QPS的消息调度能力。
    - 底层通讯协议基于高效的thrift压缩二进制协议，高效序列化，网络传输高效率。
    - Dapeng创新的 JSON Stream Process，单个服务器可以支持高到每秒 2万以上 个 JSON Rest请求调度。
  - 工具链完善，运维友好
    - 有强大的服务路由策略，可以支持各种灰度发布、A/B测试。
    - 多种负载均衡策略
    - 高效、灵活的服务限流支持
    - 全链路日志跟踪
    - 强大的服务监控能力
    - 强大的服务告警能力
    - cli 工具，支持对线上服务的各种操作（无需写代码即可设置内部参数、调用线上服务）
  - 开放API支持
  

目前提供Java和scala两种语言的开发支持，还支持生成代码模版，后续将根据需要适配PHP、Python、GO等语言。在秉承代码及文档的一致性理念基础上，支持服务的在线文档查看和测试，这得益于特有的元数据。
- 自定义的对象编解码序列化器，这相比其他的序列化性能上要强劲的多
- dapeng具有基于Streaming的高性能的Json序列化技术，使得通过http接入的rest请求处理在性能上具备跟原生客户端(java/scala)相当的能力。

环境要求：
- JDK：JDK8+
- Maven：Maven3+

## 快速开始

我们已经准备好一个完整的示例，只需要做简单修改即可开始:

```
git clone https://github.com/dapeng-soa/dapeng-hello.git

// 使用docker(如果有)启动一个zookeeper注册中心
docker pull zookeeper
docker run --name zookeeper -p 2181:2181 -d zookeeper

cd dapeng-hello/

mvn clean install

cd hello-service
mvn compile com.github.dapeng-soa:dapeng-maven-plugin:2.1.1:run -Dsoa.freq.limit.enable=false -Dsoa.transactional.enable=false

或者使用更简单的快捷方式，
mvn compile dapeng:run -Dsoa.freq.limit.enable=false -Dsoa.transactional.enable=false

注意，该方式需先在maven的配置文件中(macOs中默认为止为~/.m2/setting.xml)，添加PluginGroup:
<pluginGroups>
    <pluginGroup>com.github.dapeng-soa</pluginGroup>
</pluginGroups>


// 打开浏览器访问：http://127.0.0.1:8080 即可访问示例站点

// 当前控制台Ctrl+C即可停止服务
```
注:如果没有安装docker，可以参见[zookeeper官网](http://zookeeper.apache.org/)在本地启动zookeeper

上面就已经运行了一个最基础的demo工程，下面我们来了解一下整个项目结构和开发流程

>如果服务中启动报错，请参考FAQ: https://dapeng-soa.github.io/faqs/

## 示例项目结构
以下是项目的目录结构，整个项目分为3个模块，其中`hello-test-clinet`不是必然需要的模块。
- api:服务的IDL定义之处，存放于thrifts目录，在执行简单指令后可以生成相关代码
- service:服务的实现代码已经服务注册配置等信息

```
|--dapeng-hello
|  |--hello-api                 dapeng服务api，插件生成代码
|  |  |--src
|  |  |  |--main
|  |  |  |  |--java
|  |  |  |  |--resources
|  |  |  |  |  |--thrifts       IDL文件存放
|  |  |--pom.xml
|  |--hello-service             dapeng服务实现
|  |  |--src
|  |  |  |--main
|  |  |  |  |--java
|  |  |  |  |--resources
|  |  |  |  |  |--META-INF
|  |  |  |  |  |  |--spring
|  |  |  |  |  |  |  |--services.xml
|  |  |--pom.xml
|  |  |--logs
|  |  |--docker
|  |  |  |--Dockerfile          
|  |--.gitignore
|  |--pom.xml
|  |--hello-test-clinet         客户端测试
------------------------------------
```

## 生成接口代码
如果api中的thrift文件变更，接口变化。可以使用以下命令快速生成代码

```
cd hello-api
mvn clean compile
```

生成过程如下：

```
hello-api> mvn clean compile
[INFO] Scanning for projects...
...
...
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.367 s
[INFO] Finished at: 2018-07-04T08:02:49+08:00
[INFO] Final Memory: 19M/335M
[INFO] ------------------------------------------------------------------------
```
生成过程会打印整个过程生成了哪些文件，当出现 `BUILD SUCCESS` 时即生成完毕。


## Spring中注册dapeng服务

dapeng的应用一般使用spring来组装bean并向外界提供服务，如下配置是一个标准dapeng服务的配置模版,可在示例项目的`services.xml`文件中找如下配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:soa="http://soa-springtag.dapeng.com/schema/service"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://soa-springtag.dapeng.com/schema/service
    http://soa-springtag.dapeng.com/schema/service/service.xsd">

    <bean id="helloService" class="com.dapeng.example.service.HelloServiceImpl"/>
    <soa:service ref="helloService"/>
</beans>

```

## 使用在线文档站点进行测试
> 服务编写开发过程中，会存在频繁改动的情况，如果我们总是编写客户端进行测试不太灵活，dapeng特有内嵌的在线测试文档，当服务启动后会一并启动在线文档，将最新的服务改动在web页面进行展现，所有的结构体即入参返回和请求模版都一目了然：

在以上案例中，当服务启动完毕后打开浏览器http://localhost:8080 即可访问在线文档进行测试：
- 点击api页面会展示当前应用所包含的所有服务列表
![](http://www.struy.top/18-7-8/45419220.jpg)
- 服务详情文档，可以查看当前服务所有的接口，及接口描述，请求体结构字段详情等详细信息
![](http://www.struy.top/18-7-8/7818616.jpg)
- 点击右侧测试按钮即可进入接口快速测试，填写对应参数并点击提交测试按钮即可完成一次测试，并能预览当前请求json
![](http://www.struy.top/18-7-8/61450594.jpg)
- 当请求返回，会在返回数据中展示返回内容以json格式进行展示
![](http://www.struy.top/18-7-8/37458342.jpg)

## 如何使用docker运行dapeng服务：

```bash
// 打包镜像
cd hello-service
mvn clean pacakge

// 使用docker-compose启动
docker-compose up -d helloZk
docker-compose up -d helloService
```
>注意：项目编写了docker-compose配置文件，包含zookeeper服务和示例代码的服务(需要提前打包)，请修改配置文件中的`#host_ip`为你的主机ip

