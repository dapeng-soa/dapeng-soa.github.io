---
layout: doc
title: MockServer
---

## 1.背景

`Mockserver` 是一个前后分离的项目，前端主要是请求、响应，mock规则配置，管理控制台等。后端为实际业务交互平台。

- 前端 vue-dms
- 后台 mock-server

## 2.环境搭建

### 2.1.数据源配置
- 创建数据库 mock_server 

- 下载mock_server所需的数据sql文件: [mock_server.sql](http://file.hzways.com/mock_server_2018-11-20.sql)

- 导入初始化sql文件  


### 2.2.启动后台项目

- 启动类 `com.github.dapeng.dms.DMSApplication` 

mock_server admin 采用 springboot 工程搭建，直接允许即可启动，默认暴露的是 9000 端口。

访问 `http://127.0.0.1:9000/swagger-ui.html#/`  查看接口文档说明

![dms-front](http://pic.hzways.com/image2019-12-23_15-44-10.png)

### 2.3.前端(vue-dms)配置

```
npm install 将依赖项通过 npm 安装到本地
npm run build 构建项目
npm run serve 启动前段服务，直接访问页面即可
```

![dms-front](http://pic.hzways.com/image2019-12-23_15-43-53.png)

## 3.配置指南
> 对请求参数根据配置的信息进行匹配,匹配成功后返回配置的结果。可使用字符串插值的形式对请求参数中的部分数据进行提取，以替换结果表达式并返回。

### 3.1.匹配规则
- 用途
匹配规则为配置在mock-server中的对请求参数进行匹配的表达式。其主要使用正则等，通过配置一些表达式，来匹配请求参数，如果匹配成功，则证明
为这次请求mock成功，需要返回mock配置中的指定结果。
可同时配置多个匹配规则，并设置优先级别。


规则以 `@` 符号开头，代表当前配置的规则将采用什么形式进行解析,如果配置的值本身需要 @，则可以使用 \@ 进行转义替代

### 3.2.匹配规则功能一览

![match_rules](http://pic.hzways.com/match_rules.png)

### 3.3 匹配规则层级说明
> 在匹配规则的基础上，配置是尽可能少的。匹配过来的json，只要其层次结构上符合定义的规则，并且包含了要匹配的内容，即认为匹配成功

#### 3.3.1 字符串串匹配

- 规则

```
{
    "body": {
        "categoryCode": "666"
    }
}
```

- 匹配json

```
{
    "body": {
        "categoryCode": "666"
        "name": "category"
        ...
    }
}
```

#### 3.3.2 正则匹配模式

- 规则

```
{
    "body": {
        "categoryCode": "@r@测试.*"
    }
}
```

- 匹配json

```
{
    "body": {
        "categoryCode": "测试程序"
    }
}
{
    "body": {
        "categoryCode": "测试",
        "name": "name"
        "category": {}
    }
}
```

#### 3.3.3 整数范围
- 规则

```
{
    "body": {
        "categoryCode": "@p100..104"
    }
}
```

- 匹配json

```
{
    "body": {
        "categoryCode": "101"
    }
}

# ... 101,102,103,104
{
    "body": {
        "categoryCode": "104"
    }
}
```

#### 3.3.4 取模
- 规则

```
{
    "body": {
        "categoryCode": "@%@2n+1"
    }
}  
```

- 匹配json

```
{
    "body": {
        "categoryCode": "5"
    }
}
# Or
{
    "body": {
        "categoryCode": "11"
    }
}
```



### 3.4 基础类型匹配

```
{
  "name": "dapeng-soa", // 匹配字符串
  "bool": true, // 匹配常量
  "name": "@r@wang.*", // 匹配正则表达式 
  "age": "@range@20,40" // 匹配数值范围
}
```

### 3.5.字段提取
> 如果一个提取器被多次匹配，则返回值是一个数组。

```
{
    "name": "@f:name@", // 匹配字段，并提取字段值
    "name": "@f:name,r@wang.*" // 匹配字段，并提取字段值。如果不匹配，则返回信息中不会包括该字段值。
}
```

