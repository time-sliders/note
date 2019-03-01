FROM baseimage:latest

RUN yum install -y net-tools.x86_64
RUN cd /home/tomcat/apache-tomcat-8.5.32/ \
	&& rm -rf /home/tomcat/webapps \
	&& mkdir /home/tomcat/webapps

ADD tz-transfer.war /home/tomcat/apache-tomcat-8.5.32/webapps
COPY hosts /etc/

ENTRYPOINT ["daemon.sh","run"]