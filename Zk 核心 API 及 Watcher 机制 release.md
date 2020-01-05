## Zk 核心API 及 Watcher 机制

### 1、Zookeeper 类

通过其构造函数完成与 zk 服务器的链接：

```Java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)
```

connectString：逗号分隔的列表，即 host:port 对，host 可为ip和机器名，port 是 zk 节点对客户端提供服务的端口。客户端会任意选取其中一个 host:port 对来建立链接。

sessionTimeout：session 超时时间。

watcher：用于接收来自 zk 集群的事件。

### 2、Zookeeper 类的主要函数

以下这些函数都有异步和同步版本。

#### 1).create

```java
public String create(String path, byte[] data, List<ACL> acl, CreateMode createMode) throws KeeperException, InterruptedException {...}
```

```java
public void create(String path, byte[] data, List<ACL> acl, CreateMode createMode, StringCallback cb, Object ctx) {...}
```

#### 2).delete

```java
public void delete(String path, int version) throws InterruptedException, KeeperException {...}
```

```java
public void delete(String path, int version, VoidCallback cb, Object ctx) {...}
```

#### 3).exists

```java
public Stat exists(String path, Watcher watcher) throws KeeperException, InterruptedException {...}
```

```java
public Stat exists(String path, boolean watch) throws KeeperException, InterruptedException {...}
```

```java
public void exists(String path, Watcher watcher, StatCallback cb, Object ctx) {...}
```

#### 4).getData

boolean watch 表示是否使用上下文中默认的 watcher（下文有解释），即创建zk实例时设置的 watcher。

```java
public byte[] getData(String path, Watcher watcher, Stat stat) throws KeeperException, InterruptedException {...}
```

```java
public byte[] getData(String path, boolean watch, Stat stat) throws KeeperException, InterruptedException {...}
```

```java
public void getData(String path, Watcher watcher, DataCallback cb, Object ctx) {...}
```

```java
public void getData(String path, boolean watch, DataCallback cb, Object ctx) {...}
```

#### 5).setData

version 字段值如果是-1，则表示无条件更新，其他为有条件更新。

```java
public Stat setData(String path, byte[] data, int version) throws KeeperException, InterruptedException {...}
```

```java
public void setData(String path, byte[] data, int version, StatCallback cb, Object ctx) {...}
```

#### 6).getChildren

```java
public List<String> getChildren(String path, Watcher watcher) throws KeeperException, InterruptedException {...}
```

 ......

#### 7).sync

​	是使得 client 当前连接着的 ZooKeeper 服务器，和 ZooKeeper 的 Leader 节点同步（sync）一下数据。

​	当 follower 收到到 sync 请求时，会将这个请求添加到一个 pendingSyncs 队列里，然后将这个请求发送给leader，直到收到 leader 的 Leader.SYNC 消息时，才将这个请求从 pendingSyncs 队列里移除，并 commit 这个请求。

```java
public void sync(String path, VoidCallback cb, Object ctx) {...}
```

### 3、watcher 机制

#### 机制浅析

​	多个分布式进程通过 ZooKeeper 提供的 API 来操作共享的 ZooKeeper 内存数据对象 ZNode 来达成某种一致的行为或结果，这种模式本质上是基于状态共享的并发模型，与 Java 的多线程并发模型一致，他们的线程或进程都是”共享式内存通信“。Java 没有直接提供某种响应式通知接口来监控某个对象状态的变化，只能要么浪费 CPU 时间毫无响应式的轮询重试，或基于 Java 提供的某种主动通知（Notify）机制（内置队列）来响应状态变化，但这种机制是需要循环阻塞调用。而 ZooKeeper 实现这些分布式进程的状态（ZNode 的 Data、Children）共享时，基于性能的考虑采用了类似的异步非阻塞的主动通知模式即Watch机制，使得分布式进程之间的“共享状态通信”更加实时高效，其实这也是 ZooKeeper 的主要任务决定的—协调。Consul 虽然也实现了 Watch 机制，但它是阻塞的长轮询。

​	Watch 的整体流程：客户端先向 ZooKeeper 服务端成功注册想要监听的节点状态，同时客户端本地会存储该监听器相关的信息在 WatchManager 中，当 ZooKeeper 服务端监听的数据状态发生变化时，ZooKeeper 就会主动通知发送相应事件信息给相关会话客户端，客户端就会在本地响应式的回调相关 Watcher 的 process 方法。这里需要注意，Watcher 不能监控孙子节点。watcher 设置后，一旦触发一次后就会失效，如果要想一直监听，需要在 process 回调函数里重新注册相同的 **watcher**。核心流程如下图：

![img](https://upload-images.jianshu.io/upload_images/7382796-ae10260074027ac3..png?imageMogr2/auto-orient/strip|imageView2/2/w/538/format/webp)

```java
private final ZooKeeper.ZKWatchManager watchManager; // 类 Zookeeper 中的常量
```

```
private static class ZKWatchManager implements ClientWatchManager {
    // 分别存放相应的 Watcher 到本地内存
    private final Map<String, Set<Watcher>> dataWatches; 
    private final Map<String, Set<Watcher>> existWatches;
    private final Map<String, Set<Watcher>> childWatches;
    // 通过Zookeeper类的构造函数这种方式注册的 watcher 将会作为整个 zk 会话期间的默认 watcher，会一直被保存在客户端 ZK WatchManager 的 defaultWatcher 中
    private Watcher defaultWatcher；
    ...
}
```

```
/**
 * 事件注册类，它有三个子类：ChildWatchRegistration、DataRegistration、ExistsRegistration
 */
abstract class WatchRegistration {
    private Watcher watcher;
    private String clientPath;

    public WatchRegistration(Watcher watcher, String clientPath) {
    this.watcher = watcher;
    this.clientPath = clientPath;
    }

    // 供子类来实现
    protected abstract Map<String, Set<Watcher>> getWatches(int var1);
    ...
}
```

#### 通知事件与状态

其中 Watcher 是一个接口，需要用户实现这个接口并实现 `void process(WatchedEvent var1)` 方法。WatchedEvent 是对 KeeperState、EventType、path 的封装。从中可以获取到通知状态和事件类型。

```java
public class WatchedEvent {
    private final KeeperState keeperState;
    private final EventType eventType;
    private String path;
    ...
}
```

```java
public interface Watcher {
    void process(WatchedEvent var1);
    public interface Event {
        // 事件类型
        public static enum EventType {
            None(-1),
            NodeCreated(1),
            NodeDeleted(2),
            // 无论节点数据发生变化还是数据版本发生变化都会触发，即使被更新数据与新数据一样，数据版本dataVersion都会发生变化
            NodeDataChanged(3), 
            // 新增节点或者删除节点
            NodeChildrenChanged(4);
			...
        }
        // 通知状态
        public static enum KeeperState {
            /** @deprecated */
            @Deprecated
            Unknown(-1),
            Disconnected(0),
            /** @deprecated */
            @Deprecated
            NoSyncConnected(1),
            SyncConnected(3),
            AuthFailed(4),
            ConnectedReadOnly(5),
            SaslAuthenticated(6),
            Expired(-112);
            ...
        }
    }
}
```

KeeperState 和 EventType 关联如下：

![img](https://upload-images.jianshu.io/upload_images/7382796-b14f9d7463268462..png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

​	客户端只能收到服务器发过来的相关事件通知，并不能获取到对应数据节点的原始数据及变更后的新数据。因此，如果业务需要知道变更前的数据或者变更后的新数据，需要业务保存变更前的数据(本机数据结构、文件等)和调用接口获取新的数据。

#### 客户端与服务端的注册流程

客户端：

![img](https://upload-images.jianshu.io/upload_images/7382796-5a14cf299d9783c8..png?imageMogr2/auto-orient/strip|imageView2/2/w/1065/format/webp)

服务端：

![img](https://upload-images.jianshu.io/upload_images/7382796-5a14cf299d9783c8..png?imageMogr2/auto-orient/strip|imageView2/2/w/1065/format/webp)

​	FinalRequestProcessor 类实际是任何事务请求和任何查询的的最终处理类。也就是我们客户端对节点的set/get/delete/create/exists 等操作最终都会运行到这里。