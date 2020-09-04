---
layout: doc
title: DapengMesh网关的使用介绍
---



## 1.背景
> 什么是API网关,它的作用是什么，产生的背景是啥？

从架构的角度来看，API网关暴露http接口服务,其本身不涉及业务逻辑,只负责包括请求路由、负载均衡、权限验证、流量控制、缓存等等功能。其定位类似于Nginx请求转发、但功能要多于Nginx,背后连接了成百上千个后台服务，这些服务协议可能是rest的，也可能是rpc协议等等。

网关的定位决定了它生来就需要高性能、高效率的。网关对接着成百上千的服务接口,承受者高并发的业务需求，因此我们对其性能要求严苛，其基本功能如下:

- 协议转换
一般从前端请求来的都是HTTP接口,而在分布式构建的背景下，我们后台服务基本上都是以RPC协议而开发的服务。这样就需要网关作为中间层对请求和响应作转换。
将HTTP协议转换为RPC，然后返回时将RPC协议转换为HTTP协议

- 请求路由
网关背后可能连接着成本上千个服务,其需要根据前端请求url来将请求路由到后端节点中去，这其中需要做到负载均衡

- 统一鉴权
对于鉴权操作不涉及到业务逻辑，那么可以在网关层进行处理，不用下层到业务逻辑。

- 统一监控
由于网关是外部服务的入口，所以我们可以在这里监控我们想要的数据，比如入参出参，链路时间。

- 流量控制，熔断降级
对于流量控制，熔断降级非业务逻辑可以统一放到网关层。
有很多业务都会自己去实现一层网关层，用来接入自己的服务，但是对于整个公司来说这还不够。


## 2. DapengMesh
`DapengMesh` 是 `dapeng-soa` 维护团队推出的一款轻量级、异步流式化、高性能API网关。与业界大多数网关不同的是，dapeng-mesh 自己实现了 http 与thrift协议互转的流式化的dapeng-json协议，可高性能、低内存要求的对http和thrift协议进行转换。除此之外，其基于 netty作为服务容器，提供服务元数据模型等等都是非常具有特点的。

## 3. 使用与部署
### 3.1 第一步：下载源码，并 `build`
[https://github.com/dapeng-soa/dapeng-mesh](https://github.com/dapeng-soa/dapeng-mesh)

### 3.2 第二步：运行 `HttpServerApplication` 入口类
![image.png](https://upload-images.jianshu.io/upload_images/6393906-edc907e8a77b7106.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

运行主类，在本地调试模式下，需要加入下面这些启动参数
```
-Dsoa.open.auth.enable=false -Dsoa.white.list.enable=false -Dsoa.zookeeper.host=127.0.0.1:2181
```
- `soa.open.auth.enable` 默认值为 true，即开启鉴权
默认情况下，网关会对每次请求进行通过 `apiKey` 和 `secret` 密钥的形式对请求进行鉴权，在本机进行调试的情况下，我们可通过`soa.open.auth.enable`暂时关闭网关鉴权。

- `soa.white.list.enable` 默认值为 true，只暴露白名单服务
网关默认情况下会启用白名单模式，只有在注册中心或配置文件中配置有白名单的服务，才会在网关中暴露，并请求成功，而没有配置白名单的服务一并调用不同。在开发模式下，我们可以通过 `-Dsoa.white.list.enable=false` 来关闭，这样本地调试的过程中，所有的服务均可以暴露出去。

- `soa.zookeeper.host` 默认值为 127.0.0.1:2181 
通过 `-Dsoa.zookeeper.host` 指定网关连接的注册中心的地址。

### 3.3 第三步，在本地启动一个 dapeng 服务
> 以 dapeng-hello 为例，待启动成功。

服务启动成功之后会将自己的信息注册到zookeeper上，此时网关会进行同步。
通过访问 http://127.0.0.1:9000/api/list 查看目前网关已缓存的服务:
![已缓存服务.png](https://upload-images.jianshu.io/upload_images/6393906-2754e2bf3945dad4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)

如图，网关已缓存服务为 `com.dapeng.example.hello.service.HelloService`, 版本信息为 `1.0.0`。

### 3.4 第四步，请求示例
> 注意，开发调试情况下，上文已关闭鉴权，此请求将不携带 `apiKey` 等信息，后文会详细介绍如何通过鉴权的形式成功请求服务

首先以文档站点请求的请求数据json作为参考，然后在网关中使用Post请求来进行。
![文档站点](https://upload-images.jianshu.io/upload_images/6393906-2714f253e544709a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

- 通过 `postman` 进行数据请求,请求结构如下
![api.png](https://upload-images.jianshu.io/upload_images/6393906-c37a4b9963359b57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)

- 返回结果:

成功返回包:
```
{
	"success": "hello : maple-> Optional[hello]", -- 对应服务返回的数据
	"status": 1 -- status 为1 表示请求成功
}
```

失败返回包：
```
{
  "responseCode":"error-code",
   "responseMsg":"error-message",
   "success": {},
   "status":0	-- status 为 0 表示请求失败
}
```

## 4. API鉴权
在生产环境中使用 dapengMesh 网关时，我们需要引入鉴权，目的时有效的对网关的请求接口进行保护。

### 4.1 第一步 下载 [dapeng-mesh-auth](https://github.com/dapeng-soa/dapeng-mesh-auth) 工程

按照dapeng-mesh-auth工程的 readme 所示，采用 docker 的形式启动服务。

### 4.2 第二步，网关请求方式
新增的参数:
- apikey: 标明请求服务对应的apiKey，此key由dapeng-mesh-auth维护
- timestamp: 当前请求的时间戳，主要目的是确定 secret 的请求时效性
- secret: 请求密钥，该请求只会校验 secret 和  apiKey
- secret2 请求密钥2，与 secret 是或的关系，只会采用其中之一，secret2=MD5(apikey+tmiestamp+password+parameter)

请求模式1:
与鉴权相关的参数都以queryString的形式追加在 url 之后
```
curl 'http://gateway.xxx.cn/api/{serviceName}/{version}/{methodName}/{apikey}?timestamp=1525946628000&secret2=xxxxxx'
--data 'parameter={"body":{"code":"SKU_FINANCE_TYPE"}}'

secret2=MD5(apikey+tmiestamp+password+parameter)
```

请求模式2:
![image.png](https://upload-images.jianshu.io/upload_images/6393906-3e73f3e1edbd9655.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
保持请求service,method,version,apiKey等信息放在url之中，将上述其余的`queryString`的内容以 key-value 的形式放在body中进行请求

请求模式3:
![image.png](https://upload-images.jianshu.io/upload_images/6393906-3e7103bd8de55e2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)
将所有必要信息都放入body中，请求url中只携带 apiKey的内容


### 4.3 非鉴权模式与鉴权模式请求体区别
> 普通模式为开发过程中调试使用，不需要对接口进行鉴权。生产环境下，需要通过鉴权对接口进行保护。

下面是两种调用模式所需要的参数，总结出鉴权模式相对于普通模式需要多增加 `apiKey`,`timestamp`,`secret`或 `secret2` 三个参数。


 普通模式 |鉴权模式  |
  --- | --- |
  service  | service |
    method| method |
   version | version |
   parameter | parameter |
   X | apiKey |
   X | timestamp |
   X | secret 或 secret2 |

### 4.4 鉴权失败的部分code含义说明
> 如何判断请求是否鉴权通过呢，比如返回下面这个包，其 status 为0，说明请求失败。

```json
{
  "responseCode":"Err-Gateway-004",
   "responseMsg":"Api网关请求超时",
   "success": {},
   "status":0	
}
```
当鉴权失败时，返回的 `responseCode` 以 `Err-Gateway-` 则说明鉴权失败，具体失败原因如下表:

| code | 失败原因 |
| --- | --- |
| Err-Gateway-001 |没有对应的apiKey  |
|Err-Gateway-002  |  Secret密钥不正确|
| Err-Gateway-003 | IP规则不符合 |
| Err-Gateway-004 | timestamp时效失效 |
| Err-Gateway-005 | apiKey被禁用 |



## 5. 白名单
> 由于我们在开发调试过程中通过  `soa.white.list.enable=false` 将网关白名单功能关闭了，那么在正式生产过程中，是需要将其进行开启的，以保护内部接口。

### 5.1 加载路径

DapengMesh 会通过两种方式从两个地方去加载已配置好的白名单服务列表。
其一是在网关启动时，加载指定目录下的文件 service-whitelist.xml，该文件定义了一些固定要暴露的服务，我们可以将一些长期需要暴露使用的服务写在文件之中，具体格式如下：
![白名单服务.png](https://upload-images.jianshu.io/upload_images/6393906-3fd99256a6f54da6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/840)

其二是，网关在启动之后会订阅zookeeper  /soa/whitelist/services 下的子节点信息，我们可以将要暴露的服务全名动态的配置在 zookeeper 之上，这样就可以应对动态增加的服务接口暴露了。

### 5.2 加载过程
DapengMesh 启动时默认会加载 `/dapeng-mesh/service-whitelist.xml` 目录下的白名单xml文件。如果没有加载到此文件，则会加载 classpath下的 `service-whitelist.xml` 文件，这一部分配置固定的要暴露的白名单服务。

网关运行过程中，通过 dapeng-config-server 平台在 zookeeper 节点中的  /soa/whitelist/services 节点下增加白名单即可。

## 6. DapengMesh 部分环境变量一览


| key | 默认值  | 意义 |
| --- | --- | --- |
| soa.open.auth.enable | true  | 网关是否开启对接口进行鉴权，默认开启  |
| soa.white.list.enable | true | 网关是否开启白名单暴露，默认开启 |
| soa.zookeeper.host | 127.0.0.1:2181 | 网关连接注册中心zk地址 |


















