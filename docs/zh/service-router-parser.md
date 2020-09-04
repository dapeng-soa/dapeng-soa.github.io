---
layout: doc
title: 路由规则解析器
---
## dapeng 服务路由
> 服务路由，通过配置一系列规则，然后解析这些规则，进行服务的过滤和路由。这里就涉及到了路由解析器。包括，词法分析,路由解析,动态变更规则等。

## 至上而下的路由解析规则


- 理解词法分析器的工作原理。
- 掌握利用状态转换图设计词法分析器的基本方法。
- 实现Micro语言的词法分析程序




路由规则需要定义，定义好之后，需要定每一条路由规则进行解析，能够解析出来让路由的时候看得懂的词法解析集,这样在进行服务路由时，可以根据特定的词法解析集进行逻辑处理和解析，达到路由的规则。

---
## 路由解析
解析包括两大内容:词法解析和语法解析。词法解析是根本，语法解析是将一个个词法解析而出的内容组合在一起


### 词法解析器

词法分析是整个路由解析的第一步，它需要将我们定义的各类路由规则进行预处理,去除无用的回车、换行、引号、标点等，将规则语言解析成一个一个单词，我们这里就叫它们为`token`，这些`token`可以分为如下几种:
- 关键字

其代表在路由规则中,代表特定含义的单词，这写单词通常不能够被随便定义。包括`THEN`、`OTHERWISE`、`MATCH`、`NOT`等(更多关于这些关键字解析见下文)

- 标识符

这类一般表示规则中的一些运算符,文件符等，其目的在于对单词进行修饰，以及对整个规则进行合理的分割和定义。这类包括:`EOL`（文件换行符）、`NOT`（取反）、 `REGEXP`（正则）、 `EOF`（文件结束符）、 `SEMI_COLON`（分号）、 `COMMA`（逗号） 等

- 规则ID

这是路由规则中由用户根据需要进行自定义的一类单词。如现在dapeng的路由解析可以支持对服务名，方法名，IP,COOKIES等进行路由。比如，对服务某个方法进行路由,就需要使用的规则ID为method，这类通常包括:
method,service,version,userId,ip等

#### token 解析

路由规则每一个单词默认以空格隔开，多条路由规则之间以换行符分割。例如下面的两条路由规则:
```
method match "sayHello" => ip"192.168.12.2"
method match "hello" => ip"192.168.10.2"
```
上面为两个路由规则，以换行符进行分割。词法解析器将会从左至右逐个进行解析.
- 1.开始解析，知道碰到空格以后停止，然后拿到解析到的单词
- 2.将解析出来的单词进行分类,包装为对应的token,下面是解析出来的单词与之对应的封装的token
    - 换行符(\n)  ---> Token_EOL
    - =>           --->      Token_THEN 
    - otherwise---> Token_OTHERWISE 
    - match 关键字---> Token_MATCH 
    - not(~)---> Token_NOT 
    - 文件结束符---> Token_EOF 
    - 分号---> Token_SEMI_COLON 
    - 逗号---> Token_COMMA 
- 3.包装为对应的token后进行返回


词法解析器将路由规则从左至右解析并封装为一个一个token以后，就交给语法解析器进行语法解析。到此为止，词法解析的任务完成。

### 语法解析器
> 自上而下的语法解析器

前面提到了词法解析，词法解析是为语法解析作铺垫。我们最终是将一条路由规则解析为程序可以理解的意思。下面我们定义，最终将一条规则解析出来的结果封装为一个`Route`对象，多条路由规则将会解析为一个`List<Route>`
下面是`Router`对象
```java
/**
 * 描述: 每一条路由表达式解析出来的内容
 *
 * @author hz.lei
 * @since 2018年04月13日 下午9:45
 */
public class Route {

    private Condition left;
    private List<ThenIp> thenRouteIps;

    public Route(Condition left, List<ThenIp> thenRouteIps) {
        this.left = left;
        this.thenRouteIps = thenRouteIps;
    }

    public Condition getLeft() {
        return left;
    }

    public List<ThenIp> getThenRouteIps() {
        return thenRouteIps;
    }

    @Override
    public String toString() {
        return "Route{left=" + left +", thenRouteIps:" + thenRouteIps +'}';
    }
}
```
Route代表路由规则解析出来的内容。
例如:
```
method match "sayHello" => ip"192.168.101.1"
```

左边为`Condition` 右边 为路由到指定的ip
一个路由规则会被解析为一个Route对象，Router对象里面包含Condition和ThenIp的List

- Condition 路由成立的条件，当左边条件为true时，即会指向右边路由的Ip
- ThenIp
条件成立后指向的Ip地址封装类，路由对象Route包含的时一个List的ThenIp,表明条件成立是可以配置路由到多个Ip的。

##### 解析`Condition`

Condition 包含Matchers和Otherwise
Matcher是一个匹配规则的包装，表示该路由规则最终匹配成功与否。
ortherwise表示所有的规则都能匹配，一般放在上一条规则后面，如果某个请求过来没有匹配到上一条规则，都会被Otherwise匹配到。

##### 解析`Matchers`
```java
public class Matcher {

    private String id;
    private List<Pattern> patterns;

    public Matcher(String id, List<Pattern> patterns) {
        this.id = id;
        this.patterns = patterns;
    }

    public String getId() {
        return id;
    }

    public List<Pattern> getPatterns() {
        return patterns;
    }

    @Override
    public String toString() {
        return "Matcher{" +
                "id='" + id + '\'' +
                ", patterns=" + patterns +
                '}';
    }
}
```
Matach 本质代表的就是一次请求的某些数据是否能够与路由规则匹配成功。
```
method match "setFoo"
```
比如，本次请求的服务方法是`hello()`,那么这次请求通过路由之后,并没有匹配上，Matcher会返回false.

#### Pattern
pattern是matcher中的条件，条件可能有多个，比如:
```

method match "sayHello","hello"
```
上面的匹配到的两个方法，就是两个Pattern



#### 自上而下的语法解析规则
语法解析，自上而下

##### 第一步 从一个整体的路由对象入手`Route`
我们最终的目的是要将多个路由规则解析成多个`Route`对象
所以，第一步,我们在如下方法中直接进行解析，解析返回多个`Route`对象
```java
public List<Route> routes() {
        List<Route> routes = new ArrayList<>();
        Token token = lexer.peek();
        switch (token.type()) {
            case Token.EOL:
            case Token.OTHERWISE:
            case Token.ID:
                Route route = route();
                if (route != null) {
                    routes.add(route);
                }
                while (lexer.peek() == Token_EOL) {
                    lexer.next(Token.EOL);
                    Route route1 = route();
                    if (route1 != null) {
                        routes.add(route1);
                    }
                }
                break;
            case Token.EOF:
                warn("current service hava no route express config");
                break;
            default:
                error("expect `otherwise` or `id match ...` but got " + token);
        }
        return routes;
    }

```
##### 第二步，在获取List<Route>的基础上获取单个Route
```java
public Route route() {
        Token token = lexer.peek();
        switch (token.type()) {
            case Token.OTHERWISE:
            case Token.ID:
                Condition left = left();
                lexer.next(Token.THEN);
                List<ThenIp> right = right();
                return new Route(left, right);
            default:
                warn("expect `otherwise` or `id match ...` but got " + token);
        }
        return null;
    }


```
##### 第三步 在获取Route时，需要先获取Route左边表达式`Matcher`和右边表达式`ThenIp`列表
因此在这个方法里，分别调用left和right方法进行解析:
- left
```
public Condition left() {
    Matchers matchers = new Matchers();
    Token token = lexer.peek();
    switch (token.type()) {
        case Token.OTHERWISE:
            lexer.next();
            return new Otherwise();
        case Token.ID:
            Matcher matcher = matcher();
            matchers.macthers.add(matcher);
            while (lexer.peek() == Token_SEMI_COLON) {
                lexer.next(Token.SEMI_COLON);
                matchers.macthers.add(matcher());
            }
            return matchers;
        default:
            error("expect `otherwise` or `id match ...` but got " + token);
            return null;
    }
}
```
在left方法里会继续解析多个Matcher，调用matcher进行解析:
```
public Matcher matcher() {

        // method
        IdToken id = (IdToken) lexer.next();
        // match
        lexer.next(Token.MATCH);
        List<Pattern> patterns = patterns();

        return new Matcher(id.name, patterns);
    }

```
matcher里又会先调用patterns去解析多个条件的情形.

像这样，在解析路由规则时，一层一层至上而下，逐级解析并包装到上一层的变量中。即灵活又简介。代码各方法各司其职，分别管理自己需要解析的部分，逐层封装,最终解析出Route对象。

这就是一种自上而下的语法解析的最佳实战
     
     
    
### 路由管理(使用)
> 前面提到了dapeng服务路由的设计原理和代码解析，现在我们开始如何使用服务路由呢

#### 1.例子
将某个方法的调用路由到指定的ip服务器(如:192.168.10.1)
现在这个服务部署在三个节点上，我们将请求`hello`的方法路由到上述服务器。
在zk的节点`/soa/router/com.today.XXXService`上写上下面的路由规则:
```
method match "hello" => ip"192.168.10.1"
```
该规则可以将请求路由指定到10.1这台服务器。
该配置可以动态的进行修改，修改后，新的服务路由将立即生效。

#### 2.如何在`zookeeper`上设置路由规则
- 1.可以使用dapeng配置中心直接进行配置(详见dapeng配置中心章节内容)
- 2.可以使用`dapeng-cli`命令行工具进行配置(详见dapeng-cli章节介绍)


#### 3.应用场景
##### 1.就近访问
在多个机房部署同一服务时，最佳的调用策略是访问最近的服务。可以根据Consumer的IP，配置就近访问的服务实例。

##### 2.黑名单
可以临时性的将某个服务实例加入黑名单，屏蔽对其的访问。

##### 3.读写分离
可以按方法区分读写方法，将读服务、写服务区分路由到不同的服务器上。

##### 4.轻重分离
可以根据服务的操作复杂度，区分为重服务（处理复杂）、轻服务（处理简单），将轻重服务分散到不同的服务器上。

##### 5.AB测试
可以发布一个B版本服务，将满足一定条件的用户（譬如按用户ID取模），路由到B版本，以测试B版本的效果。

##### 6.灰度发布
在发布新服务时，运维可以动态的调整灰度策略，逐步放量用户，使用新的版本的服务，直到系统完全稳定，再切换到新版本。

##### 7.服务降级。
在进行大促时，如果部分系统无法顶住压力，可以对部分服务进行降级（或者条件降级），从而降低整个系统的压力。

#### 4.路由具体功能

##### 按实现功能：
- 1.黑名单，可以将某个服务实例 ip 加入黑名单，屏蔽对其访问。 otherwise => ~ip"192.168.1.1"
- 2.读写分离 method match r"get.*" => ip"192.168.1.1"
		  method match r"set.*" => ip"192.168.1.2"

- 3.轻重服务，根据服务类名，将不同的服务路由到不同的服务器上。
- 4.根据用户id范围，将用户路由到不同的服务器上，可以做到灰度
- 5.服务降级，将指向某个服务的请求，直接路由到不可用ip，导致该服务无法使用，做到降级

##### 按路由策略：
- 1.支持根据 服务名(serviceName)，方法名(methodName)，版本号(versionName) 进行  路由。
- 2.支持上述服务名，方法名 模糊匹配(正则) 进行路由
- 3.根据 callIp 匹配指定 ip 规则 进行路由 
- 4.根据callId。整数范围 ，整数取模 进行路由
- 5.可以路由不匹配模式 ～
- 6.可以左边使用otherwise 全部匹配 并 路由到指定 ip
- 7.ip匹配支持掩码规则和正常精确匹配（不带掩码）

注：当一个服务配置了多个路由规则，最上面的匹配到的路由规则优先级最高。因此需要注意先后顺序。

各个路由规则，用回车换行符区分

#### 路由使用规则
##### 整体规则如下：
```
express1 ; express2 => ip-express
```

左边为匹配规则，以分号区分多个匹配规则，需要全部满足，才能匹配成功


##### express 
- `method match "getFoo" ,"setFoo"`
每一个子表达式形式如上，可以通过 逗号（,）匹配多个条件，这里条件只要满足其一即可


##### =>
=> 是 Then表达式，划分左边和右边，如果左边匹配成功，将指向右边的 ip 表达式


##### right ip表达式
- `ip"192.168.1.12" `
表示如果匹配成功，请求将路由到192.168.1.12的ip的服务节点上
- `ip支持掩码的匹配形式。`
如 ip"192.168.1.0/24" 可以路由到 “192.168.1.0”的一个范围。


#### 具体例子和功能


##### 1.匹配字符串模式的变量：
```
- method match "getSkuById" => ip"192.168.12.12"
```
作用：将方法为getSkuById的请求路由到

##### 2.正则表达形式：
可以通过正则的形式进行匹配，如下，可以将以get开头的请求路由到12的机器上，将set开头的请求路由到13的机器上。
```
method match r"get.*" => ip"192.168.12.12"
method match r"set.*" => ip"192.168.12.13"
```

##### 3.匹配请求ip地址，可以应用到黑名单
```
- calleeIp match ip'192.168.1.101' => ip"192.168.2.105/30"
```
表示，请求ip为'192.168.1.101'的请求 将会 路由到 192.168.2.105/30及其掩码的ip的服务实例中
```
- calleeIp match ip'192.168.1.101' => ip"0.0.0.0"
```
表示将请求为101的ip路由到无效的ip上，实现黑名单的功能


##### 4.可以根据请求用户的id进行路由。
#### 整数范围路由
```
- userId match 10..1000 => ip"192.168.12.1"
```
表示将请求用户id为10 到 1000 的用户 路由到 ip为192.168.12.1的服务实例

##### 5.取模路由
```
- userId match %“1024n+6” => ip"192.168.12.1"
```
表示将请求用户id与1024取模结果为6时，路由到 ip为192.168.12.1的服务实例
userId match %“1024n+3..5” => ip"192.168.12.1"
表示将请求用户id与1024取模结果为3到5之间时，路由到 ip为192.168.12.1的服务实例


##### 6.不匹配模式：
```
method match r"set.*" => ~ip"192.168.12.14" 
```
表示以set开头的方法将不会路由到 ip 为 192.168.12.14 的 服务实例


##### 7.otherwise 模式
```
otherwise => ip"192.168.12.12"
```
表示左侧所有都匹配，一般作为路由规则的最后一条执行，表示前面所有路由规则都不满足时，最后执行的路由规则

##### 8.多条件模式
```
method match r"set.*" ; version match "1.0.0" => ip'192.168.1.103' 
```

同时满足上述两个条件的请求，才会路由到右侧Ip的实例上


##### 9.多条件模式（情形二）

```
method match r"set.*",r"insert.*" => ip"192.123.12.11"
```
这种情形是，当请求的方法名为 set开头 或者 insert开头时都可以匹配成功，路由到右侧Ip


##### 10.路由多个Ip模式

```
serviceName match "com.today.service.MemberService" => ip"192.168.12.1",ip"192.168.12.2"
```
上述情形表示符合左边的条件，可以路由到上述右侧两个ip上


##### 11.多路由表达式
```
method match "setFoo" => ip"192.168.10.12/24"
method match "getFoo" => ip"192.168.12.14"
otherwise => ip"192.168.12.18"
```
上述情形为多个路由表达式写法，每个路由表达式 *换行分隔*


我们会从最上面一条路由表达式开始进行匹配，当匹配到时即停止，不在继续向下匹配。
如果没有匹配到，将继续向下进行解析。
如上，当前两条都不符合时，即可路由到第三条，otherwise表示所有都符合的规则，这样最终将会路由到"192.168.12.18"的ip上

#### 总结


