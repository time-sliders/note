# FROM 
指定基础镜像 以一个镜像为基础，在其上进行定制，Dockerfile 中 FROM 是必备的指令，并且必须是第一条指令

    在 Docker Store 上有非常多的高质量的官方镜像，有可以直接拿来使用的服务类的镜像，如 nginx、redis、mongo、mysql、httpd、php、tomcat 等；也有一些方便开发、构建、运行各种语言应用的镜像，如 node、openjdk、python、ruby、golang 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。
    如果没有找到对应服务的镜像，官方镜像中还提供了一些更为基础的操作系统镜像，如 ubuntu、debian、centos、fedora、alpine 等，这些操作系统的软件库为我们提供了更广阔的扩展空间。
    除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 scratch。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。
    如果你以 scratch 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。
    不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如 swarm、coreos/etcd。对于 Linux 下静态编译的程序来说，并不需要有操作系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 FROM scratch 会让镜像体积更加小巧。使用 Go 语言 开发的应用很多会使用这种方式来制作镜像，这也是为什么有人认为 Go 是特别适合容器微服务架构的语言的原因之一。
# RUN 
执行命令
RUN 指令是用来执行命令行命令的。  
其格式有两种：  
* shell 格式：RUN <命令>，就像直接在命令行中输入的命令一样。刚才写的 Dockerfile 中的 RUN 指令就是这种格式。
    > RUN echo Hello, Docker
* exec 格式：RUN ["可执行文件", "参数1", "参数2"]，这更像是函数调用中的格式。  

    
    Dockerfile 中每一个指令都会建立一层，RUN 也不例外。每一个 RUN 的行为，就和刚才我们手工建立镜像的过程一样：新建立一层，在其上执行这些命令，执行结束后，commit 这一层的修改，构成新的镜像。
    而上面的这种写法，创建了 7 层镜像。这是完全没有意义的，而且很多运行时不需要的东西，都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。 这是很多初学 Docker 的人常犯的一个错误。
    Union FS 是有最大层数限制的，比如 AUFS，曾经是最大不得超过 42 层，现在是不得超过 127 层。

**多个RUN命令应该通过 && 合并执行**

    Dockerfile 支持 Shell 类的行尾添加 \ 的命令换行方式，以及行首 # 进行注释的格式。良好的格式，比如换行、缩进、注释等，会让维护、排障更为容易，这是一个比较好的习惯。
    此外，还可以看到这一组命令的最后添加了清理工作的命令，删除了为了编译构建所需要的软件，清理了所有下载、展开的文件，并且还清理了 apt 缓存文件。这是很重要的一步，我们之前说过，镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

# ENV 
设置环境变量  
格式有两种：  
* `ENV <key> <value>`
* `ENV <key1>=<value1> <key2>=<value2>...`

 无论是后面的其它指令，如 RUN，还是运行时的应用，都可以直接使用这里定义的环境变量。

下列指令可以支持环境变量展开： ADD、COPY、ENV、EXPOSE、LABEL、USER、WORKDIR、VOLUME、STOPSIGNAL、ONBUILD。

# ARG
构建参数  
格式： `ARG <参数名>[=<默认值>]`
 
构建参数和 ENV 的效果一样，都是设置环境变量。所不同的是，ARG 所设置的构建环境的环境变量，在将来容器运行时是不会存在这些环境变量的。但是不要因此就使用 ARG 保存密码之类的信息，因为 docker history 还是可以看到所有值的。
Dockerfile 中的 ARG 指令是定义参数名称，以及定义其默认值。该默认值可以在构建命令 docker build 中用 --build-arg <参数名>=<值> 来覆盖。
 
 
# CMD 
容器启动命令  
CMD 指令的格式和 RUN 相似，也是两种格式：  
* shell 格式：`CMD <命令>`
* exec 格式：`CMD ["可执行文件", "参数1", "参数2"...]`
 
参数列表格式：CMD ["参数1", "参数2"...]。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。  
之前介绍容器的时候曾经说过，Docker 不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数。CMD 指令就是用于指定默认的容器主进程的启动命令的。

**在运行时可以指定新的命令来替代镜像设置中的这个默认命令**

在指令格式上，一般推荐使用 exec 格式，这类格式在解析时会被解析为 JSON 数组，因此一定要使用双引号 "，而不要使用单引号。  
如果使用 shell 格式的话，实际的命令会被包装为 sh -c 的参数的形式进行执行。比如：  
`CMD echo $HOME`  
在实际执行中，会将其变更为：  
`CMD [ "sh", "-c", "echo $HOME" ]`  

> Docker容器与虚拟机的区别
    提到 CMD 就不得不提容器中应用在前台执行和后台执行的问题。这是初学者常出现的一个混淆。  
    Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用 upstart/systemd 去启动后台服务，容器内没有后台服务的概念。  
    所以不要把 Docker 和虚拟机弄混，Docker 容器只是一个进程而已，只不过利用镜像提供的 rootfs 提供了调用所需的 userland 库支持，使得进程可以在受控环境下运行而已，它并没有虚拟出一个机器出来。  
    一些初学者将 CMD 写为：  
    `CMD service nginx start`  
    然后发现容器执行后就立即退出了。甚至在容器内去使用 systemctl 命令结果却发现根本执行不了。这就是因为没有搞明白前台、后台的概念，没有区分容器和虚拟机的差异，依旧在以传统虚拟机的角度去理解容器。  
    **对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。**  
    而使用 `service nginx start` 命令，则是希望 upstart 来以后台守护进程形式启动 nginx 服务。而刚才说了  
     `CMD service nginx start`   
     会被理解为  
     `CMD [ "sh", "-c", "service nginx start"]`  
     因此主进程实际上是 sh。那么当   
     `service nginx start`   
     命令结束后，sh 也就结束了，sh 作为主进程退出了，自然就会令容器退出。  正确的做法是直接执行 nginx 可执行文件，并且要求以前台形式运行。比如：  
     `CMD ["nginx", "-g", "daemon off;"]`
 
# ENTRYPOINT 
入口点  
ENTRYPOINT 的格式和 RUN 指令格式一样，分为 exec 格式和 shell 格式。  
`<ENTRYPOINT> "<CMD>"`

    1.如果 ENTRYPOINT 不是一个等待指令，比如 echo 之类的，在 container 启动并执行完毕之后，容器会马上被关闭掉
    2.如果 ENTRYPOINT 启动的是一个 daemon 守护进程，而非主进程，则也会马上关闭
    3.如果 image 有 ENTRYPOINT，且不是 bash, 则 attach 上去是没有 bash 的，此时需要通过 exec 命令来在容器内运行一个 bash
    docker exec -it CONTAINER_ID /bin/bash

**同 CMD 不同的是， ENTRYPOINT 指令不会被 docker run 命令所覆盖，而是将 docker run 指定的参数当做 ENTRYPOINT 指定命令的参数**
  
#### 示例

**Dockerfile**

    FROM baseimage:latest
    
    RUN cd /home/tomcat/apache-tomcat-8.5.32/ \
        && rm -rf /home/tomcat/webapps \
        && mkdir /home/tomcat/webapps \
        && yum install -y net-tools.x86_64
    
    COPY xxxxxxxxxxx.war /home/tomcat/apache-tomcat-8.5.32/webapps
    
    COPY hosts /etc/
    
    ENTRYPOINT ["daemon.sh"]

**start cmd**

    docker run -dit --name xxxxxxxxxxx --cap-add=ALL -v /data/www/logs:/data/www/logs xxxxxxxxxxx:v2 run
    docker ps -a
    
    CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
    538905a7c4c5        xxxxxxxxxxx:v2      "daemon.sh run"     4 seconds ago       Up 8 seconds                            xxxxxxxxxxx
 
# VOLUME 
定义匿名卷  
格式为：  
* `VOLUME ["<路径1>", "<路径2>"...]`
* `VOLUME <路径>`
 
容器运行时应该尽量保持容器存储层不发生写操作，对于数据库类需要保存动态数据的应用，其数据库文件应该保存于卷(volume)中。为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 Dockerfile 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。  
`VOLUME /data`  
这里的 /data 目录就会在运行时自动挂载为匿名卷，任何向 /data 中写入的信息都不会记录进容器存储层，从而保证了容器存储层的无状态化。当然，运行时可以覆盖这个挂载设置。比如：  
`docker run -d -v mydata:/data xxxx`  
在这行命令中，就使用了 mydata 这个命名卷挂载到了 /data 这个位置，替代了 Dockerfile 中定义的匿名卷的挂载配置。
 
# WORKDIR 
指定工作目录  
格式为: `WORKDIR <工作目录路径>`
 
使用 WORKDIR 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，WORKDIR 会帮你建立目录。

> 之前提到一些初学者常犯的错误是把 Dockerfile 等同于 Shell 脚本来书写，这种错误的理解还可能会导致出现下面这样的错误：  
`RUN cd /app`  
`RUN echo "hello" > world.txt`  
如果将这个 Dockerfile 进行构建镜像运行后，会发现找不到 /app/world.txt 文件，或者其内容不是 hello。原因其实很简单，在 Shell 中，连续两行是同一个进程执行环境，因此前一个命令修改的内存状态，会直接影响后一个命令；而在 Dockerfile 中，这两行 RUN 命令的执行环境根本不同，是两个完全不同的容器。这就是对 Dockerfile 构建分层存储的概念不了解所导致的错误。 
之前说过每一个 RUN 都是启动一个容器、执行命令、然后提交存储层文件变更。第一层 RUN cd /app的执行仅仅是当前进程的工作目录变更，一个内存上的变化而已，其结果不会造成任何文件变更。而到第二层的时候，启动的是一个全新的容器，跟第一层的容器更完全没关系，自然不可能继承前一层构建过程中的内存变化。  
因此如果需要改变以后各层的工作目录的位置，那么应该使用 WORKDIR 指令。  

# USER
指定当前用户  
格式：`USER <用户名>`
 
USER 指令和 WORKDIR 相似，都是改变环境状态并影响以后的层。WORKDIR 是改变工作目录，USER则是改变之后层的执行 RUN, CMD 以及 ENTRYPOINT 这类命令的身份。  
当然，和 WORKDIR 一样，USER 只是帮助你切换到指定用户而已，这个用户必须是事先建立好的，否则无法切换。  
`RUN groupadd -r redis && useradd -r -g redis redis`  
`USER redis`  

> 如果以 root 执行的脚本，在执行期间希望改变身份，比如希望以某个已经建立好的用户来运行某个服务进程，不要使用 su 或者 sudo，这些都需要比较麻烦的配置，而且在 TTY 缺失的环境下经常出错。建议使用 gosu。  
    # 建立 redis 用户，并使用 gosu 换另一个用户执行命令  
    `RUN groupadd -r redis && useradd -r -g redis redis`  
    # 下载 gosu  
    `RUN wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/1.7/gosu-amd64"  
    && chmod +x /usr/local/bin/gosu 
    && gosu nobody true`  
    # 设置 CMD，并以另外的用户执行  
    `CMD [ "exec", "gosu", "redis", "redis-server" ]`
 
# COPY
复制文件  
格式：  
`COPY <源路径>... <目标路径>`
`COPY ["<源路径1>",... "<目标路径>"]`
 
# ADD
更高级的复制文件  
ADD 指令和 COPY 的格式和性质基本一致。

**如果想把一个tar.gz 包解压之后的东西拷贝到 镜像中，建议用 ADD**

    但是在 COPY 基础上增加了一些功能。
    比如 <源路径> 可以是一个 URL，这种情况下，Docker 引擎会试图去下载这个链接的文件放到 <目标路径> 去。下载后的文件权限自动设置为 600，如果这并不是想要的权限，那么还需要增加额外的一层 RUN 进行权限调整，另外，如果下载的是个压缩包，需要解压缩，也一样还需要额外的一层 RUN 指令进行解压缩。所以不如直接使用 RUN 指令，然后使用 wget 或者 curl 工具下载，处理权限、解压缩、然后清理无用文件更合理。因此，这个功能其实并不实用，而且不推荐使用。
    如果 <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。

# EXPOSE 声明端口
格式为 `EXPOSE <端口1> [<端口2>...]`
 
EXPOSE 指令是声明运行时容器提供服务端口，这只是一个声明，在运行时并不会因为这个声明应用就会开启这个端口的服务。在 Dockerfile 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 docker run -P时，会自动随机映射 EXPOSE 的端口。  

要将 EXPOSE 和在运行时使用 -p <宿主端口>:<容器端口> 区分开来。-p，是映射宿主端口和容器端口，换句话说，就是将容器的对应端口服务公开给外界访问，而 EXPOSE 仅仅是声明容器打算使用什么端口而已，并不会自动在宿主进行端口映射。
 
# 镜像的实现原理
### Docker 镜像是怎么实现增量的修改和维护的？

每个镜像都由很多层次构成，Docker 使用 Union FS 将这些不同的层结合到一个镜像中去。
通常 Union FS 有两个用途, 一方面可以实现不借助 LVM、RAID 将多个 disk 挂到同一个目录下,另一个更常用的就是将一个只读的分支和一个可写的分支联合在一起，Live CD 正是基于此方法可以允许在镜像不变的基础上允许用户在其上进行一些写操作。  
Docker 在 AUFS 上构建的容器也是利用了类似的原理。
 

