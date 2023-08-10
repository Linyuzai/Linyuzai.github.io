# 概述

在`Queue`的基础上结合了`Map`的特性

在入队的时候，如果已经存在对应的`key`则会更新`value`，否则添加的队列最后

支持阻塞队列，同时对`Map`的操作也会阻塞

# 集成

```gradle
implementation 'com.github.linyuzai:concept-mapqueue-core:1.1.0'
```

或者

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-mapqueue-core</artifactId>
  <version>1.1.0</version>
</dependency>
```

# 使用

```java
BlockingMapQueue<String, String> bmq = new LinkedBlockingMapQueue<>();

//通过 Map 操作
Map<String, String> map = bmq.map();

//通过 Queue 操作
BlockingQueue<String> queue = bmq.queue();
```

通过`map()`和`queue()`可以获得对应的实例调用对应的方法

