下载地址：<https://dev.mysql.com/downloads/mysql/>  
开发手册：<https://dev.mysql.com/doc/>  
安装完毕页面，有账号密码，不要直接关闭

### 配置mysql环境变量
mysql执行文件目录  
`/usr/local/mysql/bin`  
设置环境变量   
打开 /etc/profile 在文件尾部添加 下面一行  
`export PATH=$PATH:/usr/local/mysql/bin`  
当前目录执行命令  
`source profile`  
使环境变量生效

### 登陆Mysql
用户名密码，是安装结果页面提供的，下面命令注意不要有空格  
`mysql -uroot -pI/3,8E&jN>IE`
  
    注意： 如果密码里面有特殊字符，可以把上面命令写到.sh脚本里面，然后对特殊字符 比如 & > ! 等 通过 \& \> \! 进行转义，之后再运行.sh脚本即可

### 修改默认密码
刚安装完，必须要重置ROOT密码。  
一般本地玩，直接密码改成 root/root 好记，命令如下  
`ALTER USER 'root'@'localhost' identified by 'root’;`  
注意引号不要写错了  
