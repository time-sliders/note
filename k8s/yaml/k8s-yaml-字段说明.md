# 使用YAML创建Pod
## 创建Pod

    ---
    apiVersion: v1
    kind: Pod
    metadata:
      name: kube100-site
      labels:
        app: web
    spec:
      containers:
        - name: front-end
          image: nginx
          ports:
            - containerPort: 80
        - name: flaskapp-demo
          image: jcdemo/flaskapp
          ports:
            - containerPort: 5000

* **apiVersion**：此处值是v1，这个版本号需要根据安装的Kubernetes版本和资源类型进行变化，**记住不是写死的**。
* **kind**：此处创建的是Pod，根据实际情况，此处资源类型可以是Deployment、Job、Ingress、Service等。
* **metadata**：包含Pod的一些meta信息，name、namespace、labels等信息。
* **spe**：容器规格，包括一些container，storage，volume以及其他Kubernetes需要的参数，以及诸如是否在容器失败时重新启动容器的属性。

#### kubectl 根据pod.yaml 创建pod
`kubectl create -f pod.yaml`

