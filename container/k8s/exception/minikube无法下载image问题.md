

**特别强调：Image 的 version 不要打 latest 版本，因为 latest 总会去镜像仓库下载 Image**     
    
    k8s.gcr.io/kube-apiserver:v1.13.2  
    k8s.gcr.io/kube-controller-manager:v1.13.2  
    k8s.gcr.io/kube-scheduler:v1.13.2  
    k8s.gcr.io/kube-proxy:v1.13.2  
    k8s.gcr.io/pause:3.1  
    k8s.gcr.io/etcd:3.2.24  
    k8s.gcr.io/coredns:1.2.6  
    **Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)**

### docker.io仓库对google的容器做了镜像，可以通过下列命令下拉取相关镜像
`docker pull mirrorgooglecontainers/kube-apiserver:v1.13.2`  
`docker pull mirrorgooglecontainers/kube-controller-manager:v1.13.2`    
`docker pull mirrorgooglecontainers/kube-scheduler:v1.13.2`  
`docker pull mirrorgooglecontainers/kube-proxy:v1.13.2`  
`docker pull mirrorgooglecontainers/pause:3.1`  
`docker pull mirrorgooglecontainers/etcd:3.2.24`    
`docker pull coredns/coredns:1.2.6`  

### 通过docker tag命令来修改镜像的标签

`docker tag docker.io/mirrorgooglecontainers/kube-apiserver:v1.13.2 k8s.gcr.io/kube-apiserver:v1.13.2`  
`docker tag docker.io/mirrorgooglecontainers/kube-controller-manager:v1.13.2 k8s.gcr.io/kube-controller-manager:v1.13.2`    
`docker tag docker.io/mirrorgooglecontainers/kube-scheduler:v1.13.2 k8s.gcr.io/kube-scheduler:v1.13.2`  
`docker tag docker.io/mirrorgooglecontainers/kube-proxy:v1.13.2 k8s.gcr.io/kube-proxy:v1.13.2`  
`docker tag docker.io/mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1`  
`docker tag docker.io/mirrorgooglecontainers/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24`  
`docker tag docker.io/coredns/coredns:1.2.6 k8s.gcr.io/coredns:1.2.6`  

### k8s docker image init cmd
`docker pull mirrorgooglecontainers/kube-apiserver:v1.13.2 && docker pull mirrorgooglecontainers/kube-controller-manager:v1.13.2 && docker pull mirrorgooglecontainers/kube-scheduler:v1.13.2 && docker pull mirrorgooglecontainers/kube-proxy:v1.13.2 && docker pull mirrorgooglecontainers/pause:3.1 && docker pull mirrorgooglecontainers/etcd:3.2.24 && docker pull coredns/coredns:1.2.6 && docker tag docker.io/mirrorgooglecontainers/kube-apiserver:v1.13.2 k8s.gcr.io/kube-apiserver:v1.13.2 && docker tag docker.io/mirrorgooglecontainers/kube-controller-manager:v1.13.2 k8s.gcr.io/kube-controller-manager:v1.13.2 && docker tag docker.io/mirrorgooglecontainers/kube-scheduler:v1.13.2 k8s.gcr.io/kube-scheduler:v1.13.2 && docker tag docker.io/mirrorgooglecontainers/kube-proxy:v1.13.2 k8s.gcr.io/kube-proxy:v1.13.2 && docker tag docker.io/mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1 && docker tag docker.io/mirrorgooglecontainers/etcd:3.2.24 k8s.gcr.io/etcd:3.2.24 && docker tag docker.io/coredns/coredns:1.2.6 k8s.gcr.io/coredns:1.2.6 && docker rmi mirrorgooglecontainers/kube-apiserver:v1.13.2 && docker rmi mirrorgooglecontainers/kube-controller-manager:v1.13.2 && docker rmi mirrorgooglecontainers/kube-scheduler:v1.13.2 && docker rmi mirrorgooglecontainers/kube-proxy:v1.13.2 && docker rmi mirrorgooglecontainers/pause:3.1 && docker rmi mirrorgooglecontainers/etcd:3.2.24 && docker rmi coredns/coredns:1.2.6`
