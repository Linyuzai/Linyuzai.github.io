# 概述

一个注解实现全场景下载功能

无差别兼容`webmvc`和`webflux`

# 示例说明

### 静态下载对象

```java
/**
 * classpath下的文件。
 */
@Download(source = "classpath:/download/README.txt")
@GetMapping("/classpath")
public void classpath() {
}

/**
 * 指定路径的文件。可以指定文件夹，会自动压缩整个文件夹。
 */
@Download(source = "file:/Users/Shared/README.txt")
@GetMapping("/file")
public void file() {
}

/**
 * 指定下载地址。可以通过配置缓存下载文件，后面不需要重复下载。
 */
@Download(source = "http://127.0.0.1:8080/concept-download/image.jpg")
@GetMapping("/http")
public void http() {
}
```

### 动态下载对象

```java
/**
 * 返回指定路径的文件。可以指定文件夹，会自动压缩整个文件夹。
 */
@Download
@GetMapping("/file")
public File file() {
    return new File("/Users/Shared/README.txt");
}

/**
 * 支持组合不同类型的文件，会自动压缩所有文件。
 */
@Download(filename = "List.zip")
@GetMapping("/list")
public List<Object> list() {
    List<Object> list = new ArrayList<>();
    list.add(new File("/Users/Shared/README.txt"));
    list.add(new ClassPathResource("/download/README.txt"));
    list.add("http://127.0.0.1:8080/concept-download/image.jpg");
    return list;
}
```

借助`@Download`注解，可以把被下载的资源写在`source`参数中或者作为方法的返回值，两者没有任何区别

# 2.x.x 新版本

注意事项：2.x.x 版本与1.x.x 版本不兼容

1. 【结构优化】移除对`aop`的强依赖，使用`@ControllerAdvice`代替，注解移动到`com.github.linyuzai.download.core.annotation`下

2. 【结构优化】移除对`reactor`的强依赖，在自定义组件的时候没有响应式编程的门槛

3. 【新增功能】支持异步消费，解决长时间占用服务连接的问题

4. 【新增功能】支持`zip`加密，需要依赖`net.lingala.zip4j:zip4j`

5. 【新增功能】支持`tar`，`tar.gz`压缩，需要依赖`org.apache.commons:commons-compress`

6. 【功能优化】支持注解配置`SpEL`

7. 【问题修复】修复下载失败缓存错误的文件的问题

# 最新版本

![Maven Central](https://img.shields.io/maven-central/v/com.github.linyuzai/concept-download-spring-boot-starter)

# 集成

```gradle
implementation 'com.github.linyuzai:concept-download-spring-boot-starter:${version}'
```

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-download-spring-boot-starter</artifactId>
  <version>${version}</version>
</dependency>
```

### Kotlin协程实现网络资源的并发请求

需要手动依赖如下模块

```gradle
implementation 'com.github.linyuzai:concept-download-coroutines:${version}'
```

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-download-load-coroutines</artifactId>
  <version>${version}</version>
</dependency>
```

并且手动注入

```java
@Configuration
public class ConceptDownloadConfig {

    @Bean
    public CoroutinesSourceLoader coroutinesSourceLoader() {
        return new CoroutinesSourceLoader();
    }
}

```

# @Download 注解说明

| 参数 | 说明 |
|-|-|
|`source`|需要下载的内容，但是优先级低于返回值<br>如果方法返回值不为`null`则会使用返回值作为下载的内容|
|`inline`|如果为`true`，可以直接在浏览器预览<br>需要配合`contentType`，如图片或视频，默认`false`<br>视频文件目前存在一些问题，还在测试阶段|
|`filename`|指定下载时浏览器上显示的名称<br>如果不指定则会获取下载内容的名称，如文件则使用文件名|
|`contentType`|如果未指定，会尝试获取<br>如果尝试获取失败，则默认`application/octet-stream`<br>或`application/x-zip-compressed`|
|`compressFormat`|压缩格式，默认`zip`|
|`compressPassword`|压缩密码，默认无密码|
|`forceCompress`|强制压缩<br>如果为`true`，不管下载的文件有几个都会压缩<br>如果为`false`，有多个文件时压缩，只有一个文件时不压缩<br>默认`false`|
|`charset`|如果下载包含中文的文本文件出现乱码，可以尝试指定编码|
|`headers`|统一的响应头，每2个为一组|
|`extra`|额外的数据，当需要自行编写额外流程业务时可能会用到|

# 下载参数

下载时会结合全局配置和注解配置构造一个下载参数`DownloadOptions`

### 自定义下载参数

接口方法返回`DownloadOptions.Configurer`即可重写下载参数

```java
@Download
@GetMapping("/rewrite")
public DownloadOptions.Configurer rewrite() {
    return new DownloadOptions.Configurer() {
        @Override
        public configure(ConfigurableDownloadOptions options) {
            options.setSource(/*被下载的对象*/);
            options.setEventListener(new DownloadEventListener() {
                @Override
                public void onEvent(Object event) {
                    //这个事件监听器并只对该方法生效
                }
            });
        }
    };
}
```

### 异步消费

```java
@Download
@GetMapping("/async")
public DownloadOptions.Configurer async() {
    return new DownloadOptions.Configurer() {
        @Override
        public configure(ConfigurableDownloadOptions options) {
            options.setSource(/*被下载的对象*/);
            options.setAsyncConsumer(new InputStreamConsumer() {
                @Override
                public void consumer(InputStream is) {
                    //获得压缩后的输入流
                }
            });
            options.setAsyncConsumer(new FileConsumer() {
                @Override
                public void consumer(File file) {
                    //获得压缩后的文件，需要启用压缩缓存或源文件缓存才能拿到文件
                }
            });
        }
    };
}
```

# 下载流程

整个下载流程由`DownloadHandler`和`DownloadHandlerChain`实现链式处理

|步骤|处理器|说明|
|-|-|-|
|创建需要下载的`Source`|`CreateSourceHandler`|可实现`Source`和`SourceFactory`支持任意类型数据的扩展|
|加载需要下载的`Source`|`LoadSourceHandler`|网络请求会进行多线程加载，可实现`SourceLoader`自定义加载流程|
|压缩需要下载的`Source`|`CompressSourceHandler`|可实现`SourceCompressor`自定义压缩逻辑|
|数据写入响应|`WriteResponseHandler`|可自定义`DownloadWriter`用于处理输入输出流，异步消费时该处理器不生效|
|异步消费数据|`AsyncConsumeHandler`|异步消费时生效|

### 自定义流程扩展

可以自定义实现`DownloadHandler`并注入到`Spring`的容器中

```java
/**
 * 下载处理器。
 */
public interface DownloadHandler {

    /**
     * 执行处理。
     *
     * @param context {@link DownloadContext}
     * @param chain   {@link DownloadHandlerChain}
     */
    Object handle(DownloadContext context, DownloadHandlerChain chain);
}

```

# 配置文件

```yaml
concept:
  download:
    source:
      cache:
        enabled: true #网络资源缓存是否启用
        path: /source_cache #网络资源缓存路径，默认为 user.home/concept/download
        delete: false #下载结束后网络资源缓存是否删除
    compress:
      format: zip #压缩格式
      password: 123456 #压缩密码
      cache:
        enabled: true #压缩缓存是否启用
        path: /compress_cache #压缩缓存路径，默认为 user.home/concept/download
        delete: false #下载结束后压缩缓存是否删除
    response:
      headers: #额外的响应头
        header1 : 1
        header2 : 2
    logger:
      enabled: true #日志总开关
      standard:
        enabled: true #标准流程日志是否启用
      progress:
        enabled: true #进度计算日志是否启用，包括加载进度，压缩进度，写入响应进度
        duration: 500 #进度计算日志输出间隔，ms
        percentage: true #进度计算日志是否使用百分比输出
      time-spent:
        enabled: true #时间计算日志是否启用
```

# 下载生命周期监听

```java
public class CustomLifecycleListener implements DownloadLifecycleListener {

    /**
     * 下载开始。
     */
    @Override
    public void onStart(DownloadContext context) {
    }

    /**
     * 下载完成。
     */
    @Override
    public void onComplete(DownloadContext context) {
    }
}

```

# 下载上下文

每次下载都会生成一个`DownloadContext`实例

在同一个下载流程中可以通过上下文传递和共享数据

### 自定义上下文

通过实现`DownloadContextFactory`和`DownloadContext`或`AbstractDownloadContext`并注入到`Spring`容器中

```java
/**
 * {@link DownloadContext} 工厂。
 */
public interface DownloadContextFactory {

    /**
     * 创建一个 {@link DownloadContext}。
     *
     * @param options {@link DownloadOptions}
     * @return {@link DownloadContext}
     */
    DownloadContext create(DownloadOptions options);
}
```

# 下载源创建

所有的下载对象最终都会通过`Source`体现，作为原始的下载数据的抽象提供统一的接口

### 支持的下载类型

|类型|说明|
|-|-|
|`FileSource`|支持`File`对象<br>`file:`,`user.home:`,`user-home:`,`user_home:`前缀的字符串|
|`ClassPathSource`|支持`ClassPathResource`对象<br>`classpath:`前缀的字符串|
|`TextSource`|支持任意的`String`对象作为文本文件|
|`HttpSource`|支持`http`地址，基于`HttpURLConnection`|
|`OkHttpSource`|支持http地址，基于`OkHttp`，依赖`OkHttp`时自动适配|
|`PublisherSource`|支持`Publisher`对象|
|`MultipleSource`|支持`array`或`Collection`对象|

**同时支持上述类型任意组合的数组或集合**

```java
@Download(filename = "压缩包.zip")
@GetMapping("/list")
public List<Object> list() {
    List<Object> list = new ArrayList<>();
    list.add(new File("/Users/Shared/README.txt"));
    list.add(new ClassPathResource("/download/image.jpg"));
    list.add("http://127.0.0.1:8080/concept-download/video.mp4");
    return list;
}
```

### 支持反射的方式表示下载源

对于已经存在的数据模型，可以通过注解的方式将一些属性覆盖到对应的`Source`

```java
@Data
@SourceModel
@AllArgsConstructor
public static class BusinessModel {

    @SourceName
    private String name;

    @SourceObject
    private String url;
}

@Download
@GetMapping("/business-model")
public List<BusinessModel> businessModel() {
    List<BusinessModel> businessModels = new ArrayList<>();
    businessModels.add(new BusinessModel("BusinessModel.txt", "http://127.0.0.1:8080/concept-download/text.txt"));
    businessModels.add(new BusinessModel("BusinessModel.jpg", "http://127.0.0.1:8080/concept-download/image.jpg"));
    businessModels.add(new BusinessModel("BusinessModel.mp4", "http://127.0.0.1:8080/concept-download/video.mp4"));
    return businessModels;
}
```

##### 注解说明

|注解|说明|
|-|-|
|`@SourceModel`|标注在类上<br>表明是一个`Source`|
|`@SourceObject`|标注在具体下载对象上|
|`@SourceName`|指定名称|
|`@SourceCharset`|指定编码|
|`@SourceLength`|指定长度，即字节数|
|`@SourceAsyncLoad`|指定是否异步加载|
|`@SourceCacheEnabled`|指定是否启用缓存|
|`@SourceCacheExisted`|缓存是否存在|
|`@SourceCachePath`|缓存目录|

除了`@SourceModel`必须标注在类上

其他注解都可以标注在字段或`get`方法上

所有注解子类优先于父类

如果`Source`本身没有对应属性的`set`方法或者属性字段，则注解无法生效

##### 反射字段的数据类型

`Source`中的编码为`Charset`类型

如果我们的数据模型中对应的类型是`String`

那么在该属性上标注`@SourceCharset`将会导致反射异常

所以引入了`ValueConvertor`来处理类型的转换

当然`String`转`Charset`已经提供实现

```java
/**
 * 将 {@link String} 转为 {@link Charset} 的 {@link ValueConvertor}。
 */
public class StringToCharsetValueConvertor implements ValueConvertor<String, Charset> {

    @Override
    public Charset convert(String value) {
        return Charset.forName(value);
    }
}
```

支持自定义`ValueConvertor`实现

```java
/**
 * 值转换器。
 *
 * @param <Original> 原始类型
 * @param <Target>   目标类型
 */
public interface ValueConvertor<Original, Target> {

    /**
     * 转换。
     *
     * @param value 原始值
     * @return 目标值
     */
    Target convert(Original value);
}
```

并通过`ValueConversion`注册

```java
ValueConversion.getInstance().register(ValueConvertor);
```

### 自定义支持任意类型的下载数据

可以自定义实现`SourceFactory`或`PrefixSourceFactory`和`Source`或`AbstractSource`或`AbstractLoadableSource`并注入到`Spring`容器中

```java
/**
 * {@link Source} 工厂。
 */
public interface SourceFactory {

    /**
     * 是否支持需要下载的原始数据对象。
     *
     * @param source  需要下载的原始数据对象
     * @param context {@link DownloadContext}
     * @return 如果支持则返回 true
     */
    boolean support(Object source, DownloadContext context);

    /**
     * 创建。
     *
     * @param source  需要下载的原始数据对象
     * @param context {@link DownloadContext}
     * @return 创建的 {@link Source}
     */
    Source create(Object source, DownloadContext context);
}


```

# 网络资源加载

针对一些网络资源，如HTTP，支持并发的加载，通过`SourceLoader`来实现

### 内置的资源加载器

|类型|说明|依赖|
|-|-|-|
|`CompletableFutureSourceLoader`|线程池加载||
|`CoroutinesSourceLoader`|协程加载|`download-coroutines`|

每个`Source`都可以单独指定`asyncLoad`属性来控制是否需要异步加载，目前`HttpSource`和`OkHttpSource`默认为`true`，其他默认都为`false`

通过手动注入来切换不同的加载方式

### 自定义加载方式

可以自定义实现`SourceLoader`或`ConcurrentSourceLoader`并注入到`Spring`容器中

```java
/**
 * {@link Source} 加载器。
 */
public interface SourceLoader {

    /**
     * 执行加载。
     *
     * @param source  {@link Source}
     * @param context {@link DownloadContext}
     * @return 加载后的 {@link Source}
     */
    void load(Source source, DownloadContext context);
}
```

# 网络资源缓存

### 配置文件配置

```yaml
concept:
  download:
    source:
      cache:
        enabled: true #是否启用
        path: / #缓存目录
        delete: false #下载结束后是否删除
```

### 注解配置单个方法

```java
@Download(filename = "压缩包.zip")
@SourceCache(group = "source")
@GetMapping("/source-cache")
public String[] sourceCache() {
    return new String[]{
          "http://127.0.0.1:8080/concept-download/text.txt",
          "http://127.0.0.1:8080/concept-download/image.jpg",
          "http://127.0.0.1:8080/concept-download/video.mp4"
    };
}
```

使用`@SourceCache`注解配合`@Download`实现下载资源的缓存处理，优先级高于上面两种方式

##### @SourceCache 注解说明

| 参数 | 说明 |
|-|-|
|`enabled`|是否启用缓存|
|`group`|分组，会在缓存目录下额外创建一个对应的目录作为实际的缓存目录<br>考虑到不同功能出现相同名称的文件等冲突问题<br>默认空，不创建，及直接使用配置的缓存目录|
|`delete`|下载结束后是否删除缓存文件|

# 资源压缩

默认情况下，如果是单个资源则不会压缩，如果是多个资源或者是一整个文件夹则会压缩处理

可以使用`@Download(forceCompress = true)`强制压缩

目前只实现了基于`ZipOutputStream`的`ZipSourceCompressor`

### 自定义压缩

可以自定义实现`SourceCompressor`或`AbstractSourceCompressor`并注入到`Spring`容器中

```java
/**
 * {@link Source} 压缩器。
 *
 * @see ZipSourceCompressor
 */
public interface SourceCompressor extends OrderProvider {

    /**
     * 获得压缩格式。
     *
     * @return 压缩格式
     */
    String getFormat();

    /**
     * 判断是否支持对应的压缩格式。
     *
     * @param format  压缩格式
     * @param context {@link DownloadContext}
     * @return 如果支持则返回 true
     */
    default boolean support(String format, DownloadContext context) {
        return format.equalsIgnoreCase(getFormat());
    }

    /**
     * 如果支持对应的格式就会调用该方法执行压缩。
     *
     * @param source  {@link Source}
     * @param writer  {@link DownloadWriter}
     * @param context {@link DownloadContext}
     * @return {@link Compression}
     */
    Compression compress(Source source, DownloadWriter writer, DownloadContext context);
}

```

# 压缩缓存

### 配置文件配置

```yaml
concept:
  download:
    compress:
      cache:
        enabled: true #是否启用
        path: / #缓存目录
        delete: false #下载结束后是否删除
```

### 注解配置单个方法

```java
@Download(filename = "压缩包.zip")
@CompressCache(group = "compress", delete = true)
@GetMapping("/compress-cache")
public String[] compressCache() {
    return new String[]{
          "http://127.0.0.1:8080/concept-download/text.txt",
          "http://127.0.0.1:8080/concept-download/image.jpg",
          "http://127.0.0.1:8080/concept-download/video.mp4"
    };
}
```

使用`@CompressCache`注解配合`@Download`实现压缩文件的缓存处理，优先级高于上面两种方式

##### @CompressCache 注解说明

| 参数 | 说明 |
|-|-|
|`enabled`|是否启用缓存|
|`group`|分组，会在缓存目录下额外创建一个对应的目录作为实际的缓存目录<br>考虑到不同功能出现相同名称的文件等冲突问题<br>默认空，不创建，及直接使用配置的缓存目录|
|`name`|压缩文件名称<br>单下载源会使用该下载源的名称<br>多下载源会使用第一个有名称的下载源的名称<br>否则使用`CacheNameGenerator`生成，默认使用时间戳|
|`delete`|下载结束后是否删除缓存文件|

# 响应写入

这部分是对输入输出流的具体操作实现

默认实现`BufferedDownloadWriter`来操作字节流或字符流

### 自定义写入器

可以自定义实现`DownloadWriter`并注入到`Spring`容器中

```java
/**
 * 具体操作 {@link InputStream} 和 {@link OutputStream} 的写入器。
 */
public interface DownloadWriter extends OrderProvider {

    /**
     * 该写入器是否支持写入。
     *
     * @param resource {@link Resource}
     * @param range    {@link Range}
     * @param context  {@link DownloadContext}
     * @return 如果支持则返回 true
     */
    boolean support(Resource resource, Range range, DownloadContext context);

    /**
     * 执行写入。
     *
     * @param is      {@link InputStream}
     * @param os      {@link OutputStream}
     * @param range   {@link Range}
     * @param charset {@link Charset}
     * @param length  总大小，可能为 null
     */
    default void write(InputStream is, OutputStream os, Range range, Charset charset, Long length) {
        write(is, os, range, charset, length, null);
    }

    /**
     * 执行写入。
     *
     * @param is       {@link InputStream}
     * @param os       {@link OutputStream}
     * @param range    {@link Range}
     * @param charset  {@link Charset}
     * @param length   总大小，可能为 null
     * @param callback 回调当前进度和增长的大小
     */
    void write(InputStream is, OutputStream os, Range range, Charset charset, Long length, Callback callback);

    /**
     * 进度回调。
     */
    interface Callback {

        /**
         * 回调进度。
         *
         * @param current  当前值
         * @param increase 增长值
         */
        void onWrite(long current, long increase);
    }
}

```
# 事件

在下载的过程中会发布一系列事件

### 事件类型

|事件|说明|
|-|-|
|`DownloadStartedEvent`|下载开始|
|`DownloadCompletedEvent`|下载结束|
|`SourceCreatedEvent`|`Source`创建完成|
|`SourceCacheDeletedEvent`|`Source`缓存删除|
|`SourceReleasedEvent`|`Source`资源释放|
|`SourceAlreadyLoadedEvent`|重复加载|
|`SourceLoadedUsingCacheEvent`|加载使用缓存|
|`SourceLoadingProgressEvent`|加载进度更新|
|`SourceLoadedEvent`|加载完成|
|`SourceCompressedUsingCacheEvent`|压缩使用缓存|
|`SourceCompressionFormatEvent`|压缩格式确定|
|`SourceNoCompressionEvent`|不压缩|
|`SourceMemoryCompressionEvent`|使用内存压缩|
|`SourceFileCompressionEvent`|使用文件压缩|
|`SourceCompressingProgressEvent`|压缩进度更新|
|`SourceCompressedEvent`|压缩完成|
|`CompressionCacheDeletedEvent`|压缩缓存删除|
|`CompressionReleasedEvent`|压缩资源释放|
|`ResponseWritingProgressEvent`|响应写入进度更新|
|`ResponseWrittenEvent`|响应写入|

### 事件监听

可以通过实现`DownloadEventListener`来监听事件

同时支持`Spring`的事件监听

### 自定义事件发布者

可以自定义实现`DownloadEventPublisher`并注入`Spring`容器中，需要自己实现支持`Spring`的事件监听

```java
/**
 * {@link DownloadEvent} 发布器。
 *
 * @see SimpleDownloadEventPublisher
 * @see ApplicationDownloadEventPublisher
 */
public interface DownloadEventPublisher {

    /**
     * 发布事件。
     *
     * @param event 事件
     */
    void publish(Object event);
}
```

# 控制台日志

日志通过特定的`DownloadEventListener`实现

### 日志接口

可以自定义日志打印并注入`Spring`容器中

```java
public interface DownloadLogger {

    void info(String message);

    void error(String message, Throwable e);
}
```

### 日志类型

| 日志 | 说明 |
|-|-|
|`StandardDownloadLogger`|标准流程日志，每个流程相关的事件都会打印|
|`ProgressCalculationLogger`|进度计算日志，包括加载进度，压缩进度，响应写入进度|
|`TimeSpentCalculationLogger`|时间计算日志|

### 配置文件配置

```yaml
concept:
  download:
    logger:
      enabled: true #日志总开关
      standard:
        enabled: true #标准流程日志是否启用
      progress:
        enabled: true #进度计算日志是否启用，包括加载进度，压缩进度，写入响应进度
        duration: 500 #进度计算日志输出间隔，ms
        percentage: true #进度计算日志是否使用百分比输出
      time-spent:
        enabled: true #时间计算日志是否启用
```

# 自定义线程池

可以自定义下载线程池用于异步处理和资源加载

```java
@Bean
public DownloadExecutor downloadExecutor() {
    return new SimpleDownloadExecutor(Executors.newCachedThreadPool());
}
```

# 缓存名称

默认将使用`Source`的名称或当前时间戳作为名称

### 自定义缓存名称

可以自定义实现`CacheNameGenerator`或`AbstractCacheNameGenerator`并注入到`Spring`容器中
