# 概述

主要将异步回调的功能场景封装成同步返回

与`Future`不同，该工具主要针对从服务外部进行回调的业务场景

# 示例说明

以我们现在物联网开发为例，或多或少都会接触到物理设备的对接，甚至需要对设备进行控制等操作

但是有很大一部分设备不会同步返回结果，而是通过额外上报一条数据来响应命令

所以在下发了一条命令之后无法直接获得对应的结果

而该库能十分方便的将异步回调转换成同步返回

```java
@RestController
@RequestMapping("/concept-sync-waiting")
public class SyncWaitingController {

    /**
     * 新建一个 {@link SyncWaitingConcept} 对象
     * 也可以直接在 Spring 容器中注入一个全局使用
     */
    private final SyncWaitingConcept concept = new ConditionSyncWaitingConcept();

    /**
     * 下发命令，阻塞线程直到数据上报或超时
     *
     * @param key 每条命令唯一的id
     * @return 设备上报的数据
     */
    @RequestMapping("/send")
    public String send(@RequestParam String key) {
        try {
            return concept.waitSync(key, new SyncCaller() {
                @Override
                public void call(Object k) {
                    //在这里下发命令
                }
            }, 5000);
        } catch (SyncWaitingTimeoutException e) {
            return "下发命令超时";
        }
    }

    /**
     * 接收设备上报的数据，唤醒下发命令的线程
     *
     * @param key   一般需要从上报数据中附带命令id
     * @param value 上报数据
     */
    @RequestMapping("/receive")
    public void receive(@RequestParam String key, @RequestParam String value) {
        concept.notifyAsync(key, value);
    }
}
```

# 集成

```gradle
implementation 'com.github.linyuzai:concept-sync-waiting:1.0.0'
```

```xml
<dependency>
  <groupId>com.github.linyuzai</groupId>
  <artifactId>concept-sync-waiting</artifactId>
  <version>1.0.0</version>
</dependency>
```

# 使用

首先创建一个`SyncWaitingConcept`对象，默认实现了`ConditionSyncWaitingConcept`

```java
SyncWaitingConcept concept = new ConditionSyncWaitingConcept();
```

然后调用`waitSync`方法，并阻塞当前线程

需要传入`key`作为该次调用的标识（唯一id），`SyncCaller`作为触发业务逻辑调用的接口，`waitingTime`作为等待时间限制（小于等于0时则无限等待）

```java
Object value = concept.waitSync(key, new SyncCaller() {
            @Override
            public void call(Object k) {
                //自己的业务逻辑，并附带上key
            }
        }, 5000);
```

最后当接收到异步返回的数据时，调用`notifyAsync`方法唤醒之前阻塞的线程即可得到接收到的数据

需要传入`key`一般在返回数据中附带回来，`value`作为接收到的数据

```java
concept.notifyAsync(key, value);
```

# 高级

### 创建

`ConditionSyncWaitingConcept`可以通过`Builder`创建

```java
SyncWaitingConcept concept = new ConditionSyncWaitingConcept.Builder()
            .lock(Lock lock)
            .container(SyncWaiterContainer container)
            .recycler(SyncWaiterRecycler recycler)
            .build();
```

`Lock`用于加锁，解锁，以及新建`Condition`

`SyncWaiterContainer`用于缓存当前等待中的`SyncWaiter`，默认为`MapSyncWaiterContainer`使用`HashMap`缓存

每个`waitSync`请求都会对应一个`SyncWaiter`实例

`SyncWaiter`持有`key`和`value`以及其实现类`ConditionSyncWaiter`额外持有`Condition`对象

`SyncWaiterRecycler`用于回收复用`SyncWaiter`，默认为`DisposableSyncWaiterRecycler`不进行回收复用

同时提供了`QueueSyncWaiterRecycler`可以缓存固定数量的`SyncWaiter`

因为`ConditionSyncWaitingConcept`的操作是加锁的，所以`SyncWaiterContainer`和`SyncWaiterRecycler`的默认实现是线程不安全的

但是如果你需要将`SyncWaiterContainer`和`SyncWaiterRecycler`公用给多个`SyncWaitingConcept`

请自定义实现线程安全的`SyncWaiterContainer`和`SyncWaiterRecycler`

### 队列等待时间

当我们的`key`重复时，并在当前存在该`key`正在等待中，则会进行排队阻塞

简单来说，就是我们用一个`key`调用了`waitSync`的方法后

在数据还未返回，即还未调用`notifyAsync`时

我们再使用相同的`key`调用一次`waitSync`将不会触发业务逻辑的调用

而是会排队等待上一个等待结束后（返回或超时）继续后面的逻辑

防止在数据回调时无法区分是哪次调用

所以在调用`waitSync`时可以同时指定`queuingTime`作为排队时间限制（小于等于0时则无限排队等待）

因此并不建议在业务中允许相同的`key`出现，此实现仅仅是为了容错考虑

# 原理

主要通过`Condition.await`阻塞线程，`Condition.signalAll`唤醒线程

![sync_waiting_process](https://user-images.githubusercontent.com/18523183/154452733-b28c8615-5952-498f-b9b5-f62b2947b605.jpg)
