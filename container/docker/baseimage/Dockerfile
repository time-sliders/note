FROM centos:latest

ENV JAVA_HOME=/usr/local/jdk1.8.0_201
ENV TOMCAT_HOME=/home/tomcat/apache-tomcat-8.5.32
ENV PATH=$JAVA_HOME/bin:$TOMCAT_HOME/bin:$PATH
RUN yum -y install gcc cc make
RUN useradd -d /home/tomcat -s /usr/sbin/nologin tomcat \
    && gpasswd -a tomcat tomcat \
    && chown -R tomcat:tomcat /home/tomcat

ADD jdk-8u201-linux-x64.tar.gz /usr/local
ADD apache-tomcat-8.5.32.tar.gz /home/tomcat
RUN cd /home/tomcat/apache-tomcat-8.5.32/bin/ \
    && tar -xvf commons-daemon-native.tar.gz \
    && cd commons-daemon-1.1.0-native-src/unix/ \
    && ./configure \
    && make \
    && cp ./jsvc ../..

ENTRYPOINT ["daemon.sh","run"]
