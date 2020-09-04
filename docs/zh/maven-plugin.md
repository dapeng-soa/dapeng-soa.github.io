---
layout: doc
title: Maven插件
---

## Maven插件功能

## 启用Maven插件
修改maven的主配置文件（${MAVEN_HOME}/conf/settings.xml文件或者 ~/.m2/settings.xml文件），添加如下配置：

```
<pluginGroups>
    <pluginGroup>com.github.dapeng</pluginGroup>
  </pluginGroups>
```

## 项目结构
> 请参考: https://github.com/dapeng-soa/dapeng-hello.git

一个基于`dapeng-soa`的`Maven`项目，其结构类似如下:
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

### 客户端代码生成
我们需要通过`IDL`手工定义服务接口，为了通过`IDL`生成api代码，我们需要在项目的api子模块(例如上面的`hello-api`)的`pom.xml`中加上如下配置:
```
<build>
        <plugins>
            <plugin>
                <groupId>com.github.dapeng-soa</groupId>
                <artifactId>dapeng-maven-plugin</artifactId>
                <version>2.1.1</version>
                <executions>
                    <execution>
                        <phase>generate-sources</phase>
                        <goals>
                            <goal>thriftGenerator</goal>
                        </goals>
                        <configuration>
                            <!--配置生成哪种语言的代码[java,scala]-->
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
然后，通过`mvn compile`即可生成api代码。


## 服务启动

### 确保本地zookeeper服务已启动启动:
> 一种简单的方式， 可以使用docker镜像启动zookeeper

```
docker pull zookeeper
docker run --name zookeeper -p 2181:2181 --restart always -d zookeeper
```


### 使用下面的方式来运行一个service 

> 在`hello-service/pom.xml` 加上 `dapeng-maven-plugin`如下:

```xml
<plugin>
    <groupId>com.github.dapeng-soa</groupId>
    <artifactId>dapeng-maven-plugin</artifactId>
    <version>2.1.1</version>
</plugin>
```

> 运行service, 服务默认注册端口:9090 默认在线测试文档端口:8080
```
cd hello-service
mvn compile com.github.dapeng:dapeng-maven-plugin:2.1.1:run
```

服务启动后，就可以启动文档站点进行接口调试了 [访问 `http://localhost:8080` 在线测试]
