```
#!/bin/bash
RMI_SERVER_NAME=$(ifconfig  eth0|grep 'inet addr'|awk -F '[ :]+' '{print $4}') 
DISCONF_CONF_SERVER_HOST=disconf.tbj.com
DISCONF_APP=tz-position
DISCONF_ENV=online_formal
DISCONF_VERSION=1_0_0_0
DISCONF_ENABLE_REMOTE_CONF=true
DISCONF_USER_DEFINE_DOWNLOAD_DIR=/data/www/temp
DISCONF_DEBUG=false
DISCONF_ENABLE_LOCAL_DOWNLOAD_DIR_IN_CLASS_PATH=true

export CLASSPATH=".:/usr/java/jdk/jre/lib/rt.jar:/lib/dt.jar:/lib/tools.jar"
export JAVA_HOME="/usr/java/jdk"
export PATH=":/sbin:/bin:/usr/sbin:/usr/bin:/usr/X11R6/bin:$JAVA_HOME/bin:/root/bin"

SET_JVM_Xms=4048m
SET_JVM_Xmx=4048m
SET_JVM_Xmn=768m
SET_JVM_Xss=256k
SET_JVM_XXPermSize=64m
SET_JVM_XXMaxPermSize=256m
SET_JVM_HeapDumpPath=/var/heapDump
JAVA_OPTS="$JAVA_OPTS -server -Xms${SET_JVM_Xms} -Xmx${SET_JVM_Xmx} -Xmn${SET_JVM_Xmn} -Xss${SET_JVM_Xss} -XX:PermSize=${SET_JVM_XXPermSize} -XX:MaxPermSize=${SET_JVM_XXMaxPermSize} -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/www/temp -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:+CMSClassUnloadingEnabled"
CATALINA_OPTS="$CATALINA_OPTS -Dcom.sun.management.jmxremote -Djava.rmi.server.hostname=${RMI_SERVER_NAME} -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dfoo.jmx=true -Dfoo.jmx.detailed=true  -Ddisconf.conf_server_host=${DISCONF_CONF_SERVER_HOST} -Ddisconf.app=${DISCONF_APP} -Ddisconf.env=${DISCONF_ENV} -Ddisconf.version=${DISCONF_VERSION} -Ddisconf.enable.remote.conf=${DISCONF_ENABLE_REMOTE_CONF} -Ddisconf.user_define_download_dir=${DISCONF_USER_DEFINE_DOWNLOAD_DIR} -Ddisconf.debug=${DISCONF_DEBUG} -Ddisconf.enable_local_download_dir_in_class_path=${DISCONF_ENABLE_LOCAL_DOWNLOAD_DIR_IN_CLASS_PATH}"
JAVA_OPTS="$JAVA_OPTS -javaagent:/usr/local/tmonitor/boot/tmonitor-agent-bootstrap-1.0.jar -Dtmonitor.appname=tz-position”

-Xms1024m -Xmx1024m -Xmn768m -Xss256k -XX:PermSize=64m -XX:MaxPermSize=256m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/www/temp -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:+CMSClassUnloadingEnabled

-Djava.library.path=/Users/zhangwei/Software/or-tools-6.7.2/lib -Xms1024m -Xmx1024m -Xmn768m -Xss256k -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/www/temp -XX:+DisableExplicitGC -XX:+UseG1GC
```



