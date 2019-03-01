### 根据 docker image 启动容器

    docker run -dit --name centos_tomcat8_jdk8 -v /Users/zhangwei/tmp/k8s/volume:/usr/local/volume centos:latest /bin/bash  
    
    -v, --volume list                    Bind mount a volume 前一个目录指的是主机目录 第二个目录指的是容器内的映射目录  
    -d, --detach                         Run container in background and print container ID  
    -i, --interactive                    Keep STDIN open even if not attached  
    -t, --tty                            Allocate a pseudo-TTY  
    --name string                        Assign a name to the container
    
### 退出容器
1. 退出后依然保留容器进程 `Ctrl + P + Q`
2. 退出后不保留容器 `exit`

### 查看容器错误日志
`docker logs CONTAINER_ID`

