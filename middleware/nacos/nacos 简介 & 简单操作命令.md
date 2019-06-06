https://github.com/alibaba/nacos
https://nacos.io/en-us/docs/quick-start.html

`unzip nacos-server-0.7.0.zip`
`cd nacos/bin`
`sh startup.sh -m standalone`

#### 服务注册

`curl -X PUT 'http://127.0.0.1:8848/nacos/v1/ns/instance?serviceName=nacos.naming.serviceName&ip=20.18.7.10&port=8080'`

#### 服务发现

`curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instances?serviceName=nacos.naming.serviceName'`

#### 配置发布

`curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=helloWorld"`

#### 获取配置

`curl -X GET "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"`

#### 关闭服务

`sh shutdown.sh`
