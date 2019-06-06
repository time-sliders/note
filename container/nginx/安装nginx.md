# 安装 brew 
Homebrew简称brew，是Mac OSX上的软件包管理工具，能在Mac中方便的安装软件或者卸载软件，可以说Homebrew就是mac下的apt-get、yum神器

`ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`

耐心等待安装完毕
以下为安装完毕提示：

    ==> Homebrew has enabled anonymous aggregate user behaviour analytics.
    Read the analytics documentation (and how to opt-out) here:
      https://docs.brew.sh/Analytics.html
    ==> Next steps:
    - Run `brew help` to get started
    - Further documentation:
        https://docs.brew.sh
    
# 安装ng
终端输入:   
`cd ~`  
`brew install nginx`

耐心等待安装完成即可  
以下为安装完毕提示

    Docroot is: /usr/local/var/www
    The default port has been set in /usr/local/etc/nginx/nginx.conf to 8080 so that
    nginx can run without sudo.
    nginx will load all files in /usr/local/etc/nginx/servers/.
    To have launchd start nginx now and restart at login:
      brew services start nginx
    Or, if you don't want/need a background service you can just run:
      nginx
    ==> Summary
    /usr/local/Cellar/nginx/1.12.2: 8 files, 1MB, built in 42 seconds
    
# 其他命令
卸载ng：`brew uninstall nginx`    
卸载之后，配置文件好像还在，需要手动删除

# 相关参数以及命令
    nginx -h
    nginx version: nginx/1.12.2
    Usage: nginx [-?hvVtTq] [-s signal] [-c filename] [-p prefix] [-g directives]
    Options:
      -?,-h         : this help
      -v            : show version and exit
      -V            : show version and configure options then exit
      -t            : test configuration and exit
      -T            : test configuration, dump it and exit
      -q            : suppress non-error messages during configuration testing
      -s signal     : send signal to a master process: stop, quit, reopen, reload
      -p prefix     : set prefix path (default: /usr/local/Cellar/nginx/1.12.2/)
      -c filename   : set configuration file (default: /usr/local/etc/nginx/nginx.conf)
      -g directives : set global directives out of configuration file

# 添加 servers 配置
nginx.conf 在文件尾部默认 `include servers/*;`  即表示nginx服务在会默认加载 `/usr/local/etc/nginx/servers` 
目录下的所有服务配置文件

以下配置，可以使所有访问 www.xxx.com 的请求，都代理请求到127.0.0.1:8261的应用上

    server {
        listen       80;
        listen       443;
        server_name  www.xxx.com ;
        location / {
            root   html;
            proxy_pass  http://abc/;
            index  index.html index.htm;
        }
    }
    upstream abc {
             keepalive 512;
             server 127.0.0.1:8261;
    }

以下配置，可以使所有访问 www.xxx.com 的请求，都去服务器上 /usr/local/etc/nginx/html/ 目录下 找对应的 index.html。

    server {
        listen       80;
        listen       443;
        server_name  www.xxx.com;
        location / {
            root   /usr/local/etc/nginx/html/;
            index  index.html index.htm;
            autoindex on;
        }
    }
    
注意事项：
1.将端口改成80之后，mac系统，需要sudo 重新启动nginx才行，普通用户会提示   
`nginx: [emerg] bind() to 0.0.0.0:80 failed (13: Permission denied)`
