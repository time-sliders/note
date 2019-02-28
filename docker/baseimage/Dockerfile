FROM centos:latest

ENV JAVA_HOME=/usr/local/jdk1.8.0_201
ENV TOMCAT_HOME=/home/tomcat/apache-tomcat-8.5.38
ENV PATH=$JAVA_HOME/bin:$TOMCAT_HOME/bin:$PATH

RUN useradd -d /home/tomcat -s /usr/sbin/nologin tomcat \
    && gpasswd -a tomcat tomcat

COPY jdk-8u201-linux-x64.tar.gz /usr/local/
COPY apache-tomcat-8.5.38.tar.gz /home/tomcat/

RUN cd /usr/local \
    && tar -xvf jdk-8u201-linux-x64.tar.gz \
    && rm jdk-8u201-linux-x64.tar.gz

RUN cd /home/tomcat \
    && tar -xvf apache-tomcat-8.5.38.tar.gz \
    && rm apache-tomcat-8.5.38.tar.gz \
    && chown -R tomcat:tomcat /home/tomcat

RUN yum -y install gcc cc make 

RUN cd /home/tomcat/apache-tomcat-8.5.38/bin/ \
    && tar -xvf commons-daemon-native.tar.gz \
    && cd commons-daemon-1.1.0-native-src/unix/ \
    && ./configure \
    && make \
    && cp ./jsvc ../..

ENTRYPOINT ["sh","/home/tomcat/apache-tomcat-8.5.38/bin/daemon.sh","run"]
