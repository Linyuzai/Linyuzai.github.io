# 概述

使指定的类同时继承多个类的字段和方法

![concept-inherit](https://user-images.githubusercontent.com/18523183/187874252-dfb853f5-9b5e-47df-9a40-412ece369675.gif)

# 集成

```gradle
implementation 'com.github.linyuzai:concept-inherit-processor:1.2.0'
annotationProcessor 'com.github.linyuzai:concept-inherit-processor:1.2.0'
```

或者

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-inherit-processor</artifactId>
  <version>1.2.0</version>
</dependency>
```

# 插件

目前只提供了`intellij`插件用于代码提示

搜索插件`Concept Inherit`安装即可

最低支持`2019.3`的版本

# 使用

### @InheritField

用于字段继承

|属性|说明|
|-|-|
|`sources`|指定继承的（多个）类|
|`inheritSuper`|是否继承父类字段|
|`excludeFields`|需要排除的字段名|
|`flags`|[`InheritFlag`说明](#InheritFlag)|

### @InheritMethod

用于方法继承

|属性|说明|
|-|-|
|`sources`|指定继承的（多个）类|
|`inheritSuper`|是否继承父类方法|
|`excludeMethods`|需要排除的方法名|
|`flags`|[`InheritFlag`说明](#InheritFlag)|

### @InheritClass

用于字段和方法继承

|属性|说明|
|-|-|
|`sources`|指定继承的（多个）类|
|`inheritSuper`|是否继承父类字段和方法|
|`excludeFields`|需要排除的字段名|
|`excludeMethods`|需要排除的方法名|
|`flags`|[`InheritFlag`说明](#InheritFlag)|

# InheritFlag

|标识|说明|
|-|-|
|`BUILDER`|生成继承字段对应的`builder`方法|
|`GETTER`|生成继承字段对应的`getter`方法|
|`SETTER`|生成继承字段对应的`setter`方法|
|`OWN`|处理自身的字段和方法，搭配其他标识使用|

# 注意事项

只是将字段和方法复制到指定的类中，并不是真正的继承

请尽量避免相同签名的字段或方法，目前还未完善异常提示

如果遇到编译报错，可以尝试把依赖放到`Lombok`前面

JDK11在`import`了其他类的情况下还存在问题

# 版本

### 列表

##### 1.2.0

- 修复`import`未导入导致编译失败的问题（JDK8修复）
- 修复方法带有`@Override`注解时编译失败的问题