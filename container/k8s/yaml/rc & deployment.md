#ReplicationController(RC)
RC保证在同一时间能够运行指定数量的Pod副本，保证Pod总是可用。如果实际Pod数量比指定的多就结束掉多余的，如果实际数量比指定的少就启动缺少的。当Pod失败、被删除或被终结时，RC会自动创建新的Pod来保证副本数量，所以即使只有一个Pod，也应该使用RC来进行管理

    apiVersion: v1
    kind: ReplicationController
    metadata:
      name: frontend
      labels:
        name: frontend
    spec:
      replicas: 3
      selector:
        name: frontend
      template:
        metadata:
         labels:
           name: frontend
        spec:
         containers:
         - name: frontend
           image: kubeguide/guestbook-php-frontend:latest
           env :
           - name : GET_HOSTS_FROM
             value : env
           ports:
           - containerPort: 80

### RC 常用指令
* 创建RC  

`kubectl create -f rc.yaml`

* 查看RC具体信息  

`kubectl describe rc frontend`

* 通过RC修改Pod副本数量（需要修改yaml文件的spec.replicas字段到目标值，然后替换旧的yaml文件）  

`kubectl replace -f rc.yaml`  

`kubect edit replicationcontroller frontend`

* 对RC使用滚动升级，来发布新功能或修复BUG  

`kubectl rolling-update frontend --image=kubeguide/guestbook-php-frontend:latest`

* 当Pod中只有一个容器时，通过–image参数指定新的Tag完成滚动升级，但如果有多个容器或其他字段修改时，需要指定yaml文件  

`kubectl rolling-update frontend -f FILE.yaml`

* 如果在升级过程中出现问题（如发现配置错误、长时间无响应），可以使用CTRL+C退出，再进行回滚   

`kubectl rolling-update frontend --image=kubeguide/guestbook-php-frontend:latest --rollback`




# Deployment 
1. RC只支持基于等式的selector（env=dev或environment!=qa），但**ReplicaSet还支持新的，基于集合的selector**（version in (v1.0, v2.0)或env notin (dev, qa)），这对复杂的运维管理很方便。

1. 使用Deployment升级Pod，只需要定义Pod的最终状态，k8s会为你执行必要的操作，虽然能够使用命令# kubectl rolling-update完成升级，但它是在客户端与服务端多次交互控制RC完成的，所以REST API中并没有rolling-update的接口，这为定制自己的管理系统带来了一些麻烦。

1. Deployment拥有更加灵活强大的升级、回滚功能

目前，Replica Set与RC的区别只是支持的selector不同，后续肯定会加入更多功能。Deployment使用了Replica Set，它是更高一层的概念。除非用户需要自定义升级功能或根本不需要升级Pod，在一般情况下，我们**推荐使用Deployment**而不直接使用Replica Set

### deployment 常用指令
* **使用子命令create，创建Deployment**  

`kubectl create -f deployment.yaml --record`  

–record参数，使用此参数将记录后续创建对象的操作，方便管理与问题追溯

* **使用子命令edit，编辑spec.replicas/spec.template.spec.container.image字段，完成deployment的扩缩容与滚动升级，这要比子命令rolling-update速度快很多**  

`kubectl edit deployment hello-deployment`

* 使用rollout history命令，查看Deployment的历史信息  

`kubectl rollout history deployment hello-deployment`

* 上面提到RC在rolling-update升级成功后不能直接回滚，而使用Deployment却可以回滚到上一版本，但要加上–revision参数，指定版本号
  
`kubectl rollout history deployment hello-deployment --revision=2`

* 使用rollout undo回滚到上一版本  

`kubectl rollout undo deployment hello-deployment`

* 使用–to-revision可以回滚到指定版本  

`kubectl rollout undo deployment hello-deployment --to-revision=2`


