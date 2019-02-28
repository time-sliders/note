# 官网与下载
<http://redis.io/>  
<http://doc.redisfans.com/>

# 安装以及使用
##### 获取源码、解压、进入源码目录

    tar xzf redis-1.2.6.tar.gz
    cd redis-1.2.6
    
##### 编译生成可执行文件
由于makefile文件已经写好，我们只需要直接在源码目录执行make命令进行编译即可：

    make
    make-test
    sudo make install
    
make命令执行完成后，会在src目录下生成可执行文件，分别是redis-server、redis-cli、redis-benchmark，它们的作用如下：

**redis-server**：Redis服务器的daemon启动程序
**redis-cli**：Redis命令行操作工具。当然，你也可以用telnet 127.0.0.1 6379 根据其纯文本协议来操作
**redis-benchmark**：Redis性能测试工具，测试Redis在你的系统及你的配置下的读写性能

##### 建立Redis目录（非必须）
这个过程不是必须的，只是为了将Redis相关的资源统一管理而进行的操作。
执行以下命令建立相关目录并拷贝相关文件至目录中：
    
    sudo -s
    mkdir -p /usr/local/redis/bin
    mkdir -p /usr/local/redis/etc
    mkdir -p /usr/local/redis/var
    cp redis-server redis-cli redis-benchmark redis-stat /usr/local/redis/bin/
    cp redis.conf /usr/local/redis/etc/  (安装目录下)
    
# 配置参数
在我们成功安装Redis后，我们直接执行redis-server即可运行Redis，此时它是按照默认配置来运行的（默认配置甚至不是后台运 行）。我们希望Redis按我们的要求运行，则我们需要修改配置文件，Redis的配置文件就是我们上面第二个cp操作的redis.conf文件，它被 我们拷贝到了/usr/local/redis/etc/目录下。修改它就可以配置我们的server了。如何修改？下面是redis.conf的主要配 置参数的意义：

    daemonize：是否以后台daemon方式运行
    pidfile：pid文件位置
    port：监听的端口号
    timeout：请求超时时间
    loglevel：log信息级别
    logfile：log文件位置
    databases：开启数据库的数量
    save * *：保存快照的频率，第一个*表示多长时间，第二个*表示执行多少次写操作。在一定时间内执行一定数量的写操作时，自动保存快照。可设置多个条件。
    rdbcompression：是否使用压缩
    dbfilename：数据快照文件名（只是文件名，不包括目录）
    dir：数据快照的保存目录（这个是目录）
    appendonly：是否开启appendonlylog，开启的话每次写操作会记一条log，这会提高数据抗风险能力，但影响效率。
    appendfsync：appendonlylog如何同步到磁盘（三个选项，分别是每次写都强制调用fsync、每秒启用一次fsync、不调用fsync等待系统自己同步）
下面是一个略做修改后的配置文件内容：

    daemonize yes
    pidfile /usr/local/redis/var/redis.pid
    port 6379
    timeout 300
    loglevel debug
    logfile /usr/local/redis/var/redis.log
    databases 16   # 一般情况只会用一个库，这个值可以改成1
    save 900 1
    save 300 10
    save 60 10000
    rdbcompression yes
    dbfilename dump.rdb
    dir /usr/local/redis/var/
    appendonly no
    appendfsync always
    glueoutputbuf yes
    shareobjects no
    shareobjectspoolsize 1024

将上面内容写为redis.conf并保存到/usr/local/redis/etc/目录下
然后在命令行执行：

`/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf`

即可在后台启动 redis 服务，这时你通过 `telnet 127.0.0.1 6379` 即可连接到你的 redis 服务 
（如果没安装telnet：`brew install telnet` 这个命令安装）


