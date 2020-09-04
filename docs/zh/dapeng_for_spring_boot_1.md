---
layout: doc
title: 基于SpringBoot的Dapeng服务开发与原理
---

> 本文将介绍如何使用 Dapeng 结合SpringBoot运行，并通过 OpenFeign 调用 SpringCloud 服务。

## 1 Demo 

### 1.1 下载 dapeng-soa 工程
- 下载工程:   https://github.com/dapeng-soa/dapeng-soa
- 切换分支到 `dapeng_sc` 分支
- 编译工程
```
mvn clean compile package install -DskipTests
```

### 1.2 环境准备
dapeng 服务需要依赖 zookeeper 作为服务注册和发现中心,
springcloud 需要依赖 nacos 作为服务注册与发现中心。
我们通过 docker 的形式启动这两个组件:
- 启动 zookeeper
```
docker run -d -p 2181:2181 --name zookeeper zookeeper 
```
- 启动nacos
下载 `nacos-docker`工程:  http://file.hzways.com/nacos-docker.tgz
解压,进入目录,执行 `sh standalone.sh` 即可启动 nacos 服务。

### 1.3 启动 基于 nacos 作为服务发现的 springcloud 服务 
如下图, `HelloServerApplication` 为 sc 服务的入口类
![image.png](https://upload-images.jianshu.io/upload_images/6393906-16b91743d6d4cd53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

通过增加启动参数的形式，启动多个 `springcloud` 服务(主要目的是测试 `OpenFeign` 的负载均衡), 指定端口号和标示服务节点的 `machine`。
```
-Dmachine=server1 -Dserver.port=9003
-Dmachine=server2 -Dserver.port=9004
```

- 直接运行 `HelloServerApplication` 类，启动 sc 服务，如果要运行多个服务，可通过如下形式改变端口运行。

### 1.4 DapengSoa Springboot 工程
![image.png](https://upload-images.jianshu.io/upload_images/6393906-ba8ed758ab8991c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/440)

- 进入 `dapeng-springboot-demo/hello-springboot-provider` 目录，通过 maven 插件运行 dapeng 服务
![image.png](https://upload-images.jianshu.io/upload_images/6393906-4a28ecc77da837a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要配置如上图所示，具体启动参数参考如下：

```
compile com.github.dapeng-soa:dapeng-maven-plugin:2.2.0-SNAPSHOT:run2 -Dsoa.freq.limit.enable=false -Dsoa.transactional.enable=false -Dspringboot.main.class=com.github.dapeng.demo.DemoAnnotationApplication -Dsoa.apidoc.port=9876 -Dsoa.container.port=9071 -Dspring.config.location=/Users/maple/developer/today/dapeng/dapeng-soa/dapeng-springboot-project/dapeng-springboot-demo/hello-springboot-provider/src/main/resources/application.properties
```
#### 1.4.1 各参数说明:

- `springboot.main.class`
> 指定当前springboot应用入口类全限定名。
```
-Dspringboot.main.class=com.github.dapeng.demo.DemoAnnotationApplication 
```

- `soa.apidoc.port`
> 指定文档站点端口号
```
-Dsoa.apidoc.port=9876 
```

- `soa.container.port`
> 指定 dapeng 容器暴露端口

```
-Dsoa.container.port=9071
```

- `spring.config.location`
> 指定SpringBoot启动时的配置文件路径,这里要根据实际情况修改
```
-Dspring.config.location=/Users/maple/developer/today/dapeng/dapeng-soa/dapeng-springboot-project/dapeng-springboot-demo/hello-springboot-provider/src/main/resources/application.properties
```

### 1.5 访问文档站点
启动完成之后，访问文档站点，请求服务 `HelloService`的 `sayHello ` 方法并观察 sc 服务控制台，和返回结果信息。
![image.png](https://upload-images.jianshu.io/upload_images/6393906-395934884575e362.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





## 2 Dapeng for Springboot 思路

### 2.1 SpringBoot 注解扫描 Dapeng 服务逻辑

1.在 SpringBoot 启动类中增加 `@DapengComponentScan` 注解，其会扫描类上以 `@DapengService` 标注的实现类。

![image.png](https://upload-images.jianshu.io/upload_images/6393906-0fa63b5f0eabad38.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. `@DapengService` 标注的即为服务实现类
![image.png](https://upload-images.jianshu.io/upload_images/6393906-854fcadee90de23a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.Spring 在处理这个注解时，通过解析到 HelloServiceImpl 实现类时，会额外针对此服务生成一个 `SoaServiceDefinition`，它是通过 `Spring FactoryBean` 来做到的，具体代码如下:
![image.png](https://upload-images.jianshu.io/upload_images/6393906-1374b4352ad1858a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4.生成的 ServiceBeanProcessorFactory BeanDefinition 最后通过 getObject 方法最终得到的 Spring Bean 是 SoaServiceDefinition
![image.png](https://upload-images.jianshu.io/upload_images/6393906-5d6f4da11d1bb993.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2 Dapeng 容器与 SpringBoot 容器结合逻辑
1.新增 `SpringBootAppLoader`，区别于之前的 `SpringAppLoader`
![image.png](https://upload-images.jianshu.io/upload_images/6393906-9986c47cf3ff6e1e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


