# 概述

主要用于简化请求和响应的拦截扩展

封装`Spring Boot`提供的切面功能

同时支持`webmvc`和`webflux`

# 集成

```gradle
implementation 'com.github.linyuzai:concept-cloud-web:1.5.4'
```

或者

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-cloud-web</artifactId>
  <version>1.5.4</version>
</dependency>
```

# 拦截器

## 注解方式

```java
@Component
public class Intercepts {

    @OnRequest
    public void request(Request request) {
        //拦截请求
    }

    @OnRequest
    public void requestMethod(HandlerMethod method) {
        //拦截请求
    }

    @OnResponse
    public void response(Throwable e) {
        //拦截响应
    }

    @OnRequest
    @OnResponse
    public void http(WebContext context) {
        //拦截请求和响应
    }
}
```

在方法上添加`@OnRequest`或`@OnResponse`即可进行拦截

参数注入，`WebContext`存储了拦截过程中存储的所有数据，其他参数的注入会通过`WebContext`获取，如果没有则为`null`

### 中断拦截

```java
@Component
public class Intercepts {

    @BreakIntercept
    @OnRequest
    public boolean nonToken(Request request) {
        return request.getPath().equals("/login");
    }
}
```

使用`@BreakIntercept`配合`@OnRequest`或`@OnResponse`可以中断拦截

方法返回值必须为布尔值，返回`true`中断拦截，返回`false`继续拦截

### 拦截顺序

默认顺序为`Ordered.LOWEST_PRECEDENCE`

标记了`@BreakIntercept`的顺序为`100`

可以在方法上标注`@Order`指定顺序

```java
@Component
public class Intercepts {

    @Order(0)
    @OnRequest
    public void order() {
        
    }
}
```

## 配置方式

```yaml
concept:
  cloud:
    web:
      intercept:
        request: #请求拦截配置
          predicate: #断言
            request-path: #请求路径
              - patterns: #匹配路径
                  - /login
                negate: true #取反
                order: 0 #顺序
        response: #响应拦截配置
          predicate: #断言
            request-path: #请求路径
              - patterns: #匹配路径
                  - /xxx
                negate: true #取反
                order: 0 #顺序
        predicate: #断言拦截配置，请求响应都拦截
          request-path: #请求路径
            - patterns: #匹配路径
                - /**/api-docs/**
              negate: true #取反
              order: 0 #顺序
```

断言拦截，`true`为继续拦截，如果想要`@BreakIntercept`的效果，需要设置`negate=true`

## 接口方式

```java
@Order(0)
@OnRequest
@OnResponse
@Component
public class CustomWebInterceptor implements WebInterceptor {

    @Override
    public Object intercept(WebContext context, ValueReturner returner, WebInterceptorChain chain) {
        //拦截业务
        return chain.next(context, returner);
    }
}
```

实现`WebInterceptor`接口即可

可以在类上标注`@OnRequest`和`@OnResponse`来指定拦截请求还是响应，或是都拦截

可以在类上标注`@Order`指定顺序

### 内置拦截器

|拦截器|说明|
|-|-|
|`LoggerErrorResponseInterceptor`|如果发生异常将会打印异常信息|
|`WebResultResponseInterceptor`|包装响应值|
|`StringTypeResponseInterceptor`|返回类型为`String`的响应包装值处理|
|`PredicateWebInterceptor `|断言拦截器，返回`true`接续拦截，返回`false`中断拦截，用于扩展|
|`RequestPathPatternPredicateWebInterceptor `|请求路径断言拦截器|

# 响应包装

默认会将返回值进行包装

```java
/**
 * 用于返回值的包装，提供额外的数据信息
 */
public interface WebResult<R> extends Serializable {

    /**
     * 结果描述
     *
     * @return 对于结果的描述，如是否成功，结果码等
     */
    R getResult();

    /**
     * 返回信息
     *
     * @return 成功或失败的信息描述
     */
    String getMessage();

    /**
     * 返回数据
     *
     * @return 响应体数据
     */
    Object getObject();
}
```

`result`字段不固定类型，方便自定义，默认为布尔值

可以自定义`WebResultFactory`返回其他的`result`类型

```java
/**
 * result 为 {@link Long} 的结果包装类
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class LongWebResult implements WebResult<Long> {

    private static final long serialVersionUID = 1L;

    private Long result;

    private String message;

    private Object object;
}

@Component
public class CustomWebResultFactory extends AnnotationWebResultFactory {

    @Override
    protected WebResult<?> createSuccessWebResult(String message, Object body, WebContext context) {
        return new LongWebResult(0L, message, body);
    }

    @Override
    protected WebResult<?> createFailureWebResult(String message, Throwable e, WebContext context) {
        return new LongWebResult(-1L, message, null);
    }
}
```

# 补充

默认会启用请求拦截，响应拦截，和i18n，可以使用配置开启或关闭

```yaml
concept:
  cloud:
    web:
      intercept:
        enabled: true
        request:
          enabled: true
        response:
          enabled: true
      i18n:
        enabled: true
```

# 版本

### 1.5.4

- 修复WebContext在关闭Response拦截时不会清除的问题

- WebContextManager添加Listener用于监听