### Github 下载 etcd 代码
<https://github.com/etcd-io/etcd/releases>  
该页面附有安装命令这里不再解释了

`cd /usr/local/bin`  
`ln -s etcdctl /Users/zhangwei/Software/etcd-v3.3.11-darwin-amd64/etcdctl`

### 启动etcd
进入解压目录，执行 ./etcd 启动 (需要sudo -s)

### 简单操作etcd
操作时，需要添加 ETCDCTL_API=3 如

`ETCDCTL_API=3 etcdctl put mykey "this is awesome"`  
`ETCDCTL_API=3 etcdctl get mykey`
