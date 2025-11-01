# 概述

轻量级插件化解决方案

提供可插拔的插件能力

# 最新版本

![Maven Central](https://img.shields.io/maven-central/v/com.github.linyuzai/concept-plugin-spring-boot-starter)

# 3.x.x 新特性

注意事项：3.x.x版本与1.x.x和2.x.x版本不兼容（`PluginExtractor`和`@OnPluginExtract`可直接升级，自定义组件可能存在不兼容）

1. 支持集群环境，支持对象存储加载插件

2. 支持动态`Spring`接口，支持解析`yaml`文件

3. 提供更简单的集成方式

4. 架构优化及bug修复

5. 移除`FILE`加载模式

# 集成

```gradle
implementation 'com.github.linyuzai:concept-plugin-spring-boot-starter:${version}'
```

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-plugin-spring-boot-starter</artifactId>
  <version>${version}</version>
</dependency>
```

# 使用

`webmvc`和`webflux`在使用上无任何区别

### 1. 在启动类上添加注解 @EnablePluginConcept 启用功能

```java
@EnablePluginConcept
@SpringBootApplication
public class PluginApplication {

    public static void main(String[] args) {
        SpringApplication.run(PluginApplication, args);
    }
}
```

### 2. 配置插件提取（按需选择，示例如下）

#### PluginObservable（方式一）

```java
@Configuration
public class AIConfig {

    @Bean
    public PluginObservable<String, AI> aiPluginObservable() {
        return new GenericPluginObservable<String, AI>() {
            @Override
            public String grouping(AI plugin, PluginContext context) {
                return plugin.getName();
            }
        };
    }
}
```

```java
@RestController
@RequestMapping("/ai")
public class AIController {

    @Autowired
    private PluginObservable<String, AI> aiPluginObservable;

    @RequestMapping("/{name}")
    public void ai(@PathVariable("name") String name) {
        AI ai = aiPluginObservable.get(name);
        if (ai == null) {
            System.out.println("AI not found: " + name);
        } else {
            ai.ai();
        }
    }
}
```

注入`PluginObservable`并通过范型定义`key`和插件类

在需要使用插件的地方注入`PluginObservable`即可使用

`PluginObservable`将会自动感知插件加载和卸载

#### 实现提取器（方式二）

- 类提取器

>泛型可指定
>
>Class<? extends CustomPlugin>
>
>Class<? extends CustomPlugin>[]
>
>Collection<Class<? extends CustomPlugin>>
>
>List<Class<? extends CustomPlugin>>
>
>Set<Class<? extends CustomPlugin>>
>
>Map<Object, Class<? extends CustomPlugin>>
>

```java
public class SampleClassExtractor extends ClassExtractor<List<Class<? extends CustomPlugin>>> {

    @Override
    public void onExtract(List<Class<? extends CustomPlugin>> plugins, PluginContext context) {
        
    }
}
```

- 实例提取器

>泛型可指定
>
>CustomPlugin
>
>CustomPlugin[]
>
>Collection<(? extends )CustomPlugin>
>
>List<(? extends )CustomPlugin>
>
>Set<(? extends )CustomPlugin>
>
>Map<Object, (? extends )CustomPlugin>
>

```java
@Component
public class SampleBeanExtractor extends BeanExtractor<List<? extends CustomPlugin>> {

    @Override
    public void onExtract(List<? extends CustomPlugin> plugins, PluginContext context) {
        
    }
}
```

- 配置文件提取器

>泛型可指定
>
>Properties
>
>Properties[]
>
>Collection<(? extends )Properties>
>
>List<(? extends )Properties>
>
>Set<(? extends )Properties>
>
>Map<Object, (? extends )Properties>
>

```java
@Component
public class SamplePropertiesExtractor extends PropertiesExtractor<List<Properties>> {

    @Override
    public void onExtract(List<Properties> plugins, PluginContext context) {
        
    }
}
```

- 内容提取器

>泛型可指定
>
>String，InputStream，byte[]，ByteBuffer
>
>String[]，InputStream[]，byte[][]，ByteBuffer[]
>
>Collection<(? extends )String>，Collection<(? extends )InputStream>，Collection<byte[]>，Collection<(? extends )ByteBuffer>
>
>List<(? extends )String>，List<(? extends )InputStream>，List<byte[]>，List<(? extends )ByteBuffer>
>
>Set<(? extends )String>，Set<(? extends )InputStream>，Set<byte[]>，Set<(? extends )ByteBuffer>
>
>Map<Object, (? extends )String>，Map<Object, (? extends )InputStream>，Map<Object, byte[]>，Map<Object, (? extends )ByteBuffer>
>

```java
@Component
public class SampleContentExtractor extends ContentExtractor<List<String>> {

    @Override
    public void onExtract(List<String> plugins, PluginContext context) {
        
    }
}
```

#### 方法回调（方式三）

> 在方法上标记 @OnPluginExtract
>
> 定义需要提取的插件内容作为参数
>
> 需要能被 Spring 扫描到
>
> 任意一个参数不为 null 都会触发回调
>
> 可选参数 Plugin、PluginContext

```java
@Component
public class SampleDynamicExtractor {

    @OnPluginExtract
    public void sampleExtract(List<Class<? extends CustomPlugin>> classes,
                              Set<? extends CustomPlugin> beans,
                              Properties properties,
                              Plugin plugin, PluginContext context) {
    }
}
```

#### Spring接口（方式四）

配置文件启用扩展功能

```yml
concept:
  plugin:
    extension:
      request-mapping:
        enabled: true
```

### 3. 加载插件

加载`jar`或`zip`，支持`jar in jar`或多个`jar`打包成`zip`（视为一个插件）

通过 `/concept-plugin/management.html` 管理页面上传加载插件

<img width="1233" alt="plugin" src="https://github.com/user-attachments/assets/9d5d22b6-a436-4b60-a49f-03e410f7e341">

需要开放`/concept-plugin/**`路径，或者[扩展管理页面](#扩展管理页面)

# 所有配置（按需配置）

```yaml
concept:
  plugin:
    metadata:
      standard-type: #标准元数据类型，用于自定义标准元数据
    validation:
      max-nested-depth: #最大嵌套深度，默认不限制
      max-read-size: #最大读取大小，默认不限制
    storage:
      type: #存储类型，内存，本地，远程，默认内存
      location: #本地位置或远程bucket
      filter-suffixes: #过滤后缀，用逗号分隔，如.zip,.jar
    autoload:
      enabled: #是否启用自动加载，默认true
      period: #自动加载扫描间隔，默认5000ms
    logger:
      standard:
        enabled: #是否启用标准日志，默认true
    management:
      enabled: #是否启用管理页面，默认true
      authorization:
        password: #管理页面解锁密码，默认空
      github-corner:
        display: #Github角
      header:
        display: #头部
        title:
          display: #标题
          text: #标题内容
      footer:
        display: #底部

# webmvc 可能需要设置文件大小
spring:
  servlet:
    multipart:
      max-file-size: 50MB
      max-request-size: 50MB
```

# 存储

```yaml
concept:
  plugin:
    storage:
      type: #存储类型，内存，本地，远程，默认内存
      location: #本地位置或远程bucket
      filter-suffixes: #过滤后缀，用逗号分隔，如.zip,.jar

```

### type

- 内存 MEMORY
- 本地 LOCAL
- 远程 MINIO，AWS_V1，AWS_V2

### location

- 本地 插件存储目录
- 远程 对象存储bucket

# 过滤器

### 全局过滤（解析时过滤）

- 路径名称过滤（Ant匹配）

```java
@Configuration
public class PluginConfig {

    @Bean
    public PluginFilter pluginFilter() {
        return new EntryFilter("**/**.properties");
    }
}
```

- 类名过滤（Ant匹配）

```java
@Configuration
public class PluginConfig {

    @Bean
    public PluginFilter pluginFilter() {
        return new ClassNameFilter("com.example.**");
    }
}
```

- 类过滤

```java
@Configuration
public class PluginConfig {

    @Bean
    public PluginFilter pluginFilter() {
        return ClassFilter.create(CustomPlugin.class);
    }
}
```

- 类注解过滤

```java
@Configuration
public class PluginConfig {

    @Bean
    public PluginFilter pluginFilter() {
        return ClassFilter.annotation(CustomPluginAnnotation.class);
    }
}
```

- Modifier过滤

```java
@Configuration
public class PluginConfig {

    @Bean
    public PluginFilter pluginFilter() {
        return ClassFilter.modifier(Modifier::isFinal);
    }
}
```

可以通过`PluginFilter#negate()`进行取反

### 注解过滤（提取时过滤）

提取时可添加注解进行过滤

|注解|标记位置|说明|
|-|-|-|
|`@PluginEntry`|方法或参数|路径名称匹配|
|`@PluginText`|参数|String的编码|
|`@PluginClassName`|参数|类名匹配|
|`@PluginClass`|参数|类匹配|
|`@PluginClassAnnotation`|参数|类上注解匹配|

```java
@Component
public class CustomExtractor extends ClassExtractor<Class<?>> {

    @Override
    public void onExtract(@PluginClassName("com.example.**") Class<?> plugin, PluginContext context) {
        
    }
}

@Component
public class SampleDynamicExtractor {

    @OnPluginExtract
    public void sampleExtract(@PluginClassName("com.example.**") Class<?> plugin) {
    }
}
```

`@PluginEntry`标记在方法上将过滤方法中的所有参数

# 插件拦截器

注入`Spring`即可生效

拦截插件元数据和创建

回调顺序：beforeCreatePlugin => beforeCreateMetadata => afterCreateMetadata => afterCreatePlugin

```java
public interface PluginInterceptor {

    /**
     * 插件创建前回调
     *
     * @param definition 插件定义
     * @param context    插件上下文
     */
    default void beforeCreatePlugin(PluginDefinition definition, PluginContext context) {

    }

    /**
     * 插件元数据创建前回调
     *
     * @param definition 插件定义
     * @param context    插件上下文
     */
    default void beforeCreateMetadata(PluginDefinition definition, PluginContext context) {

    }

    /**
     * 插件元数据创建后回调
     *
     * @param metadata   插件元数据
     * @param definition 插件定义
     * @param context    插件上下文
     */
    default void afterCreateMetadata(PluginMetadata metadata, PluginDefinition definition, PluginContext context) {

    }

    /**
     * 插件创建后回调
     *
     * @param plugin     插件
     * @param definition 插件定义
     * @param context    插件上下文
     */
    default void afterCreatePlugin(Plugin plugin, PluginDefinition definition, PluginContext context) {

    }
}

```

# 插件监听器

注入`Spring`即可生效

监听插件加载和卸载（注意此处加载仅为插件树加载，不包括插件提取）

```java
/**
 * 插件监听器
 */
public interface PluginListener {

    /**
     * 插件加载
     *
     * @param plugin  插件
     * @param context 插件上下文
     */
    default void onLoad(Plugin plugin, PluginContext context) {

    }

    /**
     * 插件卸载
     *
     * @param plugin 插件
     */
    default void onUnload(Plugin plugin) {

    }
}

```

# 插件配置

可在插件包的根目录添加`plugin.properties`作为插件的配置文件

`java`项目在`resources`目录下添加即可

`zip`文件在根目录下添加即可

```properties
concept.plugin.name= #名称
concept.plugin.handler.enabled= #是否解析提取插件
concept.plugin.dependency.names= #依赖的插件
#也可添加自定义配置
custom.app=${spring.application.name} #支持Spring占位符
custom.value=custom
```

### 读取配置

可以直接绑定对象或通过名称读取

```java
@Component
public class CustomExtractor extends ClassExtractor<Class<?>> {

    @Override
    public void onExtract(@PluginClassName("com.example.**") Class<?> plugin, PluginContext context) {
        PluginMetadata metadata = context.getPlugin().getMetadata();

        CustomData data = metadata.bind("custom", CustomData.class);
        
        String app = metadata.get("custom.app");
        String value = metadata.get("custom.value");
    }
}

@Component
public class SampleDynamicExtractor {

    @OnPluginExtract
    public void sampleExtract(Class<?> plugin, Plugin plugin) {
        PluginMetadata metadata = plugin.getMetadata();

        CustomData data = metadata.bind("custom", CustomData.class);

        String app = metadata.get("custom.app");
        String value = metadata.get("custom.value");
    }
}

@Data
public static class CustomData {

    private String app;

    private String value;
}
```

# 插件内读取

通过类加载器读取

```java
InputStream is = getClass().getClassLoader().getResourceAsStream("text.txt")
```

# Spring依赖注入

插件包中的类可使用Spring相关依赖，当直接提取实例对象时会自动注入

```java
@Component
public class ExtractBeanApiImpl implements ExtractBeanApi, InitializingBean, DisposableBean, EnvironmentAware {

    @Autowired
    private ApplicationContext applicationContext;

    @Override
    public void exec() {
        
    }

    @PostConstruct
    public void initByPostConstruct() {
        System.out.println("initByPostConstruct");
    }

    @PreDestroy
    public void destroyByPreDestroy() {
        System.out.println("destroyByPreDestroy");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("initByInitializingBean");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("destroyByDisposableBean");
    }

    @Override
    public void setEnvironment(Environment environment) {
        System.out.println("EnvironmentAware: " + environment);
    }

}

```

# 插件依赖

### 内部嵌套依赖

可以把插件依赖的jar打包进插件中

### 依赖其他插件

在被依赖的插件配置文件中定义名称`concept.plugin.name=common`

在被依赖的插件配置文件中设置不解析`concept.plugin.handler.enabled=false`（可选，只做为依赖，不解析插件）

在插件配置文件中设置依赖的插件`concept.plugin.dependency.names=common`，用逗号分隔

需要注意插件的加载顺序，插件加载会校验依赖的插件是否存在

# 插件事件

|事件|说明|
|-|-|
|`PluginCreatedEvent`|插件创建事件|
|`PluginCreateErrorEvent`|插件创建异常事件|
|`PluginLoadedEvent`|插件加载事件|
|`PluginLoadErrorEvent`|插件加载异常事件|
|`PluginUnloadedEvent`|插件卸载事件|
|`PluginUnloadErrorEvent`|插件卸载异常事件|
|`PluginResolvedEvent`|插件解析事件|
|`PluginFilteredEvent`|插件过滤事件|
|`PluginMatchedEvent`|插件匹配事件|
|`PluginConvertedEvent`|插件转换事件|
|`PluginFormattedEvent`|插件格式化事件|
|`PluginExtractedEvent`|插件提取事件|
|`PluginAutoLoadEvent`|插件自动加载事件|
|`PluginAutoLoadErrorEvent`|插件自动加载异常事件|
|`PluginAutoUnloadEvent`|插件自动卸载事件|
|`PluginAutoUnloadErrorEvent`|插件自动卸载异常事件|
|`PluginConceptInitializedEvent`|Concept初始化事件|
|`PluginConceptDestroyedEvent`|Concept销毁事件|

可使用`PluginEventListener`或`@EventListener`监听

# 管理页面解锁

除了可以配置密码`concept.plugin.management.authorization.password`也可以实现自定义`PluginManagementAuthorizer`解锁逻辑

# 扩展管理页面

1. 可在 `resources/concept/plugin` 目录下添加 `concept-plugin.js` 自定义 `init`，`encodePassword(pwd)` 方法

2. `init` 中可以配置 `axios` 设置 `token` 等

```js
function init() {
    axios.interceptors.request.use(config => {
        config.headers['Authorization'] = localStorage.getItem("token");
        return config;
    });
}
```

# 注意事项

1. 管理页面默认密码为空，可直接解锁

2. 暂时不支持 `Spring Boot Jar` 结构的包
