# 概述

提供可插拔的插件能力

通过监听指定目录中文件的变化触发插件文件加载卸载流程

目前已实现基础的内容读取以及加载外部 Class

# 最新版本

![Maven Central](https://img.shields.io/maven-central/v/com.github.linyuzai/concept-plugin-spring-boot-starter)

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

### 2. 配置提取器或使用方法回调（两种方式选择一种即可，示例如下）

#### 提取器（方式一）

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

#### 方法回调（方式二）

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

### 3. 加载插件

加载`jar`或`zip`，支持`jar in jar`或多个`jar`打包成`zip`（视为一个插件）

通过 `/concept-plugin/management.html` 管理页面上传加载插件

![plugin_management](https://github.com/user-attachments/assets/9d5d22b6-a436-4b60-a49f-03e410f7e341)

需要开放`/concept-plugin/**`路径，或者[扩展管理页面](#扩展管理页面)

# 配置属性

```yaml
concept:
  plugin:
    metadata:
      standard-type: #标准配置类型
    autoload:
      enabled: #自动加载
      location:
        base-path: #自动加载目录
    jar:
      mode: #加载模式
    management:
      enabled: #管理页面
      authorization:
        password: #管理页面解锁密码
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

# 插件配置

可在插件包的根目录添加`plugin.properties`作为插件的配置文件

`java`项目在`resources`目录下添加即可

```properties
concept.plugin.name= #名称
concept.plugin.handler.enabled= #是否解析提取插件
concept.plugin.dependency.names= #依赖的插件
concept.plugin.jar.mode= #加载模式
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

# Spring依赖注入

插件包中的类可使用Spring相关依赖，当直接提取实例对象时会自动注入

```java
public class SpringPlugin implements CustomPlugin, ApplicationContextAware {

    @Value("${spring.application.name}")
    private String applicationName;

    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
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

# Jar插件加载模式

可在`application.yml`中配置默认模式

```yaml
concept:
  plugin:
    jar:
      mode: #加载模式
```

也可在插件`plugin.properties`中单独配置

```properties
concept.plugin.jar.mode= #加载模式
```

|模式|兼容性|内存占用|
|-|-|-|
|STREAM|（相对）高|（相对）高|
|FILE|（相对）低|（相对）低|

`STREAM`支持所有`jar`和`zip`，但是会将所有数据读入内存并使用软引用缓存

`FILE`在解析嵌套`jar`和`zip`时只支持`Store`方式的`Entry`，通过随机文件访问读取数据（如 Spring Boot Jar）

压缩时设置`Store`方式

```gradle
jar {
    entryCompression = ZipEntryCompression.STORED
}
```

```java
ZipOutputStream zos = ...;
zos.setMethod(ZipEntry.STORED);
//或
ZipEntry entry = ...;
entry.setMethod(ZipEntry.STORED);
```

# 插件事件

|事件|说明|
|-|-|
|`PluginCreatedEvent`|插件创建事件|
|`PluginPreparedEvent`|插件准备事件|
|`PluginLoadedEvent`|插件加载事件|
|`PluginUnloadedEvent`|插件卸载事件|
|`PluginResolvedEvent`|插件解析事件|
|`PluginFilteredEvent`|插件过滤事件|
|`PluginMatchedEvent`|插件匹配事件|
|`PluginConvertedEvent`|插件转换事件|
|`PluginFormattedEvent`|插件格式化事件|
|`PluginExtractedEvent`|插件提取事件|
|`PluginAutoLoadEvent`|插件自动加载（监听文件新增）|
|`PluginAutoUnloadEvent`|插件自动卸载事件（监听文件删除）|

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