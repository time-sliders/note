# 关于配置
conf/server.xml 中配置了默认加载目录，默认是 $TOMCAT_HOME/webapps，一般都会修改成 /data/www/ROOT

    <Host name="localhost" debug="0" appBase="/data/www/ROOT/"
          unpackWARs="true" autoDeploy="false" deployOnStartup="false"
          xmlValidation="false" xmlNamespaceAware="false" >
          <Context path="" docBase="tz-portal" debug="0" reloadable="false"/>
    </Host>
    <Host name="localhost"  appBase="/data/www/ROOT/"
                unpackWARs="true" autoDeploy="true">
    </Host>
