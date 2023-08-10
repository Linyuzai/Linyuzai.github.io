# 概述

目前主要用于动态加载外部`jar`中的`Class`

当我们遇到一些插件化的需求时可能就会想到通过如动态加载类的方式来实现

`java`中自带有`spi`，不过功能有限，是以类加载作为基础概念

而本库是以插件作为基础概念，类加载作为一种插件的具体实现方式

插件可以是一个`jar`文件，一段`java`代码，一个`Excel`文件...

由于`jar`文件相对`java`来说可能更适合作为插件的载体

所以具体实现了`jar`文件作为插件

# 示例说明

假设有一个`jar`包，我们想要提取其中`CustomPlugin.class`这个类或者子类

首先创建一个`JarPluginConcept`并添加一个类提取器`ClassExtractor`，指定提取`CustomPlugin.class`或是其子类

```java
public class ConceptPluginSample {

    /**
     * 插件提取配置
     */
    private final JarPluginConcept concept = new JarPluginConcept.Builder()
            //添加类提取器
            .addExtractor(new ClassExtractor<Class<? extends CustomPlugin>>() {

                @Override
                public void onExtract(Class<? extends CustomPlugin> plugin) {
                    //回调
                }
            })
            .build();
}
```

然后调用`load`方法传入文件地址就会回调`jar`中匹配到的类，如果没有匹配到则不会触发回调

```java
//传入文件路径
concept.load(filePath);
```

当然如果存在多个符合条件的`Class`可以直接指定集合类型

```java
public class ConceptPluginSample {

    /**
     * 插件提取配置
     */
    private final JarPluginConcept concept = new JarPluginConcept.Builder()
            //添加类提取器
            .addExtractor(new ClassExtractor<List<Class<? extends CustomPlugin>>>() {

                @Override
                public void onExtract(List<Class<? extends CustomPlugin>> plugin) {
                    //回调
                }
            })
            .build();
}
```

### [支持的提取器类型在这里](#插件提取器)

# 集成

```gradle
implementation 'com.github.linyuzai:concept-plugin-jar:1.0.1'
```

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-plugin-jar</artifactId>
  <version>1.0.1</version>
</dependency>
```

# 插件动态匹配

动态匹配可以支持任意类型与数量的插件匹配

```java
public class ConceptPluginSample {

    /**
     * 插件提取配置
     */
    private final JarPluginConcept concept = new JarPluginConcept.Builder()
            //将插件提取到...
            .extractTo(this)
            .build();

    @OnPluginExtract
    public void onPluginExtract(Class<? extends CustomPlugin> pluginClass, Properties properties) {
        //任意一个参数匹配上都会触发回调
    }

    /**
     * 加载一个 jar 插件
     *
     * @param filePath jar 文件路径
     */
    public void load(String filePath) {
        concept.load(filePath);
    }
}
```

当我们既要获得某些指定的类又想要同时获得配置文件（假设包里定义了一个`properties`文件）

可以直接定义一个方法，设置参数为我们需要提取的类和配置文件，再在方法上标注`@OnPluginExtract`

然后使用`extractTo`将定义了上述方法的对象传入就行了

### 注解支持

动态匹配还提供了更精准化的注解配置

|注解|说明|
|-|-|
|`@PluginPath`|路径匹配|
|`@PluginName`|名称匹配|
|`@PluginProperties`|`properties`文件匹配|
|`@PluginPackage`|包名匹配|
|`@PluginClassName`|类名匹配|
|`@PluginClass`|类匹配|
|`@PluginAnnotation`|类上注解匹配|

其中`@PluginProperties`可以单独指定`key`

- `@PluginProperties("concept-plugin.a")`可以直接得到对应的`String`值（只能是`String`没有做类型转换）
- `@PluginProperties("concept-plugin.map.**")`可以获得`concept-plugin.map`为前缀的`Map<String, String>`

当存在多个能匹配上的对象（类，配置文件等等）时，可以通过上述注解保证唯一或是使用集合类

由于匹配字符串使用的都是`Spring`中的`AntPathMatcher`，所有注解都支持通配符，如`@PluginPackage("com.github.linyuzai.concept.**.plugin")`

# 插件自动加载

支持通过监听本地文件目录变化来自动加载插件

默认使用`WatchService`来监听文件目录，提供了`JarNotifier`来自动触发加载卸载

```java
@Slf4j
public class ConceptPluginSample {

    //自动加载器
    private final PluginAutoLoader loader = new WatchServicePluginAutoLoader.Builder()
            .locations(new PluginLocation.Builder()
                    //监听目录
                    .path("/Users/concept/plugin")
                    //所有jar
                    .filter(it -> it.endsWith(".jar"))
                    .build())
            //指定线程池
            .executor(Executors.newSingleThreadExecutor())
            //增删改时触发自动加载，自动重新加载，自动卸载
            .onNotify(new JarNotifier(concept))
            //异常回调
            .onError(e -> log.error("Plugin auto load error", e))
            .build();

    /**
     * 开始监听
     */
    @PostConstruct
    private void start() {
        loader.start();
    }

    /**
     * 结束监听
     */
    @PreDestroy
    private void stop() {
        loader.stop();
    }
}
```

# 插件加载流程

- 通过插件工厂`PluginFactory`生成一个插件`Plugin`（`JarPlugin`支持解析文件路径，文件对象和`URL`对象）
- 准备插件`Plugin#prepare()`（通过`JarURLConnection`建立和`jar`的连接）
- 通过插件上下文工厂`PluginContextFactory`生成一个插件上下文`PluginContext`（上下文用于缓存解析过程中的中间数据）
- 初始化上下文`PluginContext#initialize()`
- 调用插件解析链解析插件`PluginResolver#resolve()`（解析`jar`内容）
    - 通过插件匹配器进行匹配`PluginMatcher#match()`（匹配内容，如匹配`Class`或`properties`）
    - 通过插件转换器进行转换`PluginConvertor#convert()`（转换内容，如配置文件转成`json`格式的字符串）
    - 通过插件格式器格式化`PluginFormatter#format()`（格式化，如使用`List`接收时格式化成对应类型）
- 提取插件（回调对应的`PluginExtractor#extract()`或是动态匹配的方法）
- 销毁上下文`PluginContext#destroy`
- 释放插件资源`Plugn#release()`（断开和`jar`的连接）

# 插件

作为插件的统一抽象`Plugin`

针对`jar`实现了`JarPlugin`

### 插件工厂

插件工厂`PluginFactory`用于将各种对象适配成插件对象

可以通过`JarPluginConcept.Builder#addFactory`添加自定义插件工厂

|工厂|说明|
|-|-|
|`JarPathPluginFactory`|支持文件路径|
|`JarFilePluginFactory`|支持`File`对象|
|`JarURLPluginFactory`|支持`jar`协议的`URL`（jar:file:/xxx!/）|

# 插件上下文

插件上下文`PluginContext`用于缓存插件加载期间的中间数据

### 插件上下文工厂

插件上下文工厂`PluginContextFactory`用来创建插件上下文

可以通过`JarPluginConcept.Builder#contextFactory`添加自定义上下文工厂

# 插件提取器

插件提取器`PluginExtractor`用于回调提取到的插件

通过`JarPluginConcept.Builder#addExtractor`添加

|提取器|说明|数据结构|
|-|-|-|
|`ClassExtractor`|支持提取`Class`|`Map<String, Class<CustomPlugin>>`<br>`List<Class<CustomPlugin>>`<br>`Set<Class<CustomPlugin>>`<br>`Collection<Class<CustomPlugin>>`<br>`Class<CustomPlugin>[]`<br>`Class<CustomPlugin>`|
|`InstanceExtractor`|支持提取实例，支持能够使用无参构造器实例化的类|`Map<String, CustomPlugin>`<br>`List<CustomPlugin>`<br>`Set<CustomPlugin>`<br>`Collection<CustomPlugin>`<br>`CustomPlugin`<br>`CustomPlugin[]`|
|`PropertiesExtractor`|支持提取后缀为`.properties`的文件|`Map<String, Properties>`<br>`List<Properties>`<br>`Set<Properties>`<br>`Collection<Properties>`<br>`Properties[]`<br>`Properties`<br>`Map<String, Map<String, String>>`<br>`List<Map<String, String>>`<br>`Set<Map<String, String>>`<br>`Collection<Map<String, String>>`<br>`Map<String, String>[]`<br>`Map<String, String>`|
|`ContentExtractor`|支持提取任意的文件内容（`jar`中会排除`.class`和`.properties`）|`Map<String, byte[]>`<br>`List<byte[]>`<br>`Set<byte[]>`<br>`Collection<byte[]>`<br>`byte[][]`<br>`byte[]`<br>`Map<String, InputStream>`<br>`List<InputStream>`<br>`Set<InputStream>`<br>`Collection<InputStream>`<br>`InputStream[]`<br>`InputStream`<br>`Map<String, String>`<br>`List<String>`<br>`Set<String>`<br>`Collection<String>`<br>`String[]`<br>`String`<br>|
|`PluginObjectExtractor`|可以获得类加载器，`URL`等数据|`Plugin`<br>`JarPlugin`|
|`PluginContextExtractor`|插件加载时的中间数据等|`PluginContext`|
|`DynamicExtractor`<br>`JarDynamicExtractor`|动态插件加载||

当使用`Map`时，对应的`key`将返回文件的路径和名称，如`com/github/linyuzai/concept/sample/plugin/CustomPluginImpl.class`

支持泛型写法

- `List<Class<? extends CustomPlugin>>`
- `Set<? extends CustomPlugin>`
- ...

# 插件过滤器

插件过滤器`PluginFilter`用于过滤插件，减少解析的内容

通过`JarPluginConcept.Builder#addFilter`添加

|过滤器|说明|
|-|-|
|`ClassFilter`|通过类过滤|
|`ClassNameFilter`|通过全限定类名过滤|
|`PackageFilter`|通过包名过滤|
|`AnnotationFilter`|通过类上标注的注解过滤|
|`ModifierFilter`|通过访问修饰符过滤|
|`PathFilter`|通过路径过滤|
|`NameFilter`|通过名称过滤|

其中`ModifierFilter`用法

```java
//是接口或是抽象类
new ModifierFilter(Modifier::isInterface, Modifier::isAbstract);
```

可以通过`PluginFilter#negate()`进行取反

```java
//不是接口并且不是抽象类
new ModifierFilter(Modifier::isInterface, Modifier::isAbstract).negate();
```

# 插件解析器

插件解析器`PluginResolver`用于解析插件内容

通过`JarPluginConcept.Builder#addResolver`添加

|解析器|说明|
|-|-|
|`JarEntryResolver`|用于获得`JarEntry`集合|
|`JarPathNameResolver`|用于获得路径名称集合|
|`JarClassNameResolver`|将路径名称解析为全限定类名|
|`JarClassResolver`|根据全限定类名加载类|
|`JarInstanceResolver`|通过类实例化成对象|
|`PropertiesNameResolver `|获得`.properties`后缀的路径名称|
|`JarPropertiesResolver `|将`.properties`后缀的路径名称加载到`Properties`|
|`JarByteArrayResolver`|读取路径名称对应的内容加载到内存|

### 动态解析

根据添加的插件提取器动态添加插件处理器

比如，当我们只添加了类提取器，那么`properties`相关的解析器就不会被添加，对应的解析逻辑也不会执行

可以近似的理解为`Gradle`或`Maven`的依赖传递

# 插件匹配器

插件匹配器`PluginMatcher`用于在插件解析出来的内容中根据插件提取器中定义的类型来匹配对应的内容

|匹配器|说明|
|-|-|
|`ClassMatcher`|用于匹配类|
|`InstanceMatcher`|用于匹配实例|
|`PropertiesMatcher`|用于匹配`.properties`文件对象，如果`Properties`或`Map<String, String>`|
|`ContentMatcher`|用于匹配文件内容对象，如`byte[]`或`InputStream`或`String`|
|`PluginObjectMatcher`|用于匹配插件对象|
|`PluginContextMatcher `|用于匹配上下文|

# 插件转换器

插件转换器`PluginConvertor`用于将插件匹配器匹配到的内容根据插件提取器中定义的类型来进行转换

如将我们获取的`Properties`对象转换为`Map<String, String>`

|转换器|说明|
|-|-|
|`ByteArrayToInputStreamMapConvertor`|用于将`Map<?, byte[]>`转换为`Map<Object, InputStream>`|
|`ByteArrayToStringMapConvertor`|用于将`Map<?, byte[]>`转换为`Map<Object, String>`|
|`PropertiesToMapMapConvertor`|用于将`Map<?, Properties>`转换为`Map<Object, Map<String, String>>`|

# 插件格式器

插件格式器`PluginFormatter`用于将插件转换器转换后的内容根据插件提取器中定义的类型来格式化

如将我们解析之后的`Map<String, Class<?>>`格式化为`List<Class<?>>`

|格式器|说明|
|-|-|
|`MapToMapFormatter`|用于将`Map<?, ?>`转换为`Map<Object, Object>`|
|`MapToListFormatter`|用于将`Map<?, ?>`转换为`List<Object>`|
|`MapToSetFormatter`|用于将`Map<?, ?>`转换为`Set<Object>`|
|`MapToArrayFormatter`|用于将`Map<?, ?>`转换为`E[]`，对应类型的数组|
|`MapToObjectFormatter`|用于将`Map<?, ?>`转换为`Object`，获取唯一一个元素|

# 插件事件

在插件加载的过程中会发布一系列的事件`PluginEvent`

通过`JarPluginConcept.Builder#addEventListener`添加监听

|事件|说明|
|-|-|
|`PluginCreatedEvent`|插件创建事件|
|`PluginPreparedEvent`|插件准备事件|
|`PluginReleasedEvent`|插件资源释放事件|
|`PluginLoadedEvent`|插件加载事件|
|`PluginUnloadedEvent`|插件卸载事件|
|`PluginResolvedEvent`|插件解析事件|
|`PluginFilteredEvent`|插件过滤事件|
|`PluginMatchedEvent`|插件匹配事件|
|`PluginConvertedEvent`|插件转换事件|
|`PluginFormattedEvent`|插件格式化事件|
|`PluginExtractedEvent`|插件提取事件|
|`PluginAutoLoadEvent`|插件自动加载（监听文件新增）|
|`PluginAutoReloadEvent`|插件自动重新加载事件（监听文件修改）|
|`PluginAutoUnloadEvent`|插件自动卸载事件（监听文件删除）|

### 事件发布者

事件发布者`PluginEventPublisher`用于发布事件

可以通过`JarPluginConcept.Builder#eventPublisher`自定义

# 插件加载日志

基于事件实现的简单的日志输出类`PluginLoadLogger`

```java
@Slf4j
public class ConceptPluginSample {

    private final JarPluginConcept concept = new JarPluginConcept.Builder()
            .extractTo(this)
            //插件加载日志
            .addEventListener(new PluginLoadLogger(log::info))
            .build();

    //省略其他代码。。。
}
```

# 插件类加载器

`jar`中的类通过`JarPluginClassLoader`加载

当`findClass`方法无法加载到对应的类时，会遍历其他的插件类加载器尝试加载

### 插件类加载器工厂

插件类加载器工厂`PluginClassLoaderFactory`用于提供插件类加载器

可以通过`JarPluginConcept.Builder#pluginClassLoaderFactory`自定义

# 插件需要依赖其他`jar`时的注意事项

当我们的`A.jar`需要依赖`B.jar`时，将两个`jar`都进行一次加载即可

但是需要注意，要在`A.jar`中的插件触发`B.jar`中类的类加载之前加载`B.jar`

简单来说就是先加载`B.jar`再加载`A.jar`

或是等所有的`jar`都加载完成后，再调用插件中的方法

如果已经发生无法加载`B.jar`中的类的情况，可以重新加载一遍`A.jar`并替换之前实例化的插件即可重新触发类加载

# 版本

### 列表

##### 1.0.1

- `AbstractPluginConcept`添加`extractTo`方法
- 修复`Windows`环境下`JarPathNameResolver`出现的`PatternSyntaxException`的问题