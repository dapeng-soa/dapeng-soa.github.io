---
layout: doc
title: 命令行工具
---
## 命令行工具[dapeng-cli]

  [dapeng-soa](https://github.com/dapeng-soa/dapeng-soa) 是一款开源的高性能服务化框架，命令行工具提供了一系列命令可以帮助我们进行系统参数调优、系统运行状态的查看，分布式服务调用、查看日志、配置中心的配置管理等等。大鹏框架的自检系统可以确保每个服务都能健康的进行，其中服务的健康度检查就是通过调用命令去完成的。

命令行工具源码下载 http://pms.today36524.com.cn:8083/basic-services/dapeng-cli

## 1.普通使用方式:  
    > cd dist  
    > java -jar cli.jar

## 2.脚本调用方式:  
### 2.1 单指令脚本调用:  
    > cd dist  
    > java -jar cli.jar service -list  
    
### 2.2 文件指令集(文件中一条指令占一行，如文件 cmd_list.cmd)脚本调用:
    > cd dist  
    > java -jar cli.jar cmd_list.cmd
       
### 2.3 指定环境变量[soa.zookeeper.host]启动:   
    > java -Dsoa.zookeeper.host=10.10.10.45 -jar cli.jar service -list   

## 3. 命令使用说明:    
    
### 1. zk 命令使用说明[zookeeper   节点相关的操作]
    1.1  zk -get [path]                             获得zk某个节点的数据
    1.2  zk -set [path] -d [data]                   设置zk某个节点的数据
    1.3  zk -nodes [path]                           获取zk节点下的子节点列表
     
###  2. set 命令使用说明[设置系统参数 目前主要支持设置 invocationContext、 zookeeper的zkHost.注意:通过set 指令设置的值， 在当前命令行生命周期有效]
    2.1  set                     查看设置的信息
    2.2  set -timeout [value]    设置invocationContext 超时时间
    2.3  set -callermid [value]  设置invocationContext Callermid
    2.4  set -calleeip [value]   设置invocationContext calleeip
    2.5  set -calleeport [value] 设置invocationContext calleeport
    2.6  set -callerip [value]   设置invocationContext callerip
    2.7  set -zkhost [value]     设置 zkhost
    2.8  set -callerfrom [value] 设置invocationContext callerfrom

###  3. unset 命令使用说明[撤销 set 指令的赋值]  
    3.2  unset -timeout     撤销invocationContext 超时时间
    3.3  unset -callermid   撤销invocationContext Callermid
    3.4  unset -calleeip    撤销invocationContext calleeip
    3.5  unset -calleeport  撤销invocationContext calleeport
    3.6  unset -callerip    撤销invocationContext callerip
    3.7  unset -zkhost      撤销 zkhost
    3.8  unset -callerfrom  撤销invocationContext callerfrom
     
### 4. metadata 命令使用说明[获取服务接口的元数据]  
    4.1 metadata -s [serviceName] -v [version]                     获取服务接口的元数据信息(结果直接打印在控制台)
    4.2 metadata -s [serviceName] -v [version] -o [path+fileName]  获取服务接口元数据信息并保存到指定路径
      
###   5. json 命令使用说明[获取服务调用的json格式样例]  
    5.1 json -s [serviceName] -v [version] -m [method]                    获取服务调用的json格式样例在控制台打印;eg:  json -s com.today.api.order.service.OrderService -m queryOrderList -v 1.0.0
    5.2 json -s [serviceName] -v [version] -m [method] -o [path+filename] 获取服务调用的json格式样例,保存到指定文件
      
###    6. service 命令使用说明[获取当前运行时实例的服务列表]  
    6.1 service -list    获取当前运行时实例的服务列表;eg: service -list
        6.1.1 service -list                            获取当前运行时实例的服务列表 控制台显示;eg: service -list
        6.1.2 service -list -o /tmp/service_list.txt   获取当前运行时实例的服务列表,输出到文件;eg: service -list  -o /tmp/service_list.txt 
    6.2 service -runtime 获取服务实例列表 控制台显示
        6.2.1 service -runtime -o /tmp/run_time.json   获取服务实例列表 输出到文件
    6.3 service -route 获取服务实例路由配置 控制台显示
        6.3.1 service -route -o /tmp/route.json         获取服务实例路由配置 输出到文件
        6.3.2 service -route -f /tmp/route.json         从文件读取路由信息设置到某个service下
        6.3.3 service -route -d "ip match ~127.0.0.1"   路由信息[ip match ~127.0.0.1]设置到某个service下
    6.4 service -config 获取服务 config配置信息  控制台显示
        6.4.1 service -config -o /tmp/config.json               获取服务config配置信息 输出到文件
        6.4.2 service -config -f /tmp/config.json               从文件读取config配置信息设置到某个service下
        6.4.3 service -config -d "timeout=1000ms;timeout=120"   config配置信息[timeout=1000ms;timeout=120]设置到某个service下
    6.5 service -method 获取服务方法列表 控制台显示
        6.5.1 service -method -o /tmp/service_method.json   获取服务方法列表 输出到文件       
        6.5.2 service -method                               获取服务方法列表 输出到控制台
    6.6 service -whitelist 增加服务白名单设置 控制台显示
        6.6.1 service -whitelist                                          获取白名单服务信息 输出到控制台
        6.6.2 service -whitelist -o /tmp/whitelist.json                   获取白名单服务信息 输出到文件
        6.6.3 service -whitelist -d com.today.api.idgen.service.IDservice 设置白名单服务信息
      
###    7. method 命令使用说明[获取服务接口的方法列表]  
    7.1 method -s [serviceName]    获取服务接口的方法;eg: method -s com.today.api.order.service.OrderService
      
###    8. request 命令使用说明[请求服务接口]  
    8.1 request -s [serviceName] -v [version] -m [method] -f [path+fileName]   请求服务接口
        -s serviceName   服务名
        -v version       版本号
        -m method        方法名
        -f fileName      请求的json格式文件    
    8.2 request -metadata [serviceName] -v [version]                     请求服务接口的元数据;eg: request -metadata com.today.api.order.service.OrderService -v 1.0.0
    8.3 request -metadata [serviceName] -v [version]  -o [path+fileName] 请求服务接口的元数据,并保存到指定文件
    8.4 request -echo [serviceName] -v [version]                         请求服务接口echo检查;eg: request -echo com.today.api.order.service.OrderService -v 1.0.0
    8.5 request -echo [serviceName] -v [version]  -o [path+fileName]     请求服务接口echo检查,并保存到指定文件

###    9. log 命令使用说明[查看堆栈错误日志]  
    使用例子： log -date 2018.07.05 -o g://log.txt -slogtime 07-05 16:42:01 121 -elogtime 07-05 16:44:00 126 -hostname 127.0.0.1 -tag OrderService 
    可选参数：
      -date 2018.07.05        日期
      -hostname hostname      hostName
      -sessiontid sessiontid  sessiontid
      -threadpool threadpool  线程号    
      -tag tag                tag 
      -slogtime slogtime      日志开始时间
      -elogtime elogtime      日志结束时间
      -o                      查询日志保存到指定文件

###    10. dumpmem 命令使用说明[查看共享内存的限流数据]  
    10.1  dumpmem -f [filePath]    查看共享内存的限流数据(-f 共享内存的路径)
       
###  11. help 命令使用说明[通过  help cmd 可以查看命令使用指南]  
    11.1 help     查看所有指令的用法
    11.2 help cmd 查看某个指令的详细用法
