## 一、安装
### 1、下载和解包

cd /usr/local/
wget http://download.redis.io/releases/redis-3.2.1.tar.gz
tar -zxvf /redis-3.2.1.tar.gz

###  2、 编译安装

cd redis-3.2.1
make && make install

## 二、创建redis节点
#### 环境准备

_cd /usr/local/
mkdir redis_cluster
mkdir 7001 8001 9001 7002 8002 9002
cp /usr/local/redis-3.2.1/redis.conf  ./redis_cluster/7001/
cp /usr/local/redis-3.2.1/redis.conf  ./redis_cluster/7002/
cp /usr/local/redis-3.2.1/redis.conf  ./redis_cluster/8001/
cp /usr/local/redis-3.2.1/redis.conf  ./redis_cluster/8002/
cp /usr/local/redis-3.2.1/redis.conf  ./redis_cluster/9001/
cp /usr/local/redis-3.2.1/redis.conf  ./redis_cluster/9002/_

#### 修改每个目录下的配置文件，以 7001 为例

`daemonize yes`                           //redis后台运行
`pidfile /var/run/redis_7000.pid`         //pidfile文件
`port 7001 `                              //端口
`cluster-enabled yes`                     //开启集群  把注释#去掉
`cluster-config-file nodes_7001.conf`     //集群的配置  配置文件首次启动自动生成
`cluster-node-timeout 5000`               //请求超时  设置5秒够了
`appendonly  yes`                         //aof日志开启  有需要就开启，它会每次写操作都记录一条日志

#### 启动节点  以 7001 为例

cd /usr/local
`redis-server redis_cluster/7001/redis.conf`

#### 检查服务
`ps -ef | grep redis`   #查看是否启动成功
`netstat -tnlp | grep redis` #可以看到redis监听端口

## 三、创建集群
前面已经准备好了搭建集群的redis节点，接下来我们要把这些节点都串连起来搭建集群。官方提供了一个工具：redis-trib.rb(/usr/local/redis-3.2.1/src/redis-trib.rb) 看后缀就知道这鸟东西不能直接执行，它是用ruby写的一个程序，所以我们还得安装ruby.
`yum -y install ruby ruby-devel rubygems rpm-build`

再用 gem 这个命令来安装 redis接口    gem是ruby的一个工具包.
`gem install redis`

#### 查看工具包用法

`/usr/local/redis-3.2.1/src/redis-trib.rb`

#### 创建集群

`/usr/local/redis-3.2.1/src/redis-trib.rb create —replicas 1 192.168.1.237:7001 192.168.1.237:7002  192.168.1.237:8001 192.168.1.238:8002  192.168.1.238:9001  192.168.1.238:9002`

    --replicas  1  表示 自动为每一个master节点分配一个slave节点  上面有6个节点，程序会按照一定规则生成 3个master（主）3个slave(从)
    运行中，提示Can I set the above configuration? (type 'yes' to accept): yes    //输入yes

#### 查看集群状态

`/usr/local/redis/src/redis-trib.rb check 192.168.1.237:7000`

## 四、测试
get 和 set数据
`redis-cli -c -p 7000`