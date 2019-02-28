[原文地址](https://www.cnblogs.com/dengyungao/p/8426878.html)

在早期的 UNIX 系统中，各个厂家各自定义了自己的 UNIX 系统文件目录，比较混乱。Linux 面世不久后，对文件目录进行了标准化，于1994年对根文件目录做了统一的规范，推出 FHS ( Filesystem Hierarchy Standard ) 的 Linux 文件系统层次结构标准。FHS 标准规定了 Linux 根目录各文件夹的名称及作用，统一了Linux界命名混乱的局面。无论何种版本的 Linux 发行版，桌面、应用是 Linux 的外衣，文件组织、目录结构才是Linux的内心。

FHS（英文:Filesystem Hierarchy Standard 中文:文件系统层次结构标准），多数 Linux 版本采用这种文件组织形式，FHS 定义了系统中每个区域的用途、所需要的最小构成的文件和目录同时还给出了例外处理与矛盾处理。
> FHS 定义了两层规范:  
        第一层是， / 下面的各个目录应该要放什么文件数据，例如 /etc应该要放置设置文件，/bin 与 /sbin 则应该要放置可执行文件等等。  
        第二层则是针对 /usr 及 /var 这两个目录的子目录来定义。例如 /var/log 放置系统登录文件、/usr/share 放置共享数据等等。  

FHS 是根据以往无数 Linux 用户和开发者的经验总结出来的，并且会维持更新，FHS 依据文件系统使用的频繁与否以及是否允许用户随意改动

    /:根目录，一般根目录下只存放目录，不要存放件，/etc、/bin、/dev、/lib、/sbin应该和根目录放置在一个分区中  

    /bin: /usr/bin: 可执行二进制文件的目录，如常用的命令ls、tar、mv、cat
  
    /boot:放置linux系统启动时用到的一些文件。/boot/vmlinuz 为 linux 的内核文件，以及 /boot/gurb。建议单独分区，分区大小100M即可
  
    /dev:存放linux系统下的设备文件，访问该目录下某个文件，相当于访问某个设备，常用的是挂载光驱 mount /dev/cdrom /mnt。
  
    /etc:系统配置文件存放的目录，不建议在此目录下存放可执行文件。
  
    /home:系统默认的用户家目录，新增用户账号时，用户的家目录都存放在此目录下，~表示当前用户的家目录，~edu 表示用户 edu 的家目录。建议单独分区，并设置较大的磁盘空间，方便用户存放数据
  
    /lib: /usr/lib: /usr/local/lib:系统使用的函数库的目录，程序在执行过程中，需要调用一些额外的参数时需要函数库的协助，比较重要的目录为 /lib/modules。
  
    /lost+fount:系统异常产生错误时，会将一些遗失的片段放置于此目录下，通常这个目录会自动出现在装置目录下。如加载硬盘于 /disk 中，此目录下就会自动产生目录 /disk/lost+found
  
    /mnt: /media:光盘默认挂载点，通常光盘挂载于 /mnt/cdrom 下，也不一定，可以选择任意位置进行挂载。
  
    /opt:给主机额外安装软件所摆放的目录。额外安装的可选应用程序包所放置的位置。以前的 Linux 系统中，习惯放置在 /usr/local 目录下
  
    /proc:此目录的数据都在内存中，如系统核心，外部设备，网络状态，由于数据都存放于内存中，所以不占用磁盘空间，可直接访问这个目录来获取系统信息。  
        /proc/cpuinfo
        /proc/interrupts
        /proc/dma
        /proc/ioports
        /proc/net/*

    /root:系统管理员root的家目录，系统第一个启动的分区为 /，所以最好将 /root和 /放置在一个分区下。  
    /sbin: /usr/sbin: /usr/local/sbin:放置系统管理员使用的可执行命令，如fdisk、shutdown、mount 等。与 /bin 不同的是，这几个目录是给系统管理员 root使用的命令，一般用户只能"查看"而不能设置和使用。  
    /tmp:一般用户或正在执行的程序临时存放文件的目录,任何人都可以访问,重要数据不可放置在此目录下  
    /srv:服务启动之后需要访问的数据目录，如 www 服务需要访问的网页数据存放在 /srv/www 内。  

    /usr:应用程序存放目录，要用到的应用程序和文件几乎都在这个目录，其中包含
        /usr/bin 存放应用程序
        /usr/sbin 超级用户的一些管理程序
        /usr/share 存放共享数据
            /usr/share/doc: 系统说明文件存放目录。
            /usr/share/man: 程序说明文件存放目录，使用 man ls 时会查询 /usr/share/man/man1/ls.1.gz 的内容建议单独分区，设置较大的磁盘空间
        /usr/lib 存放不能直接运行的，却是许多程序运行所必需的一些函数库文件
        /usr/local: 存放软件升级包 一般java也会安装在该目录下
            /usr/local/bin 本地增加的命令
            /usr/local/lib 本地增加的库根文件系统
        /usr/doc linux文档
        /usr/include linux下开发和编译应用程序所需要的头文件
        /usr/man 帮助文档
        /usr/src 源代码，linux内核的源代码就放在/usr/src/linux里

    /var:放置系统执行过程中经常变化的文件
        /var/log 随时更改的日志文件
        /var/log/message 所有的登录文件存放目录
        /var/spool/mail 邮件存放的目录
        /var/run 程序或服务启动后，其PID存放在该目录下
