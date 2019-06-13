# 下载 zookeeper 安装包

[zookeeper](http://www.apache.org/dist/zookeeper/)

# Zooker配置

Zookeeper集群模式至少需要3个进程，这里单机版我们采用不同端口来进行模拟

```sh
# 我们创建如下目录作为我们的 zookeeper 配置跟目录
cd /data/www/zk_cluster

# 在该目录下创建三个文件夹
mkdir server1
mkdir server2
mkdir server3

cd server1

# 将 zookeeper 的 zoo.cfg 配置文件拷贝到该目录
cp ~/Software/zookeeper-3.5.1-alpha/conf/zoo.cfg ./
```

**创建配置文件**

```properties
# server1
dataDir=/data/www/zk_cluster/server1
dataLogDir=/data/www/zk_cluster/server1/log
clientPort=2181
server.1=127.0.0.1:2001:3001
server.2=127.0.0.1:2002:3002
server.3=127.0.0.1:2003:3003
```

```properties
# server2
dataDir=/data/www/zk_cluster/server2
dataLogDir=/data/www/zk_cluster/server2/log
clientPort=2182
server.1=127.0.0.1:2001:3001
server.2=127.0.0.1:2002:3002
server.3=127.0.0.1:2003:3003
```

```properties
# server3
dataDir=/data/www/zk_cluster/server3
dataLogDir=/data/www/zk_cluster/server3/log
clientPort=2183
server.1=127.0.0.1:2001:3001
server.2=127.0.0.1:2002:3002
server.3=127.0.0.1:2003:3003
```

> **dataDir**：zookeeper保存数据的目录。
> **clientPort**：客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。
> **server.A=B：C：D**：其 中
> 　　　　　A 是一个数字，表示这个是第几号服务器；
> 　　　　　B 是这个服务器的 ip地址；
> 　　　　　C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；
> 　　　　　D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader

**创建 myid 文件**

在各个目录下创建 **myid** 文件
server1机器的内容为：**1**
server2机器的内容为：**2**
server3机器的内容为：**3**

```sh
#启动 zk 集群服务
/Users/zhangwei/Software/zookeeper-3.5.1-alpha/bin/zkServer.sh --config /data/www/zk_cluster/server1 start
/Users/zhangwei/Software/zookeeper-3.5.1-alpha/bin/zkServer.sh --config /data/www/zk_cluster/server2 start
/Users/zhangwei/Software/zookeeper-3.5.1-alpha/bin/zkServer.sh --config /data/www/zk_cluster/server3 start

#查看 zk 各节点状态
/Users/zhangwei/Software/zookeeper-3.5.1-alpha/bin/zkServer.sh --config /data/www/zk_cluster/server1 status
/Users/zhangwei/Software/zookeeper-3.5.1-alpha/bin/zkServer.sh --config /data/www/zk_cluster/server2 status
/Users/zhangwei/Software/zookeeper-3.5.1-alpha/bin/zkServer.sh --config /data/www/zk_cluster/server3 status
```



# 直接通过 Idea 启动 zkServer

```properties
log4j.rootLogger=INFO,C

# Console appender
log4j.appender.C=org.apache.log4j.ConsoleAppender
log4j.appender.C.layout=org.apache.log4j.PatternLayout
log4j.appender.C.layout.ConversionPattern=[%d{MMdd HH:mm:ss SSS\} %-5p] [%t] %c{3\} - %m%n
log4j.appender.C.filter.F=org.apache.log4j.varia.LevelRangeFilter
log4j.appender.C.filter.F.LevelMin=INFO
log4j.appender.C.filter.F.LevelMax=ERROR
```

`QuorumPeerMain#main` 方法中添加如下2行即可

```java
PropertyConfigurator.configure("/data/www/zk_cluster/server1/log4j.properties");
args = new String[]{"/data/www/zk_cluster/server1/zoo.cfg"};
```

