---
layout: doc
title: 应用协议
---

有网络通信就需要有通信协议，按照 OSI的7层模型，协议包括：
- 应用层：如HTTP、FTP、SMTP等上层应用协议
- 表示层：数据的表示、压缩等。
- 会话层：建立、管理、终止会话。
- 传输层：一般的，指TCP、UDP协议。
- 网络层：一般的，指IP、ICMP协议。
- 链路层：一般的，指 以太网、PPP等协议。
- 物理层：具体的数字信号表达层面。

大部分的通信应用，并没有严格的区分应用层、表示层和会话层，而是在应用层中混合了这3种能力，形成简化的5层模型。譬如：HTTP、FTP、SMTP、POP3等。

如何设计网络协议，有两大门派：面向文本流的网络协议和面向二进制流的网络协议。

面向文本流的网络协议，一般采用可打印的ascii字符(在中文世界里，也可以直接传输中文字符)，结合回车、Tab、ESC等控制字符。HTTP、FTP、SMTP、POP3等常见的的协议都是基于文本流的网络协议。（在《Unix编程艺术》这一经典中，推荐使用面向文本的配置文件和网络协议，并介绍这些是UNix的经典设计哲学之一，这也是很多的经典协议都是基于面向文本流的）。采用文本流的网络协议有一个很大的特色，就是对“开发者”友好，可以使用telnet这样的工具来模拟客户端进行通讯，也可以很容易的使用sniffer,tcpdumper这样的网络抓包工具来嗅探整个通信过程，并理解通信内容。当然，使用文本流协议也可以通过扩展命令的模式，来进行协议的版本升级。

面向二进制流的网络协议，则是采用更为紧凑的二进制来表示内容。TCP/IP自身的包头都是二进制的，DNS协议也是基于二进制的协议。作为HTTP的升级版本，HTTPS、HTTP/2也是基于二进制的协议，大部分的数据库驱动程序是采用二进制流的协议（redis则采用了文本流）。采用二进制流的优势是可以充分利用更底层的位来表示数据，可以实现定长的、更为压缩的数据包，在CPU效率、传输效率等方面都更有有时一些。这个性能的优点在吞吐量不大时（绝大部分的应用场景下），并不明显，反之，却需要承担对开发者不友好的缺点。这也是UNIX哲学中鼓励使用基于文本流的协议，而不是二进制流的原因。

# dapeng 表示层协议设计

设计一个SOA框架的通信协议，除了考虑到上述的因素之外，还有一个更为重要的考量是采用什么方式来表示“请求/返回”的数据结构。在这方面，我们列举一些我们所知的数据表示方法：
- name value pairs。使用 `key=value&key=value`的方式表示数据。这个方法非常简单，也是很多早期软件的选择。可以适应较为简单的服务，缺点是只能处理简单的数据类型，而难以处理结构、集合等复杂数据类型。
- edifact https://en.wikipedia.org/wiki/EDIFACT EDIFACT是一个早期广泛应用与航空业、政府行业的数据表示标准，兼顾紧凑性、可读性、和数据结构（支持可选、多值等）丰富度等特性。
- xml 天生就是为复杂的数据表示而设计的，并且似乎可以解决数据表示的一起问题：可阅读性满足人的需求，dtd/schema 解决数据校验的问题，wsdl等解决掉服务描述的问题。而这在00年代也曾达到了一个高潮，所有的软件方案都是面向webservice设计，webservice成为Java、.NET的一等公民。JDK中应景而生的JAXB、JAX-WS等都是满足这个需求的。XML设计的一切是那么的完满，似乎可以解决一切的数据问题，成为宇宙通信的世界语。
- json：在xml达到顶峰之时，json作为掘墓人出现了。一反XML的包揽万象的特色，JSON简单直接，没有schema，没有wsdl，在功能上只能算是XML的一个子集，最大的优点就是简单。当然，天生的适合与javascript进行处理。然后，XML就走下神坛，导致现在，已经很难看到webservices的服务了，JAXB也计划从JDK Core中移除，xml也回归到一个基本的数据表示方式，而不再是包揽一切的世界语言了。
- ASN.1: https://en.wikipedia.org/wiki/Abstract_Syntax_Notation_One。 这是一个比较早的二进制数据表示语言，也为后续的二进制数据表示提供了参考。
- thrift: http://thrift.apache.org。由Facebook设计，目前已成为apache的一部分，是目前广泛应用的一个二进制协议，有广泛的语言绑定。thrift有丰富的数据类型：
  - 基础数据类型：bool, byte, i16, i32, i64, double, string
  - struct
  - 集合数据类型：list, set, collection
  thrift 支持 binary, compact binary 等模式，后者可以有更好的网络传输性能。thrift 使用tag来标记字段（这个做法事实上也不是thrift的原创，在tuxedo中就有类似的做法），理论上，thrift可以使用JSON、XML来传输数据（这在thrift的官方实现中确实包含了，不过有不少BUG，基本上不能真实使用）
- protobuf: https://github.com/google/protobuf Google内部的二进制协议，与thrift应该算是非常相似的一个设计，在性能上也不相伯仲。
- flatbuffers: https://github.com/google/flatbuffers。 这是一个非常有创意的二进制协议，thrift/protobuf的设计者有限考虑的是网络传输性能和数据序列化、反序列化性能，并尝试达到了极致。而flatbuffers则试图直接去除掉序列化/反序列化开销。在网络上传输的数据包就是语言可以直接进行random access的数据结构，这样带来的巨大优势就是没有反序列化，从而避免不必要的反序列化所需的内存开销和CPU开销。在具有高速内部网络的环境下，如果需要交换巨大数量的消息，flatbuffers应该是一个很有价值的二进制协议。

在设计dapeng的数据协议之时，还没有flatbuffer，作者亲自经历过上述的除flatbuffers之外的7种模式，首先在文本协议和二进制协议之间犹豫甚久。作为一名UNIX的爱好者，受UNIX的影响很深，也享受过文本流协议来带的各种便利，也同时感觉到：“性能”在大部分的应用场景中其实是一个伪命题，摩尔定律存在的目的就是让软件大量的浪费内存和CPU，要不根本不会有今天的互联网。在多次的犹豫之后，还是放弃了文本协议：主要是考虑到SOA应用需要提供对编程语言友好的数据表达能力，我们需要如下的特性：

- 数据结构丰富
- 强类型支持。强类型还是弱类型，也是两大江湖门派，各有所爱。作者个人还是更倾向于强类型多一些。
- 性能。虽说，大部分应用场景都不会在序列号上有极致的要求，选择JSON可能也能够满足。但dapeng的目标还是希望应对大型互联网项目、上千个服务节点、每天数亿级的服务请求（考虑到一个PV可能到来的大量内部服务调用），还是希望选择高效的二进制协议。

dapeng的最早设计是在2013年，当时thrift和protobuf都已经成熟，最终我们选择thrift作为我们的底层数据协议（2者差异不大，选A或B都没有太大差别）。虽然是基于thrift的协议模型，但是dapeng还是做了很多的改进：

- 重新设计 Java POJO 对象的映射模型。官方生成的POJO完全不是面向开发者的，一大堆杂乱无章的序列化相关的代码。（protobuf也是同样的风格，阅读protobuf生成的代码对人来说完全是一种折磨），考虑到 SOA 的接口模型中，数据对象是API最为核心的部分，我们希望生成的代码是符合开发者阅读体验的，所以我们将 POJO 和 序列化代码进行了隔离。
- 更好的 optional 映射。 null是Java中一个尴尬的存在，NPE则是服务开发者中的一个小噩梦。我们希望使用 Optional[T] 来强语意的表达 null。
- 扩展数据类型支持。最为常见的，是对 Timestamp 和 BigDecimal 的支持。在商务类服务中，不支持这两个数据类型，是不完备的。

随着webserivces的谢幕，restful的崛起，JSON逐步成为互联网应用中霸主级的存在，使用thrift虽然可以在性能上碾压对手，但在开发便利性、跨语言支持上，却存在不足。如何完美的支持JSON、restful的应用，也是dapeng需要面对的问题。dapeng在设计之初，就将服务的元信息(metadata)作为重要的基础设施，基于metadata，我们可以在动态、高效中达成尽可能的平衡。基于metadata，我们可以完成从json <-> thrift binary的转换。这里的metadata相对与xml中的xsd，相当于java object的反射的Class对象。

关于dapeng中如何高效的进行JSON的处理，可以参考： [dapeng fast json process](http://note.youdao.com/groupshare/?token=AEA5697785494A30BE9D62F5B39DE593&gid=40340366) 一章。

# dapeng 应用层/会话层协议设计

## 请求/返回结构
```
i32  frameLength
byte[frameLength-4]  frameData
    byte stx = 0x02
    byte version = 0x01
    byte protocol
    i32  seqId
    byte[?]  header-thrift-struct
    byte[?]  body-thrift-struct
    byte  etx = 0x03
```
这里的i32/byte等都是采用非压缩的thrift二进制格式，其中 header-thrift-struct 由dapeng core定义，对应于 `com.github.dapeng.core.SoaHeader`, 包含了本次请求/回应的重要信息：包括serviceName, version, method，也包括了请求和返回的更多信息，这些信息是服务路由、服务限流、调用跟踪等诸多服务治理所需要的，详情在下面有介绍。

整个传输包采用带有包长度界定的方式，首先传输整个包的长度，然后是包的内容，包的内容中由以0x02开头，0x03结束，这样设计的优点：
- 可以高效的进行拆包、粘包的处理。在最外层可以不需要关系协议内部的具体内容，而简单的进行断包。
- 有简单的容错机制，如果包不是0x02开头，0x03结尾，则当前的会话出现了通信错乱，当前会话需要废弃。
 
version 字段目前值为 1, 这个字段是保留给后续进行版本升级的，在设计上，新版本的dapeng应该有能力兼容旧版本的协议。如果version升级，version后的内容将不需要沿用目前的版本，可以灵活扩展。

seqId 是当前请求/返回的序列号，在用一个会话（TCP连接）上，可以同时发送多个请求，而无需等待前面的请求完成，才能发送后面的请求，这一般称之为多路复用。采用多路复用技术后，可以大大提高客户端是多线程环境下的请求处理能力，不再需要每一个线程创建一个TCP连接，而是大家可以共享一个连接。这种模式在异步编程中也是非常有必要的。在多路复用时，seqId用来标注当前的返回所对应的是哪一个请求。

采用上述的包结构，也设定了dapeng不是面向流的，而是面向 request - response的。所以，目前来说，dapeng 不适合流处理，比如说一次请求，后续会逐步的收到多个数据返回的情况。

## SoaHeader
SoaHeader是dapeng服务调用的一个核心数据结构，用于传输在应用层（方法参数）之外的参数，传统的thrift调用是没有这一块的开销的，dapeng引入了这个新的结构体，带来了额外的开销，也带来了丰富的额外功能，这些构成了服务化治理的各种收益。
```
1 optional string serviceName
2 optional string methodName
3 optional string version
4 optional string callerMid
5 optional string callerIp
6 optional i32 callerPort
7 optional string sessionTid
8 optional string userIp
9 optional string callerTid
10 optional i32 timeout
11 optional string respCode
12 optional string respMessage
13 optional string calleeTid
14 optional string calleeIp
15 optional i64 operatorId
16 optional i32 calleePort
17 optional i64 userId
18 optional string calleeMid
19 optional i32 transactionId
20 optional i32 transactionSequence
21 optional i32 calleeTime1
22 optional i32 calleeTime2
23 map<string,string> cookie
```
- serviceName, version 对请求包来说是必须的，对返回包来说是不必须的。dapeng使用service:version来定位服务，同一个服务可以有多个版本(major.minor.patch):
  - major 大版本。当大版本号升级时，表示新服务版本在API上出现重大变更，并对当前版本不兼容。即3.y.z的服务不兼容2.y.z的服务。在同一个大版本下，服务需要提供向下兼容，即1.1.10应该兼容1.1.9，也应该兼容1.0.12。
  - minor 小版本。当API升级但同时兼容当前的版本时，可以增加minor版本号，如：
     - 新增新的方法
     - 在请求结构体中新增可选的字段
     - 在返回结构体中新增字段
     - 不改变现有结构体的字段tag值，及数据类型(未来版本可能会支持从byte,i16,i32到i16,i32,i64的升级)
     开发人员可以使用dapeng的命令行工具来检查服务的版本兼容性，防止在破坏了兼容性的情况下，没有正确同步升级服务版本。
  - patch 一般的，在API不发生变化的情况下，但是对服务的实现进行了升级，或者进行了bug修复。这时，可以升级patch版本号。
- method 一个服务可以包含一组method，method是最小的可调用的服务单元。
- 客户端信息相关信息
  - callerMid: 调用者ModuleId。mid用来标识一个分布式应用中的一个参与者，包括：
    - web前端
    - 服务。
    - 定时任务
    - 脚本或工具
    - 消息消费者
    通过为每一个参与者分配一个mid，可以在调用链跟踪系统中统计，监控这些参与者之间的调用情况，包括调用次数，服务时间，成功率等指标
  - callerIp：服务调用者IP
  - callerPort：服务调用者Port
  - userIp：用于跟踪最原始的userIp，这个值是在服务调用链传递的，可用于反馈最初发起服务的user所在的浏览器IP。
  - operatorId：内部操作员ID
  - userId：外部用户ID
- 服务端相关信息
  - calleeMid
  - calleeIp
  - calleePort
  - respCode
  - respMessage
  - calleeTime1
  - calleeTime2
- 线索相关信息：这一块在服务调用跟踪中详细介绍
  - sessionTid
  - callerTid
  - calleeTid
- 分布式事务：在目前的dapeng 2.0版本中暂不支持。
  - transactionId
  - transactionSequence
- cookie: 在整个服务调用链中是传输的。后续版本还会支持服务端可以返回新的cookie给到客户端。