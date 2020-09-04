---
layout: doc
title: 慢服务检测
---
## 慢服务监控
大鹏框架实现的对服务的慢服务检测，我们可以设置服务以及每个接口的最大执行时间，当某次调用时间超过设置的时间，系统就会检测到此次服务调用为慢服务，会记录服务执行的堆栈信息到日志服务器(EFK)中。
开启慢服务检测需要设置以下配置：
* 首先我们需要设置慢服务检测的总开关，设置环境变量打开慢服务检测如下：
```
 slow.service.check.enable=true //慢服务设置开关
 soa.max.process.time=4000 //设置慢服务阈值（默认3000）
```
* 设置接口的慢服务检测阈值,可以通过以下两种方法实现:

  * 开发时可以在实现接口的时候通过MaxProcessTime注解实现(此方法如要更改时间阈值需要更新代码)，配置如下：
  
    ```
    @MaxProcessTime(maxTime = 300)
      override def sayHello(hello: Hello): String = {
           // Thread.sleep(200)
        s"*******************收到的消息为： ${hello} HelloServiceImpl version=2.0.1"
        // info.formatted("%.0f")
      }
    ```
    
  * 可以在ZK配置中心加入慢服务时间阈值的配置(此方法如要更改时间阈值配置即生效)，ZK配置如下(配置类似超时时间[timeout])：
  
    在ZK的节点路径：soa/config/services/xxx.service下对相应的服务以下设置:
  
    ```
     timeout/800ms,register:4001ms,modifySupplier:200ms
     processTime/3000ms,register:4001ms,modifySupplier:200ms
     loadBalance/random,createSupplier:random,modifySupplier:roundRobin
     weight/192.168.4.107/9095/700
     weight/192.168.4.107/500
    ```
  说明：processTime:对xxx.service服务设置慢服务时间阈值，该服务中的register和modifySupplier两个接口方法设置不同的慢服务时间阈值；
