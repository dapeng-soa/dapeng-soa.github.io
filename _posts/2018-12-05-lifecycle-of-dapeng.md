---
layout: post
title:  "Dapeng-soa 服务的生命周期"
date:   2018-12-05 21:36:30 +0800
author: Ever
categories: dapeng
---
随着搭载于dapeng框架之上的业务系统一个个上线并趋于稳定，不少开发同事们在抓虫子之余，也萌生了一些“不满”：用了dapeng好久了，但是它就像一个黑盒子，我们知之甚少，能否系统的介绍一下啊。

确实，dapeng框架提供了丰富的脚手架以及详尽的开发指南(详见[dapeng官网](https://dapeng-soa.github.io)), 极易上手。但了解一下dapeng的内部机制，有利于开阔眼界，并在开发中充分利用各个特性，对于提高服务的质量也很有好处。
![dapeng-soa architechure](https://raw.githubusercontent.com/dapeng-soa/documents/master/images/dapeng-architecture.png)
## 1. 概述
本文着眼于分析一个服务从启动到结束的整个过程。
>后续会分专题陆续介绍dapeng框架的各个特性，敬请留意。

## 2. dapeng 容器目录结构

```shell
dapeng-container
├── apps                           # 1. business services
│   ├── order_service
│   └── stock_service
├── bin
│   ├── dapeng-bootstrap.jar                      # 2. dapeng bootstrap module
│   ├── lib                                       # 3. dapeng container jars(including dependency)
│   ├── shutdown.sh                               # 4. startup/shutdown shell
│   └── startup.sh
├── conf                                          # 5. configuration files for container
│   └── logback.xml
├── lib                                           # 6. core lib
│   ├── dapeng-core-2.1.1.jar
│   └── logback-*.jar
├── logs                                          # 7. directory where log files located
└── plugin                                        # 8. plugin directory
```

1. 业务服务，该目录由dapeng的`ApplicationClassLoader`加载， 可在一个dapeng容器中放多个服务，但是建议一个容器一个服务
2. dapeng bootstrap模块，容器的启动入口，负责生成其它的定制classLoader并启动容器。
3. dapeng container jar目录，容器的实现类及其三方依赖，通过`ContainerClassLoader`加载
4. 启动以及停止脚本
5. 容器的配置文件
6. 核心接口库以及日志，由`CoreClassLoader`负责加载，可用于服务端以及客户端
7. 日志目录
8. 插件目录，由`PluginClassLoader`加载

**注意**, 不同的目录由不同的classLoader加载。常见的错误是：apps路径下的服务有个对象(通过`ApplicationClassLoader`加载)，假设叫a，其类型为A，它是单例对象(通过`A.getInstance()`获得）。但是在bin路径下(也就是通过`ContainerClassLoader`加载), 通过`A.getInstance()`得到的a', 就跟a是完全不同的对象了。

## 3. zk节点结构

```shell
/soa/
├── runtime/service/                                   # 1. runtime info
│   ├── com.today.soa.idgen.service.IDService/
│   │   ├── 192.168.10.130:9081:1.0.0:0000000181        # 2. node info
│   │   ├── 192.168.20.101:9081:1.0.0:0000000179        # 3. master node
│   │   └── 192.168.10.138:9081:1.0.0:0000000183
│   ├── com.today.api.order.service.OrderService2/ 
│   │   ├── 192.168.10.133:9099:1.0.0:0000000368        
│   │   ├── 192.168.10.126:9099:1.0.0:0000000372       
│   │   ├── 192.168.20.102:9099:1.0.0:0000000367      
│   │   └── 192.168.10.131:9099:1.0.0:0000000366        # master node
│   └── ...                                             # 4. other services
└── config/
    ├── services/                                       # 5. config info
    │   ├── com.today.soa.idgen.service.IDService
    │   └── com.today.api.order.service.OrderService2
    ├── freq/                                           # 6. rate limit rules
    ├── routes/                                         # 7. router rules
    └── cookies/                                        # 8. cookie rules
```
### 3.1 服务运行时信息 `/soa/runtime/service/`
服务运行时信息由服务在启动的时候注册到zk的`/soa/runtime/service/${serviceName}`路径上，其表现形式为挂靠在该路径下的**临时节点**，格式为：`ip:port:version:seq`, seq由zookeeper自动生成，且能保证在同目录下唯一并单调递增。

**其中，dapeng容器根据本容器中服务运行时信息的seq，判断是否为master主节点(seq最低者为master)**

>一个服务对应一个临时节点。 我们可通过该路径得知目前服务集群中该服务的节点数量

### 3.2 服务配置信息 `/soa/config/`
服务配置信息存放了dapeng-soa所需要的一些配置项，可通过[命令行工具](https://github.com/dapeng-soa/dapeng-cli)或者[配置中心](https://github.com/dapeng-soa/dapeng-config-server)进行管理。

服务的配置信息当前有四种，分布在`/soa/config/`不同的子目录下。它们的结构都类似，都是把服务名字作为节点挂在配置子目录下，然后配置信息作为节点的内容。

#### 3.2.1 普通配置信息 `/soa/config/services/`
普通配置信息包括负载均衡策略、服务超时等信息。

它包含两个层次：
1. 全局性配置： 直接写进`/soa/config/services/`节点
```
timeout/100ms;loadbalance/LeastActive;
```
2. 服务私有配置： 写到具体的服务节点上， 例如`/soa/config/services/com.today.api.order.service.OrderService2`
```
timeout/100ms,createOrder:500ms,createOrderPayment:60000ms;loadbalance/LeastActive,createOrder:Random,createOrderPayment:RoundRobin;
```

#### 3.2.2 限流配置信息 `/soa/config/freq/`
dapeng 的流控做在服务端，所以**该节点只对服务端有效**。

限流信息直接写在具体的服务节点上，例如如下订单的限流配置写在`/soa/config/freq/com.today.api.order.service.OrderService2`上
```
[rule1]
match_app = listOrder # 针对具体方法限流
rule_type = callerIp # 对每个请求端IP
min_interval = 60,5  # 每分钟请求数不超过5次
mid_interval = 3600,100 # 每小时请求数不超过100次
max_interval = 86400,200 # 每天请求数不超过200次

[rule2]
match_app = * # 针对订单服务限流
rule_type = callerIp # 对每个请求端IP
min_interval = 60,600  # 每分钟请求数不超过600
mid_interval = 3600,10000 # 每小时请求数不超过1万
max_interval = 86400,80000 # 每天请求数不超过8万
```
>详见[dapeng-soa限流文档](https://github.com/dapeng-soa/dapeng-soa/wiki/DapengFreqControl)

#### 3.2.3 路由配置信息 `/soa/config/routes/`
服务路由信息也是直接写在具体的服务节点上，例如下面订单的路由配置写在`/soa/config/routes/com.today.api.order.service.OrderService2`上
```
method match r"create.*" => ip"192.168.10.0/24"
cookie_storeId match %"10n+1..6" => ip"192.168.20.128"
```
>详见[dapeng-soa路由文档](https://github.com/dapeng-soa/dapeng-soa/wiki/Dapeng-Service-Route%EF%BC%88%E6%9C%8D%E5%8A%A1%E8%B7%AF%E7%94%B1%E6%96%B9%E6%A1%88%EF%BC%89)

#### 3.2.4 cookie 规则信息 `/soa/config/cookies/`
cookie规则信息也是直接写在具体的服务节点上，例如针对来自dapengCli的手工对订单接口的调用，我们为这些调用打开TRACE功能，我们把规则配置到`/soa/config/cookies/com.today.api.order.service.OrderService2`:
```
callerIp match ip"192.168.20.200" => c"thread-log-level#TRACE"
```

## 4. 容器的生命周期
容器提供了一个LifeCycleAware接口以及若干事件，在事件发生的时候会触发相应的业务逻辑。

业务可通过实现该接口， 做一些初始化以及清理的动作。
> 例如某服务的业务，只在主节点启动一个工作线程。那么它就可以监听`MASTER_CHANGE`事件。当主节点发生变更的时候，就启动或者停止工作线程。

```java
public enum LifeCycleEventEnum {
    /**
        * dapeng 容器启动
        */
    START,
    PAUSE,
    MASTER_CHANGE,
    CONFIG_CHANGE,
    STOP
}
```

```java
/**
 * 提供给业务的lifecycle接口，四种状态
 *
 * @author hui
 * @date 2018/7/26 11:21
 */
public interface LifeCycleAware {

    /**
     * 容器启动时回调方法
     */
    void onStart(LifeCycleEvent event);

    /**
     * 容器暂停时回调方法
     */
    default void onPause(LifeCycleEvent event) {
    }

    /**
     * 容器内某服务master状态改变时回调方法
     * 业务实现方可自行判断具体的服务是否是master, 从而执行相应的逻辑
     */
    default void onMasterChange(LifeCycleEvent event) {
    }

    /**
     * 容器关闭
     */
    void onStop(LifeCycleEvent event);

    /**
     * 配置变化
     */
    default void onConfigChange(LifeCycleEvent event) {
    }
}
```

## 5. 容器的启动过程
容器启动需要协调各个插件的顺序，避免在服务还没准备好的情况下，客户端请求就涌进来。

通过脚本`startup.sh`启动容器: `java -server $JAVA_OPTS -cp ./dapeng-bootstrap.jar com.github.dapeng.bootstrap.Bootstrap`
![dapeng-soa bootstrap](https://raw.githubusercontent.com/dapeng-soa/documents/master/images/dapeng-bootstrap.png)


- Bootstrap创建三个`classLoader`, 分别是`CoreClassLoader`(负责加载lib目录的类)、`ContainerClassLoader`(负责加载bin/lib目录的类)以及`ApplicationClassLoader`(负责加载apps目录下的类)。
- Bootstrap通过`ContainerClassLoader`加载`ContainerFactory`并调用其`getContainer`方法， 获得`DapengContainer`实例。
- `DapengContainer`创建五个Plugin，并依次调用其`start`方法:
    - 启动`NettyPlugin`， 打开服务监听端口(例如9090)
    - 启动`ZkPlugin`, 跟注册中心`zookeeper`建立连接。
    - 启动`SpringPlugin`， 
        - 通过`ApplicationClassLoader`加载服务(一个服务表现为一个`Application对`象)
        - 对每个服务(假设为OrderService)，通过`ZkPlugin`把服务信息注册到`/soa/runtime/service/com.today.api.order.service.OrderService2`目录里，并启动一个对该路径的zk `watcher`(主要用于跟踪服务集群中master节点的变化, 当发生master变换时，需触发`MASTER_CHANGE`事件。
    一旦服务完成注册，嗷嗷待哺的客户端就会如潮水般涌进该服务节点。

   - 启动`SchedulePlugin`, 定时任务就绪
   - 启动`JmxPlugin`, `Jmx` 端口就绪
   - 如果是开发模式(默认), 那么启动`ApiDocPlugin`, 内置的文档站点可访问。该插件在生产环境下不会启动
- 触发容器`START`事件
- 加载服务端`Filter`(详情下回分解)。
- 最后，注册Jvm进程的`ShutdownHook`。 


至此，容器启动完毕， 服务生命周期开始。

## 6. 容器的优雅关闭过程
容器的关闭过程，同样需要协调插件的关闭顺序，确保进来的请求尽量处理完毕后再关闭容器，避免对业务产生影响。

为此， `dapeng` 容器会维护一个请求计数器`requestCounter`，计数值是当前容器内尚未处理完的请求数目。

- 启动脚本`startup.sh`会监听`kill`信号。收到`kill`信号后，脚本转发该信号到容器Jvm进程。
- 容器的`ShutdownHook`给触发,依次执行:
    - `ZkPlugin.stop`方法，断开跟`zookeeper`的连接，从而把本节点服务信息从`/soa/runtime/services/${serviceName}`上摘除，进而新的请求就不会再路由到本节点。
    - 休眠若干次，直到`requestCounter`数目为0或者超时。
    - 关闭其它插件( `Netty`有自己的优雅关闭机制，能确保`outbound`队列的消息能全部发送出去)


至此，容器关闭，服务结束了其使命。

![](https://raw.githubusercontent.com/dapeng-soa/documents/master/images/dapeng-logo/%E5%A4%A7%E9%B9%8Flogo-02.png)
