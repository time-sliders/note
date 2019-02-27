### 使用 docker 下载 centos 最新的基础镜像
docker pull centos:latest

### 启动基础 docker 容器
docker run -dit --name centos_tomcat8_jdk8 --cap-add=ALL -v /Users/zhangwei/tmp/k8s/volume:/usr/local/volume centos_tomcat8_jdk8:v1 /bin/bash

#### 关于centos capabilities 启用的说明
http://man7.org/linux/man-pages/man7/capabilities.7.html
docker run -dit --name baseimage --cap-add=ALL -v /Users/zhangwei/tmp/k8s/volume:/usr/local/volume baseimage:v1 /bin/bash

### 安装 java
cd /usr/local/volume
tar -xvf jdk-8u201-linux-x64.tar.gz -C /usr/local
export JAVA_HOME=/usr/local/jdk1.8.0_201/
export PATH=$PATH:$JAVA_HOME/bin/

### 添加 tomcat 用户
#### 添加用户 tomcat
useradd -d /home/tomcat -s /usr/sbin/nologin tomcat
#### 添加用户组
groupadd tomcat
#### 将用户 tomcat 添加到组 tomcat
gpasswd -a tomcat tomcat

### 安装 tomcat
cd /usr/local/volume
tar -xvf apache-tomcat-8.5.32.tar.gz -C /home/tomcat
export TOMCAT_HOME=/home/tomcat/apache-tomcat-8.5.32/
export PATH=$PATH:$TOMCAT_HOME/bin

#### /home/tomcat 中所有文件的所有者和组更改为用户 tomcat 和组 tomcat
chown -R tomcat:tomcat /home/tomcat

### 开启 tomcat daemon 启动模式
yum -y install gcc cc make
cd /home/tomcat/apache-tomcat-8.5.32/bin/
tar -xvf commons-daemon-native.tar.gz
cd commons-daemon-1.1.0-native-src/unix/
./configure && make
cp ./jsvc ../..
cd ../..
sh ./daemon.sh start