# etcd 环境准备
 修改各节点 HOSTNAME，保证 HOSTNAME 不同(测试 IP，请根据实际使用更改)  
k8s-master01  192.168.7.101  
k8s-master02  192.168.7.102  
k8s-master03  192.168.7.103  
# 下载 etcd 二进制文件:
    Note：https://github.com/coreos/etcd/releases页面下载最新版本的二进制文件，分发到其他etcd节点
$ wget https://github.com/etcd-io/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz  
$ tar xf etcd-v3.3.9-linux-amd64.tar.gz  
$ mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin
 
    分发ETCD二进制文件到其余节点（测试IP，请根据实际使用更改）
$ for i in {2..3}; do scp /usr/local/bin/etcd* root@192.168.7.10$i:/usr/local/bin/;done
# etcd证书创建：
    Note：为了保证通信安全，客户端(如etcdctl)与etcd 集群、etcd 集群之间的通信需要使用 TLS 加密。需安装 CloudFlare 的 PKI 工具集 cfssl 来生成 Certificate Authority (CA)证书和秘钥文件。
    安装加密工具 (在mac上，我走到这一步就失败了，一堆乱七八糟的异常，看着糟心)
    
>$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64  
$ chmod +x cfssl_linux-amd64  
$ mv cfssl_linux-amd64 /usr/local/bin/cfssl  
$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64  
$ chmod +x cfssljson_linux-amd64  
$ mv cfssljson_linux-amd64 /usr/local/bin/cfssljson  
$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64  
$ chmod +x cfssl-certinfo_linux-amd64  
$ mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo  

## 3.2 创建CA（如下操作均在/etc/etcd/ssl路径）
### 3.2.1 创建CA配置文件
    ca-config.json: 可以定义多个profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile
    signin: 表示该证书可用于签名其它证书；生成的ca.pem证书中CA=TRUE；
    server auth: 表示client 可以用该CA 对server 提供的证书进行校验；
    client auth: 表示server 可以用该CA 对client 提供的证书进行验证。
    
$ cat > ca-config.json <<EOF      
{  
  "signing": {                 
    "default": {  
      "expiry": "8760h"  
    },
    "profiles": {
      "kubernetes": {
      "expiry": "87600h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",      
          "client auth"       
        ]
      }
    }
  }
}
EOF

### 3.2.2 创建CA证书签名请求文件
    CN: Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名(User Name)；浏览器使用该字段验证网站是否合法；
    O: Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组(Group)；
    
$ cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",       
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

### 3.2.3 生成CA 证书和私钥：
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca  
$ ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem

### 3.2.4 创建etcd 证书签名请求:
    # hosts 字段指定授权使用该证书的 etcd 节点 IP
    # 192.168.7.101: 当前etcd节点ip，此文件需在其余etcd节点重新生成，重新生成当前节点证书
    
$ cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [                
    "127.0.0.1",
    "192.168.7.101",
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

### 3.2.5 生成etcd证书和私钥：
$ cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd  
$ ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem  
$ mkdir -p /etc/etcd/ssl
cp etcd.pem etcd-key.pem  ca.pem /etc/etcd/ssl/

### 3.2.6 其余节点生成证书：
    # 将CA证书分发到其余两台ETCD节点，IP地址注意修改为实际使用IP
    
$ for i in {2..3}; do ssh root@192.168.7.10$i "mkdir -p /etc/etcd/ssl/";done  
$ for i in {2..3}; do scp ca* root@192.168.7.10$i:/etc/etcd/ssl/;done

    # 重复以上d，e步，重新生成需要的证书

## 3.3 启动etcd集群：
### 3.3.1 创建etcd的systemd unit文件
$ mkdir -p /var/lib/etcd                                   # 必须先创建工作目录  
$ export NODE_IP=192.168.7.101                             # 当前节点IP  
$ export NODE_NAME=k8s-master01                            # 当前节点hostname  
$ export ETCD_NODES=k8s-master01=https://192.168.7.101:2380,k8s-master02=https://192.168.7.102:2380,k8s-master03=https://192.168.7.103:2380                                 # etcd 集群间通信的IP和端口  
$ cat > etcd.service <<EOF  
[Unit]  
Description=Etcd Server  
After=network.target  
After=network-online.target  
Wants=network-online.target  
Documentation=https://github.com/coreos  
 
[Service]  
Type=notify  
WorkingDirectory=/var/lib/etcd/                # 指定etcd的工作目录和数据目录为/var/lib/etcd，需要在启动服务前创建这个目录  
ExecStart=/usr/local/bin/etcd \\  
  --name=${NODE_NAME} \\  
  --cert-file=/etc/etcd/ssl/etcd.pem \\        # 为了保证通信安全，需要指定etcd 的公私钥(cert-file和key-file)、Peers通信的公私钥和CA 证书(peer-cert-file、peer-key-file、peer-trusted-ca-file)、客户端的CA证书(trusted-ca-file)；  
  --key-file=/etc/etcd/ssl/etcd-key.pem \\  
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\  
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\  
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \\  
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \\  
  --initial-advertise-peer-urls=https://${NODE_IP}:2380 \\  #  当前节点IP  
  --listen-peer-urls=https://${NODE_IP}:2380 \\  
  --listen-client-urls=https://${NODE_IP}:2379,http://127.0.0.1:2379 \\  
  --advertise-client-urls=https://${NODE_IP}:2379 \\  
  --initial-cluster-token=etcd-cluster-0 \\  
  --initial-cluster=${ETCD_NODES} \\  
  --initial-cluster-state=new \\              # --initial-cluster-state值为new时(初始化集群)，--name的参数值必须位于--initial-cluster列表中；  
  --data-dir=/var/lib/etcd  
Restart=on-failure  
RestartSec=5  
LimitNOFILE=65536  
 
[Install]  
WantedBy=multi-user.target  
EOF

### 3.3.2 启动etcd服务
    # 最先启动的etcd 进程会卡住一段时间，在所有的etcd 节点重复上面的步骤，直到所有的机器etcd 服务都已经启动。
    
$ systemctl daemon-reload  
$ systemctl enable etcd  
$ systemctl start etcd  
$ systemctl status etcd    

### 3.3.3 验证服务
$ etcdctl \
  --endpoints=https://${NODE_IP}:2379  \     # 任意etcd节点IP
  --ca-file=/etc/etcd/ssl/ca.pem \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  cluster-health
   
member 2df6611b3fa63449 is healthy: got healthy result from https://192.168.7.102:2379  
member 6ab4161ce2455154 is healthy: got healthy result from https://192.168.7.101:2379  
member fe146f2287e8b8c0 is healthy: got healthy result from https://192.168.7.103:2379  
cluster is healthy  
