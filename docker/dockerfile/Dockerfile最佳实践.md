# Dockerfile 最佳实践
本附录是笔者对 Docker 官方文档中 Best practices for writing Dockerfiles 的理解与翻译。

## 一般性的指南和建议
### 容器应该是短暂的

通过 Dockerfile 构建的镜像所启动的容器应该尽可能短暂（生命周期短）。「短暂」意味着可以停止和销毁容器，并且创建一个新容器并部署好所需的设置和配置工作量应该是极小的。

### 使用 .dockerignore 文件

使用 Dockerfile 构建镜像时最好是将 Dockerfile 放置在一个新建的空目录下。然后将构建镜像所需要的文件添加到该目录中。为了提高构建镜像的效率，你可以在目录下新建一个 .dockerignore 文件来指定要忽略的文件和目录。.dockerignore 文件的排除模式语法和 Git 的 .gitignore 文件相似。

### 使用多阶段构建

在 Docker 17.05 以上版本中，你可以使用 多阶段构建 来减少所构建镜像的大小。

### 避免安装不必要的包
为了降低复杂性、减少依赖、减小文件大小、节约构建时间，你应该避免安装任何不必要的包。例如，不要在数据库镜像中包含一个文本编辑器。

### 一个容器只运行一个进程
应该保证在一个容器中只运行一个进程。将多个应用解耦到不同容器中，保证了容器的横向扩展和复用。例如 web 应用应该包含三个容器：web应用、数据库、缓存。
如果容器互相依赖，你可以使用 Docker 自定义网络 来把这些容器连接起来。

### 镜像层数尽可能少
你需要在 Dockerfile 可读性（也包括长期的可维护性）和减少层数之间做一个平衡。

### 将多行参数排序
将多行参数按字母顺序排序（比如要安装多个包时）。这可以帮助你避免重复包含同一个包，更新包列表时也更容易。也便于 PRs 阅读和审查。建议在反斜杠符号 \ 之前添加一个空格，以增加可读性。  

### 构建缓存
在镜像的构建过程中，Docker 会遍历 Dockerfile 文件中的指令，然后按顺序执行。在执行每条指令之前，Docker 都会在缓存中查找是否已经存在可重用的镜像，如果有就使用现存的镜像，不再重复创建。如果你不想在构建过程中使用缓存，你可以在 `docker build` 命令中使用 `--no-cache=true` 选项。
  
在大多数情况下，只需要简单地对比 Dockerfile 中的指令和子镜像。然而，有些指令需要更多的检查和解释。

对于 ADD 和 COPY 指令，镜像中对应文件的内容也会被检查，每个文件都会计算出一个校验和。文件的最后修改时间和最后访问时间不会纳入校验。在缓存的查找过程中，会将这些校验和和已存在镜像中的文件校验和进行对比。如果文件有任何改变，比如内容和元数据，则缓存失效。

除了 ADD 和 COPY 指令，缓存匹配过程不会查看临时容器中的文件来决定缓存是否匹配。例如，当执行完 `RUN apt-get -y update` 指令后，容器中一些文件被更新，但 Docker 不会检查这些文件。这种情况下，只有指令字符串本身被用来匹配缓存。

一旦缓存失效，所有后续的 Dockerfile 指令都将产生新的镜像，缓存不会被使用。

## Dockerfile 指令
下面针对 Dockerfile 中各种指令的最佳编写方式给出建议。
### FROM
尽可能使用当前官方仓库作为你构建镜像的基础。推荐使用 Alpine 镜像，因为它被严格控制并保持最小尺寸（目前小于 5 MB），但它仍然是一个完整的发行版。
### LABEL
你可以给镜像添加标签来帮助组织镜像、记录许可信息、辅助自动化构建等。每个标签一行，由 LABEL 开头加上一个或多个标签对。下面的示例展示了各种不同的可能格式。# 开头的行是注释内容。

注意：如果你的字符串中包含空格，必须将字符串放入引号中或者对空格使用转义。如果字符串内容本身就包含引号，必须对引号使用转义。

    # Set one or more individual labels  
    LABEL com.example.version="0.0.1-beta"  
    LABEL vendor="ACME Incorporated"  
    LABEL com.example.release-date="2015-02-12" 
    LABEL com.example.version.is-production=""  
 
一个镜像可以包含多个标签，但建议将多个标签放入到一个 LABEL 指令中。
 
    # Set multiple labels at once, using line-continuation characters to break long lines
    LABEL vendor=ACME\ Incorporated \
        com.example.is-beta= \
        com.example.is-production="" \
        com.example.version="0.0.1-beta" \
        com.example.release-date="2015-02-12"
 
### RUN
为了保持 Dockerfile 文件的可读性，可理解性，以及可维护性，建议将长的或复杂的 RUN 指令用反斜杠 \ 分割成多行。
 
将 apt-get update 放在一条单独的 RUN 声明中会导致缓存问题以及后续的 apt-get install 失败。


# ADD 和 COPY
虽然 ADD 和 COPY 功能类似，但一般优先使用 COPY。因为它比 ADD 更透明。COPY 只支持简单将本地文件拷贝到容器中，使用 COPY。

# ENTRYPOINT
ENTRYPOINT 的最佳用处是设置镜像的主命令，允许将镜像当成命令本身来运行（用 CMD 提供默认选项）。
例如，下面的示例镜像提供了命令行工具 s3cmd:
ENTRYPOINT ["s3cmd"]

# VOLUME
VOLUME 指令用于暴露任何数据库存储文件，配置文件，或容器创建的文件和目录。强烈建议使用 VOLUME 来管理镜像中的可变部分和用户可以改变的部分。

# USER
如果某个服务不需要特权执行，建议使用 USER 指令切换到非 root 用户。先在 Dockerfile 中使用类似 RUN groupadd -r postgres && useradd -r -g postgres postgres 的指令创建用户和用户组。

    注意：在镜像中，用户和用户组每次被分配的 UID/GID 都是不确定的，下次重新构建镜像时被分配到的 UID/GID 可能会不一样。如果要依赖确定的 UID/GID，你应该显示的指定一个 UID/GID。
    你应该避免使用 sudo，因为它不可预期的 TTY 和信号转发行为可能造成的问题比它能解决的问题还多。如果你真的需要和 sudo 类似的功能（例如，以 root 权限初始化某个守护进程，以非 root 权限执行它），你可以使用 gosu。
    最后，为了减少层数和复杂度，避免频繁地使用 USER 来回切换用户。
    
# WORKDIR
为了清晰性和可靠性，你应该总是在 WORKDIR 中使用绝对路径。另外，你应该使用 WORKDIR 来替代类似于 RUN cd ... && do-something 的指令，后者难以阅读、排错和维护。