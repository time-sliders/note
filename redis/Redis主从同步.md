# Replication的工作原理
在Slave启动并连接到Master之后，它将主动发送一条 SYNC命令。此后Master将启动后台存盘进程，同时收集所有接收到的用于修改数据集的命令，在后台进程执行完毕后，Master将传送整个数据库文件到Slave，以完成一次完全同步。而Slave服务器在接收到数据库文件数据之后将其存盘并加载到内存中。此后，Master继续将所有已经收集到的修改命令，和新的修改命令依次传送给Slaves，Slave将在本次执行这些数据修改命令，从而达到最终的数据同步。

如果Master和Slave之间的链接出现断连现象，Slave可以自动重连Master，但是在连接成功之后，一次完全同步将被自动执行。

# 如何配置Redis主从复制

同时启动两个Redis服务器，可以考虑在同一台机器上启动两个Redis服务器，分别监听不同的端口，如6379(master)和6380(slave)。

在Slave服务器上执行一下命令：

    d:\dev\redis-2.4.5-win64>`redis-cli.exe -h 127.0.0.1 -p 6380` #这里我们假设Slave的端口号是6380  
    redis 127.0.0.1:6380> slaveof 127.0.0.1 6379 #假设Master和Slave在同一台主机，Master的端口为6379
    
上面的方式只是保证了在执行slaveof命令之后，redis-6380成为了redis-6379的slave，一旦服务(redis-6380)重新启动之后，他们之间的复制关系将终止。

如果希望长期保证这两个服务器之间的Replication(主从复制)关系，可以在redis-6380的配置文件中做如下修改：

将 `# slaveof <masterip> <masterport>` 改为 `slaveof 127.0.0.1 6379` 保存退出。

这样就可以保证 Redis-6380 服务程序在每次启动后都会主动建立与 Redis-6379 的 Replication 连接了。

redis配置文件：`/etc/redis/redis.conf`  
redis服务路径：`/etc/init.d/redis-server`

    注意：
    a、如果在Slave中删除mykey，不能同时删除Master中的mykey。
    b、Slave启动顺利跟Master启动无关联。
