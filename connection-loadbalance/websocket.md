# 概述

一个服务存在多个实例时，`WebSocket`通过网关会被负载均衡连接到其中任意一个实例上。

而当一个服务实例发送消息时，连接另一个实例的客户端就会收不到消息

为了解决这个问题，该库提供了一种解决方案，开箱即用

只需要添加一个配置注解，就可以像单体应用一样使用`WebSocket`

也可以通过自定义来支持更复杂的业务

本库同时兼容`Webmvc`和`Webflux`，使用方式上没有任何区别

# 最新版本

![Maven Central](https://img.shields.io/maven-central/v/com.github.linyuzai/concept-websocket-loadbalance-spring-boot-starter)

# 集成

```gradle
implementation 'com.github.linyuzai:concept-websocket-loadbalance-spring-boot-starter:${version}'

implementation 'org.springframework.boot:spring-boot-starter-websocket'//webmvc需要添加websocket依赖，webflux不需要
```

```xml
<dependency>
    <groupId>com.github.linyuzai</groupId>
    <artifactId>concept-websocket-loadbalance-spring-boot-starter</artifactId>
    <version>${version}</version>
</dependency>

<!--webmvc需要添加websocket依赖，webflux不需要-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

# 使用

在启动类上添加注解`@EnableWebSocketLoadBalanceConcept`启用功能

```java
@EnableWebSocketLoadBalanceConcept
@SpringBootApplication
public class WsServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(WsServiceApplication.class, args);
    }
}
```

注入`WebSocketLoadBalanceConcept`即可跨实例发送消息

```java
@RestController
@RequestMapping("/ws")
public class WsController {

    @Autowired
    private WebSocketLoadBalanceConcept concept;

    @RequestMapping("/send")
    public void send(@RequestParam String msg) {
        concept.send(msg);
    }
}
```

客户端的连接地址为`ws(s)://{服务的地址}/concept-websocket/{自定义路径}`

其中`concept-websocket`为默认的前缀，可在配置中自定义（JAVAX不支持自定义）

# 配置属性

```yaml
concept:
  websocket:
    type: auto #JAVAX/SERVLET/REACTIVE，AUTO自动适配，默认AUTO
    server: #服务配置
      default-endpoint: #默认端点
        enabled: true #是否启用默认端点，默认true
        prefix: concept-websocket #前缀，默认'/concept-websocket/'
        path-selector: #Path选择器
          enabled: false #是否启用Path选择器，默认false
        user-selector: #User选择器
          enabled: false #是否启用User选择器，默认false
      message:
        retry:
          times: 0 #客户端重试次数，默认不重试
          period: 0 #客户端重试间隔，单位ms，默认0ms
      heartbeat: #心跳配置
        enabled: true #是否启用心跳，默认true
        period: 60000 #心跳间隔，单位ms，默认1分钟
        timeout: 210000 #超时时间，单位ms，默认3.5分钟，3次心跳间隔
    load-balance: #负载均衡（转发）配置
      subscriber-master: websocket #主订阅器，默认 websocket
      subscriber-slave: none #从订阅器，默认无
      message:
        retry:
          times: 0 #转发重试次数，默认不重试
          period: 0 #转发重试间隔，单位ms，默认0ms
      monitor: #监控配置
        enabled: true #是否启用监控，默认true
        period: 30000 #轮训间隔，单位ms，默认30s
        logger: false #是否启用日志，默认false
      heartbeat: #心跳配置
        enabled: true #是否启用心跳，默认true
        period: 60000 #心跳间隔，单位ms，默认1分钟
        timeout: 210000 #超时时间，单位ms，默认3.5分钟，3次心跳间隔
    executor:
      thread-pool-size: 1 #线程池大小，默认1
```

# 原理

通过服务间的`websocket`连接或是`Redis`和`MQ`等中间件转发消息

# 连接类型

|类型|说明|
|-|-|
|Client|普通客户端|
|Subscriber|订阅其他的服务消息的连接，该类型连接接收到的消息需要被转发|
|Observable|其他服务监听自身消息的连接，发送消息时需要转发消息到该类型的连接|

# 连接域

由于本库支持多种连接（当前包括`WebSocket`和`Netty`）同时配置，所以引入连接域来进行限制。

在自定义组件时需要指定该组件所适配的连接类型（`NettyScoped.NAME/WebSocketScoped.NAME`）

可通过重写`boolean support(String scope)`方法或是调用`addScopes(String... scopes)`来配置

### 事件监听器

可以实现`WebSocketEventListener`来监听事件

### 生命周期监听器

可以实现`WebSocketLifecycleListener`来监听生命周期（连接发布/连接关闭）

# 主从订阅

可以配置2种订阅转发方式提高容错

如以`Kafka`为主，`Redis`为从

当`Kafka`转发消息失败后，会切换到`Redis`重新转发，并开始对`Kafka`进行定时的的心跳检测

等到`Kafka`心跳检测正常，则重新切回到`Kafka`

抛出`MessageTransportException`将会触发切换

### 配置

```yaml
concept:
  websocket:
    load-balance:
      subscriber-master: websocket #主订阅者器，默认 websocket
      subscriber-slave: none #从订阅器，默认无
```

### 枚举

|配置|说明|
|-|-|
|websocket|每个服务实例ws双向连接，只能配置为主订阅器，且不支持主从切换|
|websocket_ssl|每个服务实例wss双向连接，只能配置为主订阅器，且不支持主从切换|
|kafka_topic|kafka转发|
|rabbit_fanout|rabbit转发|
|redis_topic|redis发布订阅|
|redis_topic_reactive|redis发布订阅|
|redisson_topic|redisson发布订阅|
|redisson_topic_reactive|redisson发布订阅|
|redisson_shared_topic|redisson发布订阅|
|redisson_shared_topic_reactive|redisson发布订阅|
|none|不转发|

# 幂等转发

通过`Kafka/RabbitMQ`转发消息时可能会重复消费

提供`MessageIdempotentVerifier`抽象`messageId`生成以及验证是否重复

默认缓存所有`messageId`在内存中（存在缓存越来越大的问题，建议自定义使用`Redis`或数据库存储）

可通过`MessageIdempotentVerifierFactory`自定义并注入`Spring`生效

# 消息发送

可通过自定义`ConnectionSelector`来实现消息的准确发送，如发送给某个路径或带有某个参数的连接

### 给指定路径的客户端发送消息

假设前端连接的`WebSocket`地址为`ws://localhost:8080/concept-websocket/sample`，`sample`为我们自定义路径

在配置中启用路径选择器

```yaml
concept:
  websocket:
    server: 
      default-endpoint: 
        path-selector: 
          enabled: true #启用Path选择器
```

使用`PathMessage`给所有的`sample`客户端发送消息

```java
@RestController
@RequestMapping("/ws")
public class WsController {

    @Autowired
    private WebSocketLoadBalanceConcept concept;

    @RequestMapping("/send-path")
    public void sendPath(@RequestParam String msg) {
        concept.send(new PathMessage(msg, "sample"));
    }
}
```

### 给指定用户发送消息

假设前端连接的`WebSocket`地址为`ws://localhost:8080/concept-websocket/user?userId=1`

其中`userId`为固定参数名

在配置中启用用户选择器

```yaml
concept:
  websocket:
    server: 
      default-endpoint: 
        user-selector: 
          enabled: true #启用用户选择器
```

使用`UserMessage`给指定的用户发送消息

```java
@RestController
@RequestMapping("/ws")
public class WsController {

    @Autowired
    private WebSocketLoadBalanceConcept concept;

    @RequestMapping("/send-user")
    public void sendUser(@RequestParam String msg) {
        concept.send(new UserMessage(msg, "1"));
    }
}
```

### 组合条件

如果既有路径条件，又有用户条件

```java
@RestController
@RequestMapping("/ws")
public class WsController {

    @Autowired
    private WebSocketLoadBalanceConcept concept;

    @RequestMapping("/send-path-user")
    public void sendUser(@RequestParam String msg) {
        Message message = new ObjectMessage(msg);
        PathMessage.condition("sample").apply(message);
        UserMessage.condition("1").apply(message);
        concept.send(message);
    }
}
```

### 选择过滤器

默认情况下，`ConnectionSelector`对于每次发送消息都只会有一个生效（只能生效一种过滤条件）

对于`ConnectionSelector`提供扩展`FilterConnectionSelector`

将会作为一个过滤器来支持多种条件，即[组合条件](#组合条件)模式

### PooledMessage(Beta)

为解决同一个数据多次编码成相同的数据

提供`PooledMessage`来缓存编码后的数据

```java
ObjectMessage message = new ObjectMessage("消息数据");
PooledMessage pooled = PooledMessage.wrap(message);
concept.send(pooled);
```

# 消息接收

实现`WebSocketMessageHandler`来处理客户端发送的消息

# 编解码器

默认配置的编解码器都是转`JSON`

可以通过`(Abstract)MessageCodecAdapter`自定义

# 组件说明

所有组件均可自定义扩展（可能需要指定`Order`来保证自定义的组件生效）

### 连接仓库

`ConnectionRepository`用于缓存连接实例

默认使用`Map<String, Map<Object, Connection>> connections = new ConcurrentHashMap<>();`缓存在内存中

可自定义`ConnectionRepositoryFactory`注入容器生效

### 连接服务管理器

`ConnectionServerManager`用于获取其他服务实例信息（`ws`双向连接中使用）和自身服务信息

默认使用`DiscoveryClient`和`Registration`来获得信息

可自定义`ConnectionServerManagerFactory`注入容器生效

### 连接订阅器

`ConnectionSubscriber`用于订阅其他服务的消息

提供配置文件配置

可自定义`(Abstract)ConnectionSubscriberFactory`或`(Abstract)MasterSlaveConnectionSubscriberFactory`注入容器生效

### 连接工厂

`ConnectionFactory`用于扩展`Connection`（如`WebSocketConnection/NettyConnection`）

可自定义`ConnectionFactory`注入容器生效

### 连接选择器

`ConnectionSelector`用于在发送消息时选择发送给哪些连接

可自定义`ConnectionSelector`或`FilterConnectionSelector`注入容器生效

### 消息工厂

`MessageFactory`用于适配创建消息

可自定义`MessageFactory`注入容器生效

### 消息编解码适配器

`MessageCodecAdapter`用于适配各种[连接类型](#连接类型)的消息编码器和消息解码器

可自定义`(Abstract)MessageCodecAdapter`注入容器生效

### 消息重试策略

`MessageRetryStrategyAdapter`用于适配各种[连接类型](#连接类型)的消息重试策略

消息重试不对`ping`和`pong`生效

可自定义`(Abstract)MessageRetryStrategyAdapter`注入容器生效

### 消息幂等校验器

`MessageIdempotentVerifier`用于生成消息ID以及校验消息是否处理

可自定义`MessageIdempotentVerifierFactory`注入容器生效

### 执行器

`ScheduledExecutor`用于执行各种延时/定时任务，如心跳等

可自定义`ScheduledExecutorFactory`注入容器生效

### 日志

`ConnectionLogger`用于打印日志

默认使用`Spring`的日志库

可自定义`ConnectionLoggerFactory`注入容器生效

### 事件发布器

`ConnectionEventPublisher`用于发布事件

默认支持`@EventListener`

可自定义`ConnectionEventPublisherFactory`注入容器生效

### 事件监听器

`WebSocketEventListener`用于监听事件

# 事件

|事件|说明|
|-|-|
|`ConnectionLoadBalanceConceptInitializeEvent`|`ConnectionLoadBalanceConcept`初始化|
|`ConnectionLoadBalanceConceptDestroyEvent`|`ConnectionLoadBalanceConcept`销毁|
|`ConnectionEstablishEvent`|连接建立|
|`ConnectionCloseEvent`|连接关闭|
|`ConnectionCloseErrorEvent`|连接关闭异常|
|`ConnectionErrorEvent`|连接异常|
|`ConnectionSubscribeErrorEvent`|连接订阅异常|
|`MessagePrepareEvent`|消息准备|
|`MessageSendEvent`|消息发送|
|`MessageSendSuccessEvent`|消息发送成功|
|`MessageSendErrorEvent`|消息发送异常|
|`DeadMessageEvent`|当一个消息不会发送给任何一个连接|
|`MessageDecodeErrorEvent`|消息解码异常|
|`MessageForwardEvent`|消息转发|
|`MessageForwardErrorEvent`|消息转发异常|
|`MessageReceiveEvent`|消息接收|
|`MessageReceivePredicateErrorEvent`|消息接收断言异常|
|`MessageDiscardEvent`|消息丢弃|
|`MasterSlaveSwitchEvent`|主从切换|
|`MasterSlaveSwitchErrorEvent`|主从切换异常|
|`HeartbeatTimeoutEvent`|心跳超时|
|`EventPublishErrorEvent`|事件发布异常|
|`LoadBalanceMonitorEvent`|监控触发|
|`UnknownCloseEvent`|未知的连接关闭|
|`UnknownErrorEvent`|未知的连接异常|
|`UnknownMessageEvent`|未知的消息|

# 默认服务端点配置

可以自定义`DefaultEndpointCustomizer`来配置

`Servlet`环境下会回调`WebSocketHandlerRegistration`

`Reactive`环境下会回调`ReactiveWebSocketServerHandlerMapping`

# WebSocketClientFactory

`2.2.0`版本针对高版本`javax`变为`jakarta`提供`Servlet`方式的`WebSocketClient`工厂