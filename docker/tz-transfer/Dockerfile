FROM baseimage:latest

yum install -y net-tools.x86_64

RUN cd /home/tomcat/apache-tomcat-8.5.32/ && rm -rf webapps && mkdir webapps
COPY tz-transfer.war /home/tomcat/apache-tomcat-8.5.32/webapps

ENTRYPOINT ["sh","/home/tomcat/apache-tomcat-8.5.32/bin/daemon.sh","run"]