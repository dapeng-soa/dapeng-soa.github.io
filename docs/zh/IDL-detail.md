---
layout: doc
title: 详解IDL
---

## 写在大纲之前：
	Thrift作为久经考验的老牌RPC编解码协议, 以其丰富的数据表达能力/以及高效的序列化反序列化能力而广受好评, dapeng使用Thrift作为其RPC应用协议.

## 参考文献:
	http://thrift.apache.org/static/files/thrift-20070401.pdf  (thrift 白皮书)

## 1. 什么是thrfit?
	Thrift是一个最初由Facebook公司开发的软件库和代码产生工具集，它加速了服务前端和后端的开发和实现。它的主要目标是使跨语言的高效、可靠通信成为可能，通过抽象每种语言的特定部分,满足由各种语言实现的通用库趋于最大化定制的需求。尤其是，Thrift允许开发者在一个语言描述文档中定义数据结构和服务，并产生构建RPC客户端和服务器端的所有必需代码。

	理论上thrift可以支持任何语言：
	1.1. Thrift原生支持C++, java, python, Ruby 和 PHP 
	1.2. 目前dapeng框架提供的maven & scala 插件仅支持生成Java & Scala 语言的开发模板， 更多跨语言模板敬请期待 (https://github.com/dapeng-soa/dapeng-soa)

## 2. 为什么用thrift. 

###     2.1 通用:
	    
		由于现实服务开发中会有多语言并存的情况(例如：java, scala, JavaScript等)同时进行开发，我们需要一种跨语言的技术作为桥梁来链接不同的开发语言

###     2.2 简单
		是的，使用thrift生成工具，更方便开发人员面向接口编程，开发只需要关注业务结构，接口定义， 编写thrift文件，再也不需要做一些额外的体力活了，业务只需要关注具体的实现即可，这能大大提高开发效率

### 	2.3 方便 

	使用thrift代码工具生成代码只需要两步: 编写thrift文件 -> 编译文件 (通过dapeng的maven插件或者sbt插件可以很简单的一条指令编译好thrift文件)
![compile](https://github.com/dapeng-soa/documents/blob/master/images/dapeng-thrift/thrift_compile.png?raw=true)
	

## 3. 如何定义thrift文件
### 3.1. 基础类型
**thrift 原生支持的类型如下:**
* ***bool 布尔类型***
* ***byte 类型***
* ***i16 表示一个带符号16位整型 可以等同于 short***
* ***i32 表示一个带符号32位整型 可以等同于 int***
* ***i64 表示一个带符号32位整型 可以等同于 long***
* ***double 表示一个带符号的64位浮点数***
* ***string 表示一个字符串(注意是小写的 string)***

以上是Thrift原生支持的数值类型， 那么问题来了，如果需要用到比较常见的Timestamp, BigDecimal类型怎么办呢?
> ***大鹏框架自研的thrift生成器通过注解的方式提供了自定义数值类型的生成模板:***
```
struct HelloDataType {
    /**
    *  @datatype(name="bigdecimal")
    *  
    *  java版：生成java.
    **/
    1: double decimalNumber
    /**
    *  @datatype(name="date")
    **/
    2: i64 date
}
```
> dapeng框架的thrift生成器通过 ***@datatype(name="type")*** 注解可以拓展我们所需的数值类型，目前框架仅支持 java.util.Date, BigDecimal的类型拓展，更多的数据类型敬请期待.........


### 3.2. 结构体(Struct) 
一个thrift结构定义了一个通用的对象以此来跨语言，在面向对象语言中，一个Struct本质上就等同于一个类。而每个struct的字段都有一个唯一的标识。
```
struct StructExample {
    /**
    * 布尔类型字段
    **/
    1: bool isExample
    /**
    * 32位整型
    **/
    2: i32  structId
    /**
    * 字符串类型
    **/
    3: string content
    /**
    * struct类型结构
    **/
    4: HelloDataType hello
    /**
    *  集合类型
    **/
    5: list<i64> someVals
}
```

>a.  如上所示， StructExample 定义了一个thrift结构体， 里面包含了基础类型(bool, i32, string), 结构体(Struct: HelloDataType), 集合(list).
<br/>
>b. 同时你应该也注意到了在类型前面定义的序号， 这是结构体字段属性的唯一标识，注意该标识需要唯一； 但是他们不是必须连续的，你可以把序号定义为 (1,3,5,7,9), 只需要保证数值唯一即可。
<br />
>`Ps: 如果你设计的结构需要拓展的话，不要重设字段的序号， 这样会导致前后版本不一致, 所以建议设计结构体的时候，序号可以预留一些空间， 如定义为(1,5,10.,15,20). 详细的版本兼容情况，请查看第7点`


### 3.3. 集合
>Thrift中， 支持三种集合:
>* ***list<dataType>***    一个有序元素列表，等同于java的ArrayList
>* ***set<dataType>***     一个无序不重复元素集, 等同于java的HashSet    
>* ***map<dataType1, dataType2>*** 一个主键唯一键值映射表，等同于java的HashMap

如3.2样例所示, 跟java一样，集合中可以包含任意数值类型

### 3.4. 服务(service)

> 服务通过thrift类型 `service` 来定义， 一个服务的定义在语义上相当于面向对象编程中定义一个接口，所以thrift编程也相当于接口编程。服务定义的格式如下：
<br />
service <serviceName> {
<br />
&ensp; &ensp;   <returnType> <methodName>(<arguments>)
<br />
}


我们来定义一个简单的helloService接口:

```java
service HelloService {
    void saySomething(1: string content)
}

```
> HelloService 定义了1个方法:
<br />
> 1.  saySomething 接收一个String类型参数， 返回值是void

`Ps: 注意void类型是除了所有已经被定义的thrift类型外的一个合法的函数的返回类型。`

### 3.5 thrift文件引用
> 一般我们在设计接口的时候,会把接口定义与Struct定义分开定义在不同的thrift文件, <br> 如： 
<br>
domain.thrift  -> 用来定义结构体
<br>
service.thrift -> 用来定义接口
<br>
> `那么service.thrift定义的接口如何能引用到domain.thrift定义的结构体呢?`

下面我们来看一个完整的例子 (domain.thrift 与 service.thrift需要在同一个文件夹)

> `domain.thrift`
```
namespace java com.github.dapeng.domain

struct HelloWorld {
    1: string content
}

```

> service.thrift

```
namespace java com.github.dapeng.service

include "domain.thrift"

service HelloService {
    void saySomething(1: domain.HelloWorld content)
}

```

> 我们还是以EchoService 为例， 不过saySomething的方法改造了一下. 首先接口要引入domain.thrift文件里的结构体 需要用 `include` 关键字显式的导入domain.thrift文件， 然后使用 `文件名.结构名` 的方式引用结构体， 如上所示的 saySomething方法的接口参数调用了 domain里的HelloWorld 结构体.  


### 4. thrift编译器

> 上面第三节我们已经简单介绍了thrift的一些简单语法以及结构，那么这里我们来看一下基于thrift文件来生成接口&数据结构实例（生成方式请参考 第一章 `快速入门`）

#### 4.1 dapeng版编译器

> 使用sbt-dapeng插件执行 `> sbt clean api/compile` 会生成以下文件
![summary](https://github.com/dapeng-soa/documents/blob/master/images/dapeng-thrift/thrift_summary.png?raw=true)

> <b><p style="color:red">上图是基于HelloService thrift文件编译后的文件(这里只列出了`Java`版代码，对`scala`版本感兴趣的可以自行编译查看，这里不再赘述) </p><b> 

* ***1. HelloWorld*** &ensp; &ensp; &ensp; &ensp; helloWorld的结构体
* ***2. HelloWorldSerializer*** &ensp; &ensp; &ensp; &ensp; helloWolrd结构体的序列化器
* ***3. HelloService*** &ensp; &ensp; &ensp; &ensp;  HelloService服务端接口
* ***4. HelloServiceCodec*** &ensp; &ensp; &ensp; &ensp;  HelloService服务端接口编解码器
* ***5. HelloServiceAsync*** &ensp; &ensp; &ensp; &ensp;  HelloService服务端异步接口
* ***6. HelloServiceAsyncCodec*** &ensp; &ensp; &ensp; &ensp;  HelloService服务端异步接口编解码器
* ***7. HelloServiceClient*** &ensp; &ensp; &ensp; &ensp;  HelloService客户端
* ***8. HelloServiceAsyncClient*** &ensp; &ensp; &ensp; &ensp;  HelloService异步客户端
* ***9. HelloServiceSuperCodec*** &ensp; &ensp; &ensp; &ensp;  HelloService父类编解码器


##### 4.1.1 HelloWorld 实体
```
public class HelloWorld {

    public String content;
    public String getContent() {
        return this.content;
    }
    public void setContent(String content) {
        this.content = content;
    }
    public String content() {
        return this.content;
    }
    public HelloWorld content(String content) {
        this.content = content;
        return this;
    }
    public static byte[] getBytesFromBean(HelloWorld bean) throws TException {
        ......
    }
    public static HelloWorld getBeanFromBytes(byte[] bytes) throws TException {
        ......
    }
    public String toString() {
       ......
    }
}
```

`PS: 需要一提的是 getBytesFromBean 与 getBeanFromBytes 两个方法是dapeng thrift编译器 额外提供实现的对象实体与bytes数组之间序列化与反序列的工具方法`

##### 4.1.2 HelloWorldSerializer
```
public class HelloWorldSerializer implements BeanSerializer<com.github.dapeng.domain.HelloWorld> {

    @Override
    public com.github.dapeng.domain.HelloWorld read(TProtocol iprot) throws TException {
        ......
    }
    @Override
    public void write(com.github.dapeng.domain.HelloWorld bean, TProtocol oprot) throws TException {
        ......
    }
    public void validate(com.github.dapeng.domain.HelloWorld bean) throws TException {
        ......
    }
    @Override
    public String toString(com.github.dapeng.domain.HelloWorld bean) {
        return bean == null ? "null" : bean.toString();
    }
}
```

> HelloWorldSerializer 实现了HelloWorld实体序列化反序列化的读写方法


##### 4.1.3 HelloService接口
```
@Service(name = "com.github.dapeng.service.HelloService", version = "1.0.0")
@Processor(className = "com.github.dapeng.HelloServiceCodec$Processor")
public interface HelloService {
    void saySomething(com.github.dapeng.domain.HelloWorld content) throws com.github.dapeng.core.SoaException;
}
```
> 生成的HelloService与我们平时定义的java接口没什么区别，只是多了两个注解, 这两个注解主要用于 dapeng-spring 的服务发现机制，dapeng-spring通过该注解搜索对应的服务并初始化 (后续dapeng容器启动详解会详细说明)

##### 4.1.4 HelloServiceCodec
```
public class HelloServiceCodec {

    public static class saySomething<I extends com.github.dapeng.service.HelloService> extends SoaFunctionDefinition.Sync<I, saySomething_args, saySomething_result> {
        public saySomething() {
            ......
        }
        @Override
        public saySomething_result apply(I iface, saySomething_args saySomething_args) throws SoaException {
            ......
        }
    }

    public static class getServiceMetadata<I extends com.github.dapeng.service.HelloService> extends SoaFunctionDefinition.Sync<I, getServiceMetadata_args, getServiceMetadata_result> {
        public getServiceMetadata() {
            ......
        }
        @Override
        public getServiceMetadata_result apply(I iface, getServiceMetadata_args args) {
            ......
        }
    }

    public static class echo<I extends com.github.dapeng.service.HelloService> extends SoaFunctionDefinition.Sync<I, echo_args, echo_result> {
        public echo() {
            ......
        }
        @Override
        public echo_result apply(I iface, echo_args args) {
           ......
        }
    }

    @SuppressWarnings("unchecked")
    public static class Processor<I extends com.github.dapeng.service.HelloService> extends SoaServiceDefinition<com.github.dapeng.service.HelloService> {
        public Processor(com.github.dapeng.service.HelloService iface, Class<com.github.dapeng.service.HelloService> ifaceClass) {
            super(iface, ifaceClass, buildMap(new java.util.HashMap<>()));
        }
        @SuppressWarnings("unchecked")
        private static <I extends com.github.dapeng.service.HelloService> java.util.Map<String, SoaFunctionDefinition<I, ?, ?>> buildMap(java.util.Map<String, SoaFunctionDefinition<I, ?, ?>> processMap) {
            processMap.put("saySomething", new saySomething());
            processMap.put("getServiceMetadata", new getServiceMetadata());
            processMap.put("echo", new echo());
            return processMap;
        }
    }

}
```

> * saySomething 是HelloService thrift文件的方法定义
> * getServiceMetadata 是大鹏框架附加的一个针对服务定制的获取服务元数据的方法实现， 用于获取服务的元数据信息 (元数据: xml格式描述服务信息的文件)
> * echo 方法是dapeng框架用于健康检查的附加方法（调用echo 方法来判断该服务是否正常）
> * Processor 是dapeng框架基于spring自定义标签<soa:service>来做服务发现的（通过标签来获取服务的 BeanDefinition信息(包含了服务接口信息，方法信息) ）


##### 4.1.5  HelloServiceAsync
```
@Service(name = "com.github.dapeng.service.HelloService", version = "1.0.0")
@Processor(className = "com.github.dapeng.HelloServiceAsyncCodec$Processor")
public interface HelloServiceAsync extends com.github.dapeng.core.definition.AsyncService {
    Future<Void> saySomething(com.github.dapeng.domain.HelloWorld content) throws com.github.dapeng.core.SoaException;
}
```
> HelloServiceAsync 是HelloService的异步接口定义


##### 4.1.6 HelloServiceAsyncCodec
```
public class HelloServiceAsyncCodec {
    public static class saySomething<I extends com.github.dapeng.service.HelloServiceAsync> extends SoaFunctionDefinition.Async<I, saySomething_args, saySomething_result> {
        public saySomething() {
           ......
        }
        @Override
        public CompletableFuture<saySomething_result> apply(HelloServiceAsync iface, saySomething_args saySomething_args) throws SoaException {
            ......
        }
    }

    public static class getServiceMetadata<I extends com.github.dapeng.service.HelloServiceAsync> extends SoaFunctionDefinition.Async<I, getServiceMetadata_args, getServiceMetadata_result> {
        public getServiceMetadata() {
            ......
        }
        @Override
        public CompletableFuture<getServiceMetadata_result> apply(I iface, getServiceMetadata_args args) {
           ......
        }
    }

    public static class echo<I extends com.github.dapeng.service.HelloServiceAsync> extends SoaFunctionDefinition.Async<I, echo_args, echo_result> {
        public echo() {
            ......
        }
        @Override
        public CompletableFuture<echo_result> apply(I iface, echo_args args) {
            ......
        }
    }

    @SuppressWarnings("unchecked")
    public static class Processor<I extends com.github.dapeng.service.HelloServiceAsync> extends SoaServiceDefinition<com.github.dapeng.service.HelloServiceAsync> {
        public Processor(com.github.dapeng.service.HelloServiceAsync iface, Class<com.github.dapeng.service.HelloServiceAsync> ifaceClass) {
            super(iface, ifaceClass, buildMap(new java.util.HashMap<>()));
        }
        @SuppressWarnings("unchecked")
        private static <I extends com.github.dapeng.service.HelloServiceAsync> java.util.Map<String, SoaFunctionDefinition<I, ?, ?>> buildMap(java.util.Map<String, SoaFunctionDefinition<I, ?, ?>> processMap) {
            processMap.put("saySomething", new saySomething());
            processMap.put("getServiceMetadata", new getServiceMetadata());
            processMap.put("echo", new echo());
            return processMap;
        }
    }
}
```
> HelloServiceAyncCode 跟HelloServiceCodec类似，只是提供的异步的实现


4.1.7  HelloServiceClient
```
public class HelloServiceClient implements HelloService {
    private final String serviceName;
    private final String version;
    private SoaConnectionPool pool;
    private final SoaConnectionPool.ClientInfo clientInfo;

    public HelloServiceClient() {
        ......
    }

    public HelloServiceClient(String serviceVersion) {
        ......
    }

    public void saySomething(com.github.dapeng.domain.HelloWorld content) throws SoaException {
        ......
    }
    /**
     * getServiceMetadata
     **/
    public String getServiceMetadata() throws SoaException {
        ......
    }

    /**
     * echo
     **/
    public String echo() throws SoaException {
       ......
    }
}
```
> HelloServiceClient dapeng编译器已经提供生成了HelloService接口的客户端实现，用户不必再手写，直接调用即可 -> 即定义好thrift文件后，只需要关注服务端业务逻辑的实现, 省去不必要的代码编写操作.
 

##### 4.1.7 HelloServiceAsyncClient
```
public class HelloServiceAsyncClient implements HelloServiceAsync {
    private final String serviceName;
    private final String version;

    private SoaConnectionPool pool;
    private final SoaConnectionPool.ClientInfo clientInfo;

    public HelloServiceAsyncClient() {
        ......
    }

    public HelloServiceAsyncClient(String serviceVersion) {
        ......
    }

    public CompletableFuture<Void> saySomething(com.github.dapeng.domain.HelloWorld content) throws SoaException {
        ......
    }

    /**
     * getServiceMetadata
     **/
    public String getServiceMetadata() throws SoaException {
        ......
    }

    /**
     * echo
     **/
    public String echo() throws SoaException {
        ......
    }
}
```
> HelloServiceAyncClient 是客户端的异步实现. 由此可以看到，大鹏同异步处理，客户端跟服务端是完全独立分离的, 你可以:
> * syncClient invoke syncServer
> * syncClient invoke asyncServer
> * asyncClient invoke syncServer
> * asyncClient invoke asyncServer
<br>

> Ps: ***但是要注意dapeng框架限定***: 同一个服务只能存在一种实现方式，同一个服务不能同时存在同步实现与异步实现(即: 在注册SpringBean<soa: service>的时候，不能同时注册两个不同实现的ServiceBean)


##### 4.1.8  HelloServiceSuperCodec 
```
public class HelloServiceSuperCodec {
    //1. method_args
    public static class saySomething_args {
        ......
    }
    //2. method_result
    public static class saySomething_result {
    }
    //3. args_serializer
    public static class SaySomething_argsSerializer implements BeanSerializer<saySomething_args> {
        @Override
        public saySomething_args read(TProtocol iprot) throws TException {
            ......
        }

        @Override
        public void write(saySomething_args bean, TProtocol oprot) throws TException {
           ......
        }

        public void validate(saySomething_args bean) throws TException {
           ......
        }

        @Override
        public String toString(saySomething_args bean) {
            return bean == null ? "null" : bean.toString();
        }
    }

    //4.ResultSerializer
    public static class SaySomething_resultSerializer implements BeanSerializer<saySomething_result> {
        @Override
        public saySomething_result read(TProtocol iprot) throws TException {
            ......
        }

        @Override
        public void write(saySomething_result bean, TProtocol oprot) throws TException {
            ......
        }
        public void validate(saySomething_result bean) throws TException {
        }
        @Override
        public String toString(saySomething_result bean) {
            return bean == null ? "null" : bean.toString();
        }
    }

    //5.
    //6. meta_args
    public static class getServiceMetadata_args {
    }

    //7. meta_result.
    public static class getServiceMetadata_result {
        ......
    }

    //8. args_serializer
    public static class GetServiceMetadata_argsSerializer implements BeanSerializer<getServiceMetadata_args> {
        @Override
        public getServiceMetadata_args read(TProtocol iprot) throws TException {
            ......
        }

        @Override
        public void write(getServiceMetadata_args bean, TProtocol oprot) throws TException {
            ......
        }

        public void validate(getServiceMetadata_args bean) throws TException{
            ......
        }
        @Override
        public String toString(getServiceMetadata_args bean) {
            return bean == null ? "null" : bean.toString();
        }
    }

    //9. meta_resultSerializer
    public static class GetServiceMetadata_resultSerializer implements BeanSerializer<getServiceMetadata_result> {
        @Override
        public getServiceMetadata_result read(TProtocol iprot) throws TException {
            ......
        }

        @Override
        public void write(getServiceMetadata_result bean, TProtocol oprot) throws TException {
            ......
        }

        public void validate(getServiceMetadata_result bean) throws TException {
            ......
        }

        @Override
        public String toString(getServiceMetadata_result bean) {
            return bean == null ? "null" : bean.toString();
        }
    }

    //10. echo_args
    public static class echo_args {}

    //11. echo_result.
    public static class echo_result {
        ......
    }

    //12. echo_argsSerializer
    public static class echo_argsSerializer implements BeanSerializer<echo_args> {

        @Override
        public echo_args read(TProtocol iprot) throws TException {
            ......
        }
        @Override
        public void write(echo_args bean, TProtocol oprot) throws TException {
            ......
        }

        public void validate(echo_args bean) throws TException {}

        @Override
        public String toString(echo_args bean) {
            return bean == null ? "null" : bean.toString();
        }
    }

    //13. echo_resultSerializer
    public static class echo_resultSerializer implements BeanSerializer<echo_result> {
        @Override
        public echo_result read(TProtocol iprot) throws TException {
            ......
        }
        @Override
        public void write(echo_result bean, TProtocol oprot) throws TException {
            ......
        }
        public void validate(echo_result bean) throws TException {
            ......
        }
        @Override
        public String toString(echo_result bean) {
            return bean == null ? "null" : bean.toString();
        }
    }

}
```

> HelloServiceSuperCodec 定义了同步，异步方法相关实体的序列化，反序列实现。
> 由于 HelloServiceCodec & HelloServiceAsyncCode定义的相关实体其实是一致的，所以把它们抽离出来用以压缩生成的api包大小 (减少1/3左右)


#### 4.2 Thrift原生编译器

> `> thrift-0.11.0 -gen java domain.thrift`

>  `> thrift-0.11.0 -gen java service.thrift`



##### 4.1.3 dapeng thrift编译器 Vs Thrift原生编译器的优势
> 1. dapeng thrift编译器是基于scala自主研发实现的，自己拓展功能比较方便，可读性好
> 2. 文件生成大小相对来说较小，生成的代码可读性较好
> 3. 原生thrift支持的数据类型比较局限，拓展较困难， dapeng的数据类型拓展比较方便，如BigDecimal, Timestamp的拓展
> 4. 元数据的支持，dapeng编译还有一大利器就是服务的元数据信息。这点可以第5小节。


#### 5 服务元数据(serviceMetadata)

元数据文件解析模块采用TypeScript语言(xml)对服务进行描述，并通过该文件的解析，可以动态生成文档站点。 元数据信息包括但不限于：服务元信息，方法，服务包含的请求体，枚举类型，注解。元数据信息作为服务的载体，描述了服务的这些必要信息，方便了我们服务信息的展示与针对服务的一些特定操作(如dapeng-json的序列化与反序列的操作就是基于服务元数据)

> 元数据(Metadata) 是关于`数据的数据` 或者叫做用来描述数据的数据，是信息共享和交换的前提，它具有以下特点:
* 元数据是对服务的结构化的描述，用户不必过度认识元数据文件本身
* 元数据包含用于描述信息对象的内容和位置的数据元素集，而且元数据不仅是对服务对象进行描述，为了方便，我们还能针对服务做一系列拓展(如: 针对服务分组展示，添加服务事件)


> com.github.dapeng.hello.service.HelloService.xml
```
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<service namespace="com.github.dapeng.hello.service" name="HelloService">
    <meta>
        <version>1.0.0</version>
        <timeout>30000</timeout>
    </meta>
    <methods>
        <method name="saySomething">
            <request name="saySomething_args">
                <fields>
                    <field tag="1" name="content" optional="false" privacy="false">
                        <dataType>
                            <kind>STRUCT</kind>
                            <ref>com.github.dapeng.hello.domain.HelloWorld</ref>
                        </dataType>
                        <doc></doc>
                    </field>
                </fields>
            </request>
            <response name="saySomething_result">
                <fields>
                    <field tag="0" name="success" optional="false" privacy="false">
                        <dataType>
                            <kind>VOID</kind>
                        </dataType>
                        <doc></doc>
                    </field>
                </fields>
            </response>
            <isSoaTransactionProcess>false</isSoaTransactionProcess>
        </method>
    </methods>
    <structs>
        <struct namespace="com.github.dapeng.hello.domain" name="HelloWorld">
            <fields>
                <field tag="1" name="content" optional="false" privacy="false">
                    <dataType>
                        <kind>STRING</kind>
                    </dataType>
                </field>
            </fields>
        </struct>
    </structs>
    <enums/>
</service>

```
> 上面的代码是我们的HelloService的元数据文件，可以看到他基本描述了HelloService服务的基础结构, 任何三方语言都可以通过元数据对dapeng服务的输入输出进行编解码，也就是说dapeng服务理论上可以支持所有语言.

1 像我们的文档站点就是依赖这个元数据文件来获取到服务的详细信息，可以通过元数据在线展示并测试我们的HelloService服务
![](https://github.com/dapeng-soa/documents/blob/master/images/dapeng-apiDoc/HelloService1.png?raw=true)
> 上图的文档站点就是基于元数据信息来构建站点页面的, 通过元数据强大的类型描述，我们的web页面可以非常方便的获取到服务的基本信息，并随心所欲的展示出来，当服务变动的时候，我们不需要改动页面代码就能够动态的展示更新后的服务内容并对其接口进行测试。大大提高了测试效率.

<b />

`Ps: 有了这一利器，可以非常方便我们平时的接口开发测试`

2  高性能的dapeng-json序列化器也是基于元数据做流式的序列化与反序列化 
***

