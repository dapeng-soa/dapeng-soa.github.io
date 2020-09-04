---
layout: post
title:  "Dapeng框架入门指北"
date:   2018-11-08 09:12:38 +0800
author: struy
categories: dapeng
---
Dapeng是一个支持SOA架构的开发框架，提供微服务的定义、开发、部署、监控、文档、测试、运维的一个完整工具链。Dapeng 提供的特色包括：

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

> 本章就以Java编写dapeng服务作为切入点，来逐渐熟悉大鹏的各种特性！

环境要求：
- 操作系统：主流操作系统都支持，但我们希望操作系统对于docker支持比较友好
- JDK：JDK8+
- Maven：Maven3+

## 使用maven构建项目结构
如果使用Java编写Dapeng服务，我们建议使用Maven构建Dapeng项目,不仅能管理项目的开发、编译、测试、打包，还可以享受丰富插件带来的便捷,如dapeng-maven-plugin可以生成接口代码，快速启动服务进行测试,下面就介绍如何使用maven从零开始建立一个dapeng-hello项目，并进行在线测试。

示例构建环境:
- Java：java version "1.8.0_151"
- Maven：Apache Maven 3.5.2
- OS：macOS High Sierra 版本 10.13.5

建立工程：
工程应当包含两个Module
- api:服务接口定义
- service:服务实现及确切业务逻辑

目录结构如下
```
- dapeng-hello
    - hello-api
    - hello-service
```

## api工程初始化
> 在dapeng开发的所有服务中，所有的接口定义都来自于Apache Thrift软件框架,用于可扩展的跨语言服务开发,dapeng-maven-plugin包含了将thrift生成为java代码的功能，无需开发者手动编写接口代码,只需定义统一的IDL,另外我们还支持scala代码的生成,(后面章节会如何让使用scala来开发dapeng微服务).

工欲善其事，必先利其器。首先介绍dapeng-maven-plugin的使用：

在api工程pom.xml
文件中添加大鹏插件如下:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.github.dapeng</groupId>
            <artifactId>dapeng-maven-plugin</artifactId>
            <version>2.0.2</version>
            <executions>
                <execution>
                    <phase>generate-sources</phase>
                    <goals>
                        <goal>thriftGenerator</goal>
                    </goals>
                    <configuration>
                        <language>java</language>
                        <sourceFilePath>src/main/resources/thrifts/</sourceFilePath>
                        <targetFilePath>src/main/</targetFilePath>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```
各个参数的含义:
- phase ：插件将在maven的generate-sources阶段调用此插件
- goal ：thriftGenerator 将在指定阶段执行thrift生成源码的任务
- language ：指定将thrift生成为哪种源代码，现支持Java和Scala
- sourceFilePath：指定thrift定义的IDL存放在何处
- targetFilePath：指定生成后的代码存放于何处，建议存放在上文定义的路径

除了本插件之外，我们还需要添加dapeng-core的依赖，生成后的代码中的序列化器，注解等都依赖dapeng-core

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <dapeng.version>2.0.2</dapeng.version>
</properties>

<dependencies>
    <dependency>
        <groupId>com.github.dapeng</groupId>
        <artifactId>dapeng-core</artifactId>
        <version>${dapeng.version}</version>
    </dependency>
</dependencies>
```
> 注：项目要求是JDK1.8+,在此指定maven使用JDK1.8的版本进行构建

在resources中新建一个文件夹thrifts以存放thrift文件,正如在插件中定义的thrift路径

到此，api工程的初始化就已完成，下面来看看如何定义最简单的服务IDL


## 定义IDL
> 附录? -> thrift

上文已经初始化了完整的api项目结构，现在只需要在上文新建的thrifts文件夹中按照如下添加thrift文件就可以生成服务接口代码：


```
src/main/resources/thrifts/
    -- hello_domain.thrift
    -- hello_service.thrift
```


hello_domain.thrift
```
namespace java com.dapeng.example.hello.domain

/**
* hello
**/
struct Hello {
/**
* name
**/
    1: string name,

/**
* message
**/
    2: optional string message

}
```

hello_service.thrift

```
include "hello_domain.thrift"

namespace java com.dapeng.example.hello.domain

/**
* Hello-Service
**/
service HelloService {

/**
# say hello

**/
    string sayHello(1:string name),
/**
# say hello2

**/
    string sayHello2(1:hello_domain.Hello hello)

}(group="hello")
```
## 代码生成

确保已经配置好对应的maven插件并添加了thrift文件，执行如下命令即可生成代码

```
cd hello-api

mvn clean compile
```

生成过程如下：
> 一个完成的代码生成过程示例：

```
➜  hello-api mvn clean compile
[INFO] Scanning for projects...
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] Building hello-api 1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ hello-api ---
[INFO] Deleting /Users/struy/project/github/dapeng-hello/hello-api/target
[INFO] 
[INFO] --- dapeng-maven-plugin:2.0.2:thriftGenerator (default) @ hello-api ---
 sourceFilePath: /Users/struy/project/github/dapeng-hello/hello-api/src/main/resources/thrifts/
 targetFilePath: /Users/struy/project/github/dapeng-hello/hello-api/src/main/
scrooge:-gen java -all -in /Users/struy/project/github/dapeng-hello/hello-api/src/main/resources/thrifts/ -out /Users/struy/project/github/dapeng-hello/hello-api/src/main/
parse /Users/struy/project/github/dapeng-hello/hello-api/src/main/resources/thrifts/hello_service.thrift success
parse /Users/struy/project/github/dapeng-hello/hello-api/src/main/resources/thrifts/hello_domain.thrift success
parse /Users/struy/project/github/dapeng-hello/hello-api/src/main/resources/thrifts/hello_service.thrift success
parse /Users/struy/project/github/dapeng-hello/hello-api/src/main/resources/thrifts/hello_domain.thrift success
parse /Users/struy/project/github/dapeng-hello/hello-api/src/main/resources/thrifts/hello_service.thrift success
parse /Users/struy/project/github/dapeng-hello/hello-api/src/main/resources/thrifts/hello_domain.thrift success
=========================================================
生成struct:Hello.java
生成struct:Hello.java 完成
=========================================================
=========================================================
服务名称:HelloService
生成service:HelloService.java
生成service:HelloService.java 完成
生成AsyncService:HelloServiceAsync.java
生成AsyncService:HelloServiceAsync.java 完成
生成client:HelloServiceClient.java
生成client:HelloServiceClient.java 完成
生成AsyncClient:HelloServiceAsyncClient.java
生成AsyncClient:HelloServiceAsyncClient.java 完成
生成serializer
 生成Serializer: HelloSerializer..
 生成Serializer: HelloSerializer..完成
生成SuperCodec:HelloServiceSuperCodec.java
生成SupperCodec:HelloServiceSuperCodec.java 完成
生成Codec:HelloServiceCodec.java
生成Codec:HelloServiceCodec.java 完成
生成AsyncCodec:HelloServiceAsyncCodec.java
生成AsyncCodec:HelloServiceAsyncCodec.java 完成
生成metadata:com.dapeng.example.hello.service.HelloService.xml
检查xml格式=>com.dapeng.example.hello.service.HelloService.xml 无误
生成metadata:com.dapeng.example.hello.service.HelloService.xml 完成
==========================================================
生成耗时:187ms
生成状态:完成
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ hello-api ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 3 resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ hello-api ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 9 source files to /Users/struy/project/github/dapeng-hello/hello-api/target/classes
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.367 s
[INFO] Finished at: 2018-07-04T08:02:49+08:00
[INFO] Final Memory: 19M/335M
[INFO] ------------------------------------------------------------------------

```
生成过程会打印整个过程生成了哪些文件，当出现 BUILD SUCCESS 时即生成完毕。

## 实现第一个dapeng接口
api接口生成完毕之后,应当提供相应的实现，来实现具体的业务逻辑。如上IDL的定义,我们有两个sayHello接口，一个入参是字符串，一个入参是Hello结构体。现在我们将实现接口将入参数直接返回：

编写实现类如下：

```java
package com.dapeng.example.service;

import com.dapeng.example.hello.domain.Hello;
import com.dapeng.example.hello.service.HelloService;
import com.github.dapeng.core.SoaException;


/**
 * @author with dapeng
 */
 
public class HelloServiceImpl implements HelloService {
    @Override
    public String sayHello(String name) throws SoaException {
        return "hello : " + name;
    }

    @Override
    public String sayHello2(Hello hello) throws SoaException {
        return "hello : " + hello.name + "-> " + hello.message;
    }
}

```

## spring中注册dapeng服务

Dapeng的应用一般使用spring来组装向外界提供服务，如下配置是一个标准dapeng服务的配置：

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
## 单元测试
hello-service中添加测试依赖：

```
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>4.3.5.RELEASE</version>
    <scope>test</scope>
</dependency>
```
编写测试case：

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:META-INF/spring/services.xml"})
public class HelloTestCase {

    @Autowired
    HelloService helloService;

    @Test
    public void sayHelloTest() {
        try {
            String result = helloService.sayHello("Dapeng");
            System.out.println("result-->" + result);
            Assert.assertTrue(result.contains("Dapeng"));
        } catch (SoaException e) {
            System.out.println(e.getMsg());
        }
    }
}
```


## 开发模式
开发模式下，服务在本地启动服务容器提供服务，并会启动对应的服务在线文档站点提供本地测试。

1.parent(hello)项目下执行
```
mvn clean install
```
本地启动:默认端口9090
1. 插件简称启动模式需要配置~/.m2/settings.xml

```
<pluginGroups>
    <pluginGroup>com.github.dapeng</pluginGroup>
  </pluginGroups>
```

2. 直接启动  

```
# 第一种(插件简称启动,需要配置pluginGroups)
cd hello-service
mvn compile dapeng:run -Dsoa.remoting.mode=local

# 第二种
cd hello-service
mvn compile com.github.dapeng:dapeng-maven-plugin:2.0.0:run -Dsoa.remoting.mode=local
```
当前控制台Ctrl+C即可停止服务

完整示例：https://github.com/dapeng-soa/dapeng-hello

## 编写客户端进行测试
在这里我们新建一个Module名为hello-test-client(也可以新建一个project进行测试编写)，并参照如下编写客户端测试代码，使用客户端进行测试时，需要将服务先启动之后确保能够正常提供服务后再启动客户端进行测试！

在当前新建的Module对应pom添加api依赖：

```
<dependency>
    <groupId>com.dapeng.example</groupId>
    <artifactId>hello-api</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>

<dependency>
    <groupId>com.github.dapeng</groupId>
    <artifactId>dapeng-client-netty</artifactId>
    <version>${dapeng.version}</version>
</dependency>
```

编写测试代码：

```
public class HelloClientTest {
    public static void main(String[] args) throws SoaException {
        System.setProperty("soa.zookeeper.host", "127.0.0.1:2181");
        HelloServiceClient client = new HelloServiceClient();
        String res = client.sayHello("Dapeng");
        System.out.println("result-->" + res);
    }
}
```
启动hello-service服务，运行HelloClientTest.main即可测试，并打印返回结果：

```
result-->hello : Dapeng
```

## 使用在线文档站点进行测试
> 服务编写开发过程中，会存在频繁改动的情况，如果我们总是编写客户端进行测试不太灵活，dapeng特有内嵌的在线测试文档，当服务启动后会一并启动在线文档，将最新的服务改动在web页面进行展现，所有的结构体即入参返回和请求模版都一目了然：

在以上案例中，当服务启动完毕后打开浏览器http://localhost:8080 即可访问在线文档进行测试：

- 打开服务对应的文档测试站点地址默认进入如下视图
![](http://www.struy.top/18-7-8/80749924.jpg)
- 点击api页面会展示当前应用所包含的所有服务列表
![](http://www.struy.top/18-7-8/45419220.jpg)
- 服务详情文档，可以查看当前服务所有的接口，及接口描述，请求体结构字段详情等详细信息
![](http://www.struy.top/18-7-8/7818616.jpg)
- 点击右侧测试按钮即可进入接口快速测试，填写对应参数并点击提交测试按钮即可完成一次测试，并能预览当前请求json
![](http://www.struy.top/18-7-8/61450594.jpg)
- 当请求返回，会在返回数据中展示返回内容以json格式进行展示
![](http://www.struy.top/18-7-8/37458342.jpg)


## dapeng里的几个关键概念

- **服务端**：服务端是服务的提供者,启动时会暴露指定端口，并且会将当前服务信息(服务全限定名，主机ip，服务通讯端口，版本号)注册到注册中心，在dapeng开发中通常只需实现IDL定义后生成的服务接口即可
- **客户端**：客户端是服务调用者，客户端提供的可调用接口通常与服务端接口保持一一对应，当服务端正常提供服务时，客户端发起服务调用时，会通过服务信息在注册中心中找到对应的服务信息
- **注册中心**：Dapeng默认使用zookeeper作为服务注册中心，服务端主动注册服务信息，客户端发起调用时主动发现服务信息并完成一次网络调用
- **服务元数据(ServiceMetaData)**：服务元数据是在通过Thrift编写IDL生成接口代码的同时生成的xml文件，文件包含服务的所有描述元信息，包括接口，注释，结构体，枚举，注解等信息。客户端默认实现了getServiceMetaData方法来获取这些信息
- **Dapeng容器**：提供服务的初始化、启动、请求处理、服务监控、统一日志过滤以及服务整个生命周期的管理

## 一键生成dapeng服务工程
dapeng提供了基于.g8的工程模板, 可一键生成整个项目的代码. 目前暂时只支持Scala, 详见第二章:使用scala开发dapeng服务


## 使用docker打包Dapeng服务

> 自从容器化的迅速普及，使得微服务的优点进一步放大，因此Dapeng支持容器化也是必然的。dapeng提供基础的dapeng容器镜像，我们只需要编写简单的Dockerfile，并配置maven插件即可打包服务镜像，这里我们使用maven-antrun-plugin+docker-maven-plugin来打包dapeng服务镜像。(需要透露的是，当我们使用scala开发时提供了更加便利，无入侵的整套镜像打包方案)：


编写Dockerfile如下

docker/Dockerfile

```
FROM dapengsoa/dapeng-container:2.0.2 
MAINTAINER dapengsoa@gmail.com

ENV CONTAINER_HOME /dapeng-container
ENV PATH $CONTAINER_HOME:$PATH
ENV PROJECT_NAME hello-service

RUN chmod +x ${CONTAINER_HOME}/bin/*.sh
WORKDIR ${CONTAINER_HOME}/bin
COPY ${PROJECT_NAME} ${CONTAINER_HOME}/apps/${PROJECT_NAME}/

ENTRYPOINT ["./startup.sh"]

```
> 其中dapengsoa/dapeng-container:2.0.2 就是dapeng服务的基础镜像，其中不但包含了dapeng的核心启动程序和对应版本的类库，还包含我们jstack，jmx，dapeng-cli，
fluent-bit，skywalking-agent，等常用的工具。有了这些工具，动态操作Dapeng配置，命令行调用服务，日志收集，调用链分析都不在话下
插件配置:

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.3.2</version>
    <configuration>
        <outputDirectory>${project.build.directory}/${project.artifactId}/</outputDirectory>
    </configuration>
</plugin>
<plugin>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>1.8</version>
    <executions>
        <execution>
            <id>copy files</id>
            <phase>package</phase>
            <configuration>
                <!-- copy child's output files into target/docker -->
                <tasks>
                    <copy todir="${basedir}/docker/${project.artifactId}/">
                        <fileset dir="${project.build.directory}/${project.artifactId}"/>
                    </copy>
                </tasks>
            </configuration>
            <goals>
                <goal>run</goal>
            </goals>
        </execution>
        <execution>
            <id>clean up the docker folder</id>
            <phase>clean</phase>
            <configuration>
                <!-- delete folder under docker -->
                <tasks>
                    <delete dir="${basedir}/docker/${project.artifactId}"/>
                </tasks>
            </configuration>
            <goals>
                <goal>run</goal>
            </goals>
        </execution>
    </executions>
</plugin>
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.4.10</version>
    <configuration>
        <imageName>docker.dapeng.example/hello-service:1.0-SNAPSHOT</imageName>
        <dockerDirectory>${basedir}/docker</dockerDirectory>
    </configuration>
    <executions>
        <execution>
            <id>build-image</id>
            <phase>package</phase>
            <goals>
                <goal>build</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
打包docker镜像：

```
cd hello-service

mvn clean pacakge

```

## 使用docker来启动dapeng服务

```
docker run -p 8999:8999 --name="helloService" -e soa_container_port=8999 -e soa_container_ip=yourIp -e soa_service_port=8999 -e soa_service_ip=yourIp -e soa_zookeeper_host=yourIp:2181 -e LANG=zh_CN.UTF-8
```
上面一堆参数是不是头都大了，或者使用更加方便的docker-compose来启动服务

docker-compose.yml


```
version: '2'
services:
  helloZk:
    container_name: helloZk
    image: zookeeper:3.4.11
    ports:
      - 2181:2181
    labels:
      - project.source=public-image
      - project.owner=struy

  helloService:
    container_name: helloService
    image: docker.dapeng.example/hello-service:1.0-SNAPSHOT
    volumes:
      - "/data/logs/dapeng/hello-service:/dapeng-container/logs"
    environment:
      - soa_monitor_enable=false
      - soa_transactional_enable=false
      - soa_core_pool_size=100
      - LANG=zh_CN.UTF-8
      - TZ=CST-8
      - soa_container_port=${hello_service_port}
      - soa_container_ip=${hello_service_ip}
      - soa_zookeeper_host=${soa_zookeeper_host}
    ports:
      - "${hello_service_port}:${hello_service_port}"
    labels:
      - project.source=https://github.com/dapeng-soa/dapeng-hello.git
      - project.depends=helloZk
      - project.owner=struy
```
运行如下命令即可启动zookeeper和服务了
```
docker-compose up -d 
```
> docker容器模式下启动服务没有在线测试站点,如果需要测试可以编写客户端服务来进行测试

## 总结
以上是Dapeng服务一个完整开发流程，dapeng不仅有方便的代码生成，友好的开发模式和spring的无缝集成，还有独特的元数据支持，在线文档测试等特性，使得SOA服务开发变的简单和有效。
