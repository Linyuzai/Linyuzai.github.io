# 概述

该库支持在一个项目中（同时）配置多个 `Kafka` 和 `RabbitMQ`

如有必要还可扩展 `ActiveMQ` 或 `RocketMQ` 等任何符合事件模型的组件

同时以简单的事件模型作为抽象，支持不对任何中间件强绑定的场景

支持可视化动态添加（未实现，考虑后续版本实现）

# 集成

```gradle
implementation 'com.github.linyuzai:concept-event-spring-boot-starter:1.1.1'
```

或者

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-event-spring-boot-starter</artifactId>
  <version>1.1.1</version>
</dependency>
```

在项目中使用`@EnableEventConcept`启用功能

```java
@EnableEventConcept
@SpringBootApplication
public class ConceptSampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConceptSampleApplication.class, args);
    }
}
```

# 配置

### 配置Kafka

```yaml
concept:
  event:
    kafka:
      enabled: true #需要手动开启
      endpoints: #在该节点下配置多个kafka，属性同spring.kafka
        kafka1: #端点名称-kafka1
          inherit: parent #继承名称为parent的端点配置
          bootstrap-servers:
            - 192.168.30.100:9092
            - 192.168.30.101:9092
            - 192.168.30.102:9092
          consumer:
            group-id: kafka1
        kafka2: #端点名称-kafka2
          inherit: parent #继承名称为parent的端点配置
          bootstrap-servers:
            - 192.168.60.200:9092
            - 192.168.60.201:9092
            - 192.168.60.202:9092
          consumer:
            group-id: kafka2
        parent: #作为其他端点的父配置
          enabled: false #是否启用该端点，这里作为父配置不启用
          producer:
            retries: 0
            acks: 1
          consumer:
            enable-auto-commit: false
          template:
            default-topic: sample
          listener:
            ack-mode: manual_immediate
```

`kafka`的配置属性同`spring.kafka`

### 配置RabbitMQ

```yaml
concept:
  event:
    rabbitmq:
      enabled: true #需要手动开启
      endpoints: #在该节点下配置多个rabbitmq，属性同spring.rabbitmq
        rabbitmq1: #端点名称-rabbitmq1
          inherit: parent #继承名称为parent的端点配置
          host: 192.168.30.140
          template:
            routing-key: rabbitmq1.dev
            exchange: rabbitmq1
        rabbitmq2: #端点名称-rabbitmq2
          inherit: parent #继承名称为parent的端点配置
          host: 192.168.30.141
          template:
            routing-key: rabbitmq2.dev
            exchange: rabbitmq2
        parent:
          enabled: false #是否启用该端点，这里作为父配置不启用
          username: admin
          password: 123456
          port: 5672
```

`rabbitmq`的配置属性同`spring.rabbitmq`

### 配置继承

额外提供配置继承可将一些相同的配置提取出来，使用`inherit`属性指定继承的端点

# 发布事件

### 简单方式

```java
@RestController
@RequestMapping("/concept-event")
public class EventController {

    @Autowired
    private EventConcept concept;

    @GetMapping("/send")
    public void send() {
        concept.template().publish(Object);//发布事件
    }
}
```

需要注意该方式需要提前通过 [事件引擎自定义配置](#事件引擎自定义配置)/[事件端点自定义配置](#事件端点自定义配置) 配置组件 [事件发布器](#事件发布器)/[事件订阅器](#事件订阅器)

### 自定义方式

```java
@RestController
@RequestMapping("/concept-event")
public class EventController {

    @Autowired
    private EventConcept concept;

    @GetMapping("/send")
    public void send() {
        concept.template()
                .context(KeyValue) //配置上下文（用于满足自定义数据传递）
                .exchange(EventExchange) //指定发布到哪些端点（多个Kafka中哪几个）
                .encoder(EventEncoder) //指定事件编码器（如把对象转成json）
                .error(EventErrorHandler) //指定异常处理器（发布异常的后续操作）
                .publisher(EventPublisher) //指定事件发布器（如使用KafkaTemplate给指定的Topic发消息）
                .publish(Object); //事件对象
    }
}
```

# 接收事件

### 简单方式

```java
@Configuration
public class EventSubscriberRegister implements ApplicationRunner {

    @Autowired
    public EventConcept concept;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        concept.template().subscribe(EventListener);//监听事件
    }
}
```

需要注意该方式需要提前通过 [事件引擎自定义配置](#事件引擎自定义配置)/[事件端点自定义配置](#事件端点自定义配置) 配置组件 [事件发布器](#事件发布器)/[事件订阅器](#事件订阅器)

### 自定义方式

```java
@Configuration
public class EventSubscriberRegister implements ApplicationRunner {

    @Autowired
    public EventConcept concept;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        concept.template()
                .context(KeyValue) //配置上下文（用于满足自定义数据传递）
                .exchange(EventExchange) //指定订阅哪些端点（多个Kafka中哪几个）
                .decoder(EventDecoder) //指定事件解码器（如把json转成对象）
                .error(EventErrorHandler) //指定异常处理器（订阅或消费异常的后续操作）
                .subscriber(EventSubscriber) //指定事件订阅器（如订阅哪个Topic）
                .subscribe(EventListener); //监听事件
    }
}
```

# 事件引擎

抽象为`EventEngine`

表示中间件类型，目前支持 `Kafka` 和 `RabbitMQ`

```java
//如有需要可以使用该方法获得Kafka事件引擎
KafkaEventEngine.get(EventConcept);

//如有需要可以使用该方法获得RabbitMQ事件引擎
RabbitEventEngine.get(EventConcept);
```

### 事件引擎工厂

抽象为`EventEngineFactory`

`Kakfa`事件引擎工厂`KafkaEventEngineFactory`，默认实现`KafkaEventEngineFactoryImpl`

可自定义并注入`Spring`容器生效

```java
@Component
public class CustomKafkaEventEngineFactory implements KafkaEventEngineFactory {

    @Override
    public KafkaEventEngine create(KafkaEventProperties config) {
        //自定义
        return null;
    }
}
```

`RabbitMQ`事件引擎工厂`RabbitEventEngineFactory`，默认实现`RabbitEventEngineFactoryImpl`

可自定义并注入`Spring`容器生效

```java
@Component
public class CustomRabbitEventEngineFactory implements RabbitEventEngineFactory {
    
    @Override
    public RabbitEventEngine create(RabbitEventProperties config) {
        //自定义
        return null;
    }
}
```

### 事件引擎自定义配置

抽象为`EventEngineConfigurer`

`Kakfa`使用`KafkaEventEngineConfigurer`扩展配置

可自定义并注入`Spring`容器生效

```java
@Configuration
public class CustomKafkaEventEngineConfigurer implements KafkaEventEngineConfigurer {

    @Override
    public void configure(KafkaEventEngine engine) {
        //自定义配置
    }
}
```

`RabbitMQ`使用`RabbitEventEngineConfigurer`扩展配置

可自定义并注入`Spring`容器生效

```java
@Configuration
public class CustomRabbitEventEngineConfigurer implements RabbitEventEngineConfigurer {
    
    @Override
    public void configure(RabbitEventEngine engine) {
        //自定义配置
    }
}
```

# 事件端点

抽象为`EventEndpoint`

表示多个`Kafka`和`RabbitMQ`服务（集群）

### 事件端点工厂

抽象为`EventEndpointFactory`

`Kakfa`事件引擎工厂`KafkaEventEndpointFactory`，默认实现`KafkaEventEndpointFactoryImpl`

可自定义并注入`Spring`容器生效

```java
@Component
public class CustomKafkaEventEndpointFactory implements KafkaEventEndpointFactory {
    
    @Override
    public KafkaEventEndpoint create(String name, KafkaEventProperties.ExtendedKafkaProperties config, KafkaEventEngine engine) {
        //自定义
        return null;
    }
}
```

`RabbitMQ`事件引擎工厂`RabbitEventEndpointFactory`，默认实现`RabbitEventEndpointFactoryImpl`

可自定义并注入`Spring`容器生效

```java
@Component
public class CustomRabbitEventEndpointFactory implements RabbitEventEndpointFactory {
    
    @Override
    public RabbitEventEndpoint create(String name, RabbitEventProperties.ExtendedRabbitProperties config, RabbitEventEngine engine) {
        //自定义
        return null;
    }
}
```

### 事件端点自定义配置

抽象为`EventEndpointConfigurer`

`Kakfa`使用`KafkaEventEndpointConfigurer`扩展配置

可自定义并注入`Spring`容器生效

```java
@Configuration
public class CustomKafkaEventEndpointConfigurer implements KafkaEventEndpointConfigurer {

    @Override
    public void configure(KafkaEventEndpoint endpoint) {
        //自定义配置
    }
}
```

`RabbitMQ`使用`RabbitEventEndpointConfigurer`扩展配置

可自定义并注入`Spring`容器生效

```java
@Configuration
public class CustomRabbitEventEndpointConfigurer implements RabbitEventEndpointConfigurer {
    
    @Override
    public void configure(RabbitEventEndpoint endpoint) {
        //自定义配置
    }
}
```

# 事件上下文

抽象为`EventContext`

用于事件发布和事件订阅的过程中的数据交互

同时方便用户自定义扩展处理

### 事件上下文工厂

抽象为`EventContextFactory`，默认实现`MapEventContextFactory`

可自定义并注入`Spring`容器生效

```java
@Component
public class CustomEventContextFactory implements EventContextFactory {
    
    @Override
    public EventContext create() {
        //自定义
        return null;
    }
}
```

# 事件交换机

抽象为`EventExchange`

用于在发布事件或订阅事件时指定对应的事件端点

可自定义并注入`Spring`容器全局生效

```java
@Component
public class CustomEventExchange implements EventExchange {
    
    @Override
    public Collection<? extends EventEndpoint> exchange(Collection<? extends EventEngine> engines, EventContext context) {
        //自定义筛选
        return null;
    }
}
```

手动指定优先级高于全局配置

默认提供的事件交换机

|事件交换机|说明|
|-|-|
|`EngineExchange`|指定一个或多个引擎下的所有端点|
|`EndpointExchange`|指定一个引擎下的一个或多个端点|
|`KafkaEngineExchange`|指定`Kafka`所有端点|
|`KafkaEndpointExchange`|指定`Kafka`一个或多个端点|
|`RabbitEngineExchange`|指定`RabbitMQ`所有端点|
|`RabbitEndpointExchange`|指定`RabbitMQ`一个或多个端点|
|`ComposeEventExchange`|组合多个事件交换机|

# 事件发布器

抽象为`EventPublisher`

用于指定事件的发布逻辑（一般调用`KafkaTemplate`或`RabbitTemplate`来发送消息）

可基于`KafkaEventPublisher`或`AbstractKafkaEventPublisher`和`RabbitEventPublisher`或`AbstractRabbitEventPublisher`自定义事件发布器

可通过 [事件引擎自定义配置](#事件引擎自定义配置)/[事件端点自定义配置](#事件端点自定义配置) 的方式配置

```java
@Configuration
public class CustomKafkaEventEngineConfigurer implements KafkaEventEngineConfigurer {

    @Override
    public void configure(KafkaEventEngine engine) {
        engine.setPublisher(new CustomKafkaEventPublisher());
    }
}
```

或者

```java
@Configuration
public class CustomKafkaEventEndpointConfigurer implements KafkaEventEndpointConfigurer {

    @Override
    public void configure(KafkaEventEndpoint endpoint) {
        endpoint.setPublisher(new CustomKafkaEventPublisher());
    }
}
```

手动指定的优先级高于事件端点中的配置

事件端点中的配置优先级高于事件引擎中的配置

默认提供的事件发布器

|事件发布器|说明|
|-|-|
|`TopicKafkaEventPublisher`|指定`Topic`的事件发布器|
|`ConfigurableKafkaEventPublisher`|可配置`Topic`，`Partition`，`Timestamp`，`Key`的事件发布器|
|`DefaultKafkaEventPublisher`|基于`KafkaTemplate#sendDefault`的事件发布器|
|`RoutingRabbitEventPublisher`|指定`exchange`和`routingKey`的事件发布器|
|`ConfigurableRabbitEventPublisher`|可配置`exchange`，`routingKey`和`correlationData`的事件发布器|
|`DefaultRabbitEventPublisher`|基于`RabbitTemplate#convertAndSend`的事件发布器|
|`ComposeEventPublisher`|组合多个事件发布器|

### RabbitMQ初始化

`AbstractRabbitEventPublisher`提供`#binding`方法在发布时创建`Exchange/Queue/Binding`，用法同`BindingBuilder`

```java
public class CustomRabbitEventPublisher extends AbstractRabbitEventPublisher {

    @Override
    public void send(Object event, RabbitEventEndpoint endpoint, EventContext context) {
        endpoint.getTemplate().convertAndSend(endpoint);
    }

    @Override
    public void binding(RabbitBinding binding) {
        //创建和绑定
        binding.bind(new Queue("queue"))
                .to(new TopicExchange("topic"))
                .with("routingKey");
    }
}
```

# 事件订阅器

抽象为`EventSubscriber`

用于指定事件的订阅逻辑

可基于`KafkaEventSubscriber`或`AbstractKafkaEventSubscriber`和`RabbitEventSubscriber`或`AbstractRabbitEventSubscriber`自定义事件订阅器

可通过 [事件引擎自定义配置](#事件引擎自定义配置)/[事件端点自定义配置](#事件端点自定义配置) 的方式配置

```java
@Configuration
public class CustomKafkaEventEngineConfigurer implements KafkaEventEngineConfigurer {

    @Override
    public void configure(KafkaEventEngine engine) {
        engine.setSubscriber(new CustomKafkaEventSubscriber());
    }
}
```

或者

```java
@Configuration
public class CustomKafkaEventEndpointConfigurer implements KafkaEventEndpointConfigurer {

    @Override
    public void configure(KafkaEventEndpoint endpoint) {
        endpoint.setSubscriber(new CustomKafkaEventSubscriber());
    }
}
```

手动指定的优先级高于事件端点中的配置

事件端点中的配置优先级高于事件引擎中的配置

默认提供的事件订阅器

|事件订阅器|说明|
|-|-|
|`TopicKafkaEventSubscriber`|指定`Topic`的事件订阅器|
|`TopicPatternKafkaEventSubscriber`|指定`Topic Pattern`的事件订阅器|
|`TopicPartitionOffsetKafkaEventSubscriber`|指定`TopicPartitionOffset`的事件订阅器|
|`DefaultKafkaEventSubscriber`|基于`KafkaListenerEndpoint`的事件订阅器|
|`QueueRabbitEventSubscriber`|指定`Queue`的事件订阅器|
|`DefaultRabbitEventSubscriber`|基于`RabbitListenerEndpoint`的事件订阅器|
|`ComposeEventSubscriber`|组合多个事件订阅器|

### 订阅句柄

抽象为`Subscription`

事件订阅器订阅之后会返回一个订阅句柄

可通过订阅句柄取消订阅`Subscription#unsubscribe()`

### RabbitMQ初始化

`AbstractRabbitEventSubscriber`提供`#binding`方法在订阅时创建`Exchange/Queue/Binding`，用法同`BindingBuilder`

```java
public class CustomRabbitEventSubscriber extends AbstractRabbitEventSubscriber {
    
    @Override
    public MessageListenerContainer createMessageListenerContainer(RabbitEventEndpoint endpoint, EventContext context, MessageListener messageListener) {
        return endpoint.getListenerContainerFactory().createListenerContainer();
    }

    @Override
    public void binding(RabbitBinding binding) {
        //创建和绑定
        binding.bind(new Queue("queue"))
                .to(new TopicExchange("topic"))
                .with("routingKey");
    }
}
```

# 事件编码器

抽象为`EventEncoder`

用于在事件发布时对事件进行编码，默认为`null`，不进行编码处理

可自定义并注入`Spring`容器全局生效

```java
@Configuration
public class EventConfig {

    @Bean
    public EventEncoder eventEncoder() {
        return new JacksonEventEncoder();
    }
}
```

也可通过 [事件引擎自定义配置](#事件引擎自定义配置)/[事件端点自定义配置](#事件端点自定义配置) 的方式配置

```java
@Configuration
public class CustomKafkaEventEngineConfigurer implements KafkaEventEngineConfigurer {

    @Override
    public void configure(KafkaEventEngine engine) {
        engine.setEncoder(new JacksonEventEncoder());
    }
}
```

或者

```java
@Configuration
public class CustomKafkaEventEndpointConfigurer implements KafkaEventEndpointConfigurer {

    @Override
    public void configure(KafkaEventEndpoint endpoint) {
        endpoint.setEncoder(new JacksonEventEncoder());
    }
}
```

手动指定的优先级高于事件端点中的配置

事件端点中的配置优先级高于事件引擎中的配置

事件引擎中的配置优先级高于全局配置

默认提供的事件编码器

|事件编码器|说明|
|-|-|
|`JacksonEventEncoder`|基于`Jackson`的`json`编码|
|`SerializationEventDecoder`|基于`jdk`序列化的编码|

# 事件解码器

抽象为`EventDecoder`

用于在监听到事件时对事件进行解码，默认为`null`，不进行解码处理

可自定义并注入`Spring`容器全局生效

```java
@Configuration
public class EventConfig {

    @Bean
    public EventDecoder eventDecoder() {
        return new JacksonEventDecoder();
    }
}
```

也可通过 [事件引擎自定义配置](#事件引擎自定义配置)/[事件端点自定义配置](#事件端点自定义配置) 的方式配置

```java
@Configuration
public class CustomKafkaEventEngineConfigurer implements KafkaEventEngineConfigurer {

    @Override
    public void configure(KafkaEventEngine engine) {
        engine.setDecoder(new JacksonEventDecoder());
    }
}
```

或者

```java
@Configuration
public class CustomKafkaEventEndpointConfigurer implements KafkaEventEndpointConfigurer {

    @Override
    public void configure(KafkaEventEndpoint endpoint) {
        endpoint.setDecoder(new JacksonEventDecoder());
    }
}
```

手动指定的优先级高于事件端点中的配置

事件端点中的配置优先级高于事件引擎中的配置

事件引擎中的配置优先级高于全局配置

默认提供的事件解码器

|事件解码器|说明|
|-|-|
|`JacksonEventDecoder`|基于`Jackson`的`json`解码|
|`SerializationEventDecoder`|基于`jdk`序列化的解码|

# 事件监听器

抽象为`EventListener`

用于在事件订阅时监听数据

`JacksonEventDecoder`可结合`#getType()`方法解析数据

提供`GenericEventListener<T>`自动提取泛型作为`#getType()`返回值

```java
@Configuration
public class KafkaEventSubscriberRegister implements ApplicationRunner {

    @Autowired
    public EventConcept concept;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        concept.template().subscribe(new GenericEventListener<KafkaData>() {

            @Override
            public void onGenericEvent(KafkaData event, EventEndpoint endpoint, EventContext context) {
                System.out.println(event);
            }
        });
    }
}
```

# 异常处理器

抽象为`EventErrorHandler`

用于发布或订阅时的异常处理，默认实现`LoggerEventErrorHandler`

可自定义并注入`Spring`容器全局生效

```java
@Component
public class CustomEventErrorHandler implements EventErrorHandler {

    @Override
    public void onError(Throwable e, EventEndpoint endpoint, EventContext context) {
        //自定义异常处理
    }
}
```

也可通过 [事件引擎自定义配置](#事件引擎自定义配置)/[事件端点自定义配置](#事件端点自定义配置) 的方式配置

```java
@Configuration
public class CustomKafkaEventEngineConfigurer implements KafkaEventEngineConfigurer {

    @Override
    public void configure(KafkaEventEngine engine) {
        engine.setErrorHandler(new CustomEventErrorHandler());
    }
}
```

或者

```java
@Configuration
public class CustomKafkaEventEndpointConfigurer implements KafkaEventEndpointConfigurer {

    @Override
    public void configure(KafkaEventEndpoint endpoint) {
        endpoint.setErrorHandler(new CustomEventErrorHandler());
    }
}
```

手动指定的优先级高于事件端点中的配置

事件端点中的配置优先级高于事件引擎中的配置

事件引擎中的配置优先级高于全局配置

# 事件模版

抽象为`EventTemplate`

用于不同场景的配置复用

可以持有事件模版直接进行发布事件

```java
@RestController
public class Business1Controller {

    private final EventTemplate template;

    @Autowired
    public Business1Controller(EventConcept concept) {
        this.template = concept.template()
                .exchange(new Business1EventExchange())
                .publisher(new Business1EventPublisher());
    }

    @PostMapping("/business1")
    public void business1() {
        template.publish(new Business1Event());
    }
}
```

# 配置继承处理器

抽象为`ConfigInheritHandler`

`Kafka`配置继承处理器`KafkaConfigInheritHandler`，默认实现`ReflectionKafkaConfigInheritHandler`

可自定义并注入`Spring`容器生效

```java
@Component
public class CustomKafkaConfigInheritHandler implements KafkaConfigInheritHandler {
    
    @Override
    public void inherit(KafkaEventProperties config) {
        
    }
}
```

`RabbitMQ`配置继承处理器`RabbitConfigInheritHandler`，默认实现`ReflectionRabbitConfigInheritHandler`

可自定义并注入`Spring`容器生效

```java
@Component
public class CustomRabbitConfigInheritHandler implements RabbitConfigInheritHandler {
    
    @Override
    public void inherit(RabbitEventProperties config) {
        
    }
}
```

# 生命周期监听器

通过实现`EventConceptLifecycleListener`接口可以在`EventConcept`初始化和销毁时扩展额外的逻辑

```java
@Component
public class CustomEventConceptLifecycleListener implements EventConceptLifecycleListener {
    
    @Override
    public void onInitialize(EventConcept concept) {
        //初始化
    }

    @Override
    public void onDestroy(EventConcept concept) {
        //销毁
    }
}
```

# 原生支持

### KafkaTemplate

可通过指定`@Qualifier`获取，名称为：`${事件端点名称}KafkaTemplate`

```java
@RequestMapping("/concept-event/kafka")
public class KafkaEventController {

    @Autowired
    @Qualifier("devKafkaTemplate")
    private KafkaTemplate<Object, Object> kafkaTemplate;
}
```

### KafkaListener

可通过指定`containerFactory`使用，名称为：`${事件端点名称}KafkaListenerContainerFactory`

```java
@Configuration
public class KafkaEventSubscriberRegister {

    @KafkaListener(topics = "sample", containerFactory = "devKafkaListenerContainerFactory")
    public void receiveLocal(String msg, Acknowledgment acknowledgment) {
        System.out.println("dev-" + msg);
        acknowledgment.acknowledge();
    }
}
```

### RabbitTemplate

可通过指定`@Qualifier`获取，名称为：`${事件端点名称}KafkaTemplate`

```java
@RequestMapping("/concept-event/rabbit")
public class RabbitEventController {

    @Autowired
    @Qualifier("devRabbitTemplate")
    private RabbitTemplate rabbitTemplate;
}
```

### RabbitListener

可通过指定`containerFactory`使用，名称为：`${事件端点名称}RabbitListenerContainerFactory`

```java
@Configuration
public class KafkaEventSubscriberRegister {

    @SneakyThrows
    @RabbitListener(queues = "queue", containerFactory = "devRabbitListenerContainerFactory")
    public void receiveDev(Message message, Channel channel) {
        System.out.println("dev-" + new String(message.getBody()));
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    }
}
```

# Bean名称

下列`Bean`全部使用`ConfigurableBeanFactory#registerSingleton`注册

- Kafka

|Bean|Name|
|-|-|
|`ProducerFactory`|`${事件端点名称}KafkaProducerFactory`|
|`ProducerListener`|`${事件端点名称}KafkaProducerListener`|
|`KafkaTemplate`|`${事件端点名称}KafkaTemplate`|
|`ConsumerFactory`|`${事件端点名称}KafkaConsumerFactory`|
|`KafkaTransactionManager`|`${事件端点名称}KafkaTransactionManager`|
|`KafkaListenerContainerFactory`|`${事件端点名称}KafkaListenerContainerFactory`|
|`KafkaAdmin`|`${事件端点名称}KafkaAdmin`|
|`KafkaEventEndpoint`|`${事件端点名称}KafkaEventEndpoint`|

- RabbitMQ

|Bean|Name|
|-|-|
|`ConnectionFactory`|`${事件端点名称}RabbitConnectionFactory`|
|`RabbitListenerContainerFactory`|`${事件端点名称}RabbitListenerContainerFactory`|
|`RabbitTemplate`|`${事件端点名称}RabbitTemplate`|
|`RabbitAdmin`|`${事件端点名称}RabbitAdmin`|
|`RabbitEventEndpoint`|`${事件端点名称}RabbitEventEndpoint`|

# 流程图

![event-config](https://user-images.githubusercontent.com/18523183/184789920-f86cfef8-3a2c-4ce6-8a14-c6c289b8cbb6.svg)

![event-publish](https://user-images.githubusercontent.com/18523183/184793482-d5f5a6d4-77fd-4882-9e17-a8d84c5dfd7f.svg)

![event-subscribe](https://user-images.githubusercontent.com/18523183/184804217-4b336881-a0f2-495a-bfa1-a7df389bf615.svg)

# 版本

### 列表

##### 1.1.0

- 代码结构优化

##### 1.1.1

- `InheritHandler`改为`ConfigInheritHandler`
- 漏注册`RabbitEventEndpoint`的问题
