

# 如何处理CONNECTION_LOSS异常

CONNECTION_LOSS means the link between the client and server was broken. **It doesn't necessarily mean that the request failed**. If you are doing a create request and the link was broken after the request reached the server and before the response was returned, the create request will succeed. If the link was broken before the packet went onto the wire, the create request failed. Unfortunately, there is no way for the client library to know, so it returns CONNECTION_LOSS. **The programmer must figure out if the request succeeded or needs to be <u>retried</u>.** Usually this is done in an application specific way. Examples of success detection include checking for the presence of a file to be created or checking the value of a znode to be modified.

When a client (session) becomes partitioned from *(partitioned:网络分片,即断连接了)* the ZK serving cluster it will begin searching the list of servers that were specified during session creation. Eventually, **when connectivity between the client and at least one of the servers is re-established, the session will either again transition to the "connected" state (if reconnected within the session timeout value) or it will transition to the "expired" state (if reconnected after the session timeout)**. The ZK client library will handle reconnect for you automatically. In particular we have heuristics built into the client library to handle things like "herd effect", etc... Only create a new session when you are notified of session expiration (mandatory).



# 如何处理SESSION_EXIPRE异常

SESSION_EXPIRED automatically **closes the ZooKeeper handle**. In a correctly operating cluster, you should never see SESSION_EXPIRED. It means that the client was partitioned off from the ZooKeeper service for more than the session timeout and ZooKeeper decided that the client died. Because the ZooKeeper service is ground truth, the client should consider itself dead and go into recovery. If the client is only reading state from ZooKeeper, recovery means just reconnecting. **In more complex applications, recovery means recreating ephemeral nodes, vying for leadership roles, and reconstructing published state.(恢复意味着要重建临时节点，重新获取Leader权限，重建数据)**

**Library writers should be conscious of the severity of the expired state and not try to recover from it.** Instead libraries should return a fatal error. Even if the library is simply reading from ZooKeeper, the user of the library may also be doing other things with ZooKeeper that requires more complex recovery.

Session expiration is managed by the ZooKeeper cluster itself, not by the client. When the ZK client establishes a session with the cluster it provides a "timeout" value. This value is used by the cluster to determine when the client's session expires. Expirations happens when the cluster does not hear from the client within the specified session timeout period (i.e. no heartbeat). **At session expiration the cluster will delete any/all ephemeral nodes owned by that session and immediately notify any/all connected clients of the change (anyone watching those znodes)**. At this point the client of the expired session is still disconnected from the cluster, it will not be notified of the session expiration until/unless it is able to re-establish a connection to the cluster. The client will stay in disconnected state until the TCP connection is re-established with the cluster, at which point the watcher of the expired session will receive the "session expired" notification.

1. 'connected': session is established and client is communicating with cluster (client/server communication is operating properly)(客户端连接成功)
2. client is partitioned from the cluster (客户端发生网络分片)
3. 'disconnected': client has lost connectivity with the cluster(客户端丢失连接)
4. time elapses, after 'timeout' period the cluster expires the session, nothing is seen by client as it is disconnected from cluster(服务端发现客户端超时，直接移除，客户端自己还不知道)
5. time elapses, the client regains network level connectivity with the cluster(客户端一段时间之后又有网络交互)
6. 'expired': eventually the client reconnects to the cluster, it is then notified of the expiration(这次交互的时候，集群告诉客户端:“你超时了!”)

# Watcher的处理

在Zookeeper客户端建立连接的时候,会传入一个Watcher对象,该watcher主要是响应"与链接状态转换"有关的事件(比如,"建立链接","链接关闭"等,参见KeeperState).此默认watcher由zkClient本地持有，且生命周期伴随整个zookeeper实例,而不是"一次触发即消亡",当Client收到EventType.NONE类型的消息时,则会触发这个"默认wather"被执行。

在Zookeeper的客户端包WatcherManager中，存在如下2个map

```java
private final HashMap<String, HashSet<Watcher>> watchTable =
    new HashMap<String, HashSet<Watcher>>();

private final HashMap<Watcher, HashSet<String>> watch2Paths =
    new HashMap<Watcher, HashSet<String>>();
```

可以看出，同一个Watcher在一个Patch上只可以注册一次。我们一般在Watcher子类的Process方法内部，可以尝试重新设置Watcher，来达到永久监听的效果。另外在与Zookeeper Cluster连接超时重建的时候，也会重新创建整个系统的所有Watcher，尝试恢复系统状态，此时建立的Watcher 与 Watcher子类内部收到Disconnected 与 SessionExpired事件时重建的Watcher不会同时存在（HashSet 与 HashMap的去重机制）。实现这一个效果的前提是，**所有的 Watcher 必须重写 HashCode 与 Equals 方法**。

 