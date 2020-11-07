# K8S初步实践

## K8S原理介绍
  Kubernetes(K8S)是一个可以帮助我们管理微服务(microservices)的系统，他可以自动化地部署及管理多台机器上的多个容器(容器)。更进一步而言，Kubernetes想解决的问题是：“手动部署多个容器到多台机器上并监测管理这些容器的状态非常麻烦。”而Kubernetes要提供的解法：提供一个平台以高层结构的抽象化去自动化操作与管理容器们。

K8S能做到
* 同时部署多个容器到多台机器上(部署)
* 服务的乘载量有变化时，可以对容器做自动扩展(缩放)
* 管理多个容器的状态，自动检测并重启故障的容器(管理)

## K8S的基本组件

### K8S架构图

![avatar](https://raw.githubusercontent.com/kubernetes/kubernetes/release-1.2/docs/design/architecture.png)

K8S的核心组件
* etcd保存了整个集群的状态；
* apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
* controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
* scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
* kubelet负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
* Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
* kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；

[官网全部组件介绍](https://kubernetes.io/zh/docs/concepts/overview/components/)


### Pod

Kubernetes运作的最小单位，一个Pod对应到一个应用服务（Application），表示一个Pod可能会对应到一个API Server。
* 每个Pod都有一个身分证，也就是属于这个Pod的yaml档
* 一个Pod里面可以有一个或者多个Container，但一般情况一个Pod最好只有一个Container
* 同一个Pod中的Containers共享相同资源及网路，彼此透过local port number沟通
* Pod，是 Kubernetes 项目的原子调度单位。

[官方介绍](https://kubernetes.io/docs/concepts/workloads/pods/)

### Node

K8s集群中的计算能力由Node提供，最初Node称为服务节点Minion，后来改名为Node。K8s集群中的Node也就等同于Mesos集群中的Slave节点，是所有Pod运行所在的工作主机，可以是物理机也可以是虚拟机。不论是物理机还是虚拟机，工作主机的统一特征是上面要运行kubelet管理节点上运行的容器。


## K8S 集群的简单部署准备

* 系统：mac os
* docker: 19.03.13
* docker image
    * 实现了输出hello world
    * 实现接口/panic 非正常终止
    * 实现/version 查看当前api 版本

### kubectl 安装
mac 通过brew安装
```
brew install kubectl 
```
or
```
brew install kubernetes-cli
```

验证是否安装成功
```
kubectl version --client
```

[其他的安装方式](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

### minikube 安装
这里我们安装阿里云的版本
```
curl -Lo minikube https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v1.13.0/minikube-darwin-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```
验证
```
●✚  minikube version
minikube version: v1.13.0
commit: 23aa1eb200a03ae5883dd9d453d4daf3e0f59668
```

## K8S 简单部署（借助minikube）

### 启动一个单节点的minikube

```
minikube start --cpus=2 --memory=2048mb --registry-mirror=https://t65rjofu.mirror.aliyuncs.com --driver=virtualbox
```
这里指定了cpu和内存资源

```
●✚  minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
●✚  kubectl get no
NAME       STATUS   ROLES    AGE    VERSION
minikube   Ready    master   106s   v1.19.0
```
pod的yaml文件
```
apiVersion: v1
kind: Pod
metadata:
  name: docker-demo
spec:
  containers:
    - name: docker-demo
      image: sundacheng/docker_demo:latest
```
执行部署
```
●✚  kubectl apply -f .
pod/docker-demo created
●✚  kubectl get all
NAME              READY   STATUS              RESTARTS   AGE
pod/docker-demo   0/1     ContainerCreating   0          17s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3m46s
```
现在我们没有将服务暴露出来
所以我们要执行
```
●✚  kubectl port-forward docker-demo 3000:3000
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
●✚  curl 127.0.0.1:3000
hello docker%
```

### 部署一个service
刚刚我们部署的pod，通过pord-forward暴露对应端口到宿主机，这种只能是用来测试，我们可以通过k8s的反向代理来实现一个service.
在这之前简单介绍下Label和Label selector

#### todo 灰度发布

#### Label

Label是Kubernetes系列中另外一个核心概念。是一组绑定到K8s资源对象上的key/value对。同一个对象的labels属性的key必须唯一。label可以附加到各种资源对象上，如Node,Pod,Service,RC等
通过给指定的资源对象捆绑一个或多个不用的label来实现多维度的资源分组管理功能，以便于灵活，方便地进行资源分配，调度，配置，部署等管理工作。

#### Label selector(标签选择器)

Label selector是Kubernetes核心的分组机制，通过label selector客户端/用户能够识别一组有共同特征或属性的资源对象。

#### Label selector的使用场景

1.kube-controller进程通过资源对象RC上定义的Label Selector来筛选要监控的Pod副本的数量，从而实现Pod副本的数量始终符合预期设定的全自动控制流程

2.kupe-proxy进程通过Service的Label Selector来选择对应的Pod，自动建立器每个Service到对应Pod的请求转发路由表，从而实现Service的智能负载均衡机制

3.通过对某些Node定义特定的Label,并且在Pod定义文件中使用NodeSelector这种标签调度策略，Kube-scheduler进程可以实现Pod定向调度的特性

[Laber 和Label selector 官方文档](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)

#### 发布服务的类型
* ClusterIP：通过集群的内部 IP 暴露服务，选择该值，服务只能够在集群内部可以访问，这也是默认的 ServiceType。

* NodePort：通过每个 Node 上的 IP 和静态端口（NodePort）暴露服务。 NodePort 服务会路由到 ClusterIP 服务，这个 ClusterIP 服务会自动创建。 通过请求 <NodeIP>:<NodePort>，可以从集群的外部访问一个 NodePort 服务。

* LoadBalancer：使用云提供商的负载局衡器，可以向外部暴露服务。 外部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务。

* ExternalName：通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容（例如， foo.bar.example.com）。 没有任何类型代理被创建。

#### 部署svc
pod的yml
```
apiVersion: v1
kind: Pod
metadata:
  name: docker-demo
  labels:
    app: docker-demo #给 pod加上标签
spec:
  containers:
    - name: docker-demo
      image: sundacheng/docker_demo:v1.0.1
      env:
      - name: MY_NODE_NAME
        valueFrom:
          fieldRef:
            fieldPath: spec.nodeName
      - name: MY_POD_NAME
        valueFrom:
          fieldRef:
           fieldPath: metadata.name
      - name: MY_POD_NAMESPACE
        valueFrom:
          fieldRef:
            fieldPath: metadata.namespace
      - name: MY_POD_IP
        valueFrom:
          fieldRef:
            fieldPath: status.podIP

```
svc的yml
```
apiVersion: v1
kind: Service
metadata:
  name: docker-demo
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
      nodePort: 31080 # 指定发布服务类型是nodePort 并且指定对外(宿主机)端口号
  selector:
    app: docker-demo #Label selector
  type: NodePort

```
执行部署和验证
```
●✚  kubectl apply -f .
pod/docker-demo created
service/docker-demo created

●✚  kubectl get all
NAME              READY   STATUS              RESTARTS   AGE
pod/docker-demo   0/1     ContainerCreating   0          43s

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/docker-demo   NodePort    10.110.72.67   <none>        3000:31080/TCP   43s
service/kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP          2m52s

●✚  minikube service docker-demo --url
http://192.168.99.129:31080
●✚  curl http://192.168.99.129:31080
Hello World!%
```

#### 可以通过svc进行蓝绿部署

well,我们现在有一个版本号是1.0.1的pod，如果我们要升级或者降级，可以通过svc实现蓝绿部署
首先pod的image需要更改到要部署的image,并加上新的lable代表版本号
v1.0.0.yml
```
apiVersion: v1
kind: Pod
metadata:
  name: docker-demo-v1.0.0
  labels:
    app: docker-demo
    version: v1.0.0 
spec:
  containers:
    - name: docker-demo
      image: sundacheng/docker_demo:v1.0.0
```
v1.0.1把对应参数改了就行
svc的yml
```
apiVersion: v1
kind: Service
metadata:
  name: docker-demo
spec:
  ports:
    - name: http
      port: 3000
      targetPort: 3000
      nodePort: 31080
  selector:
    app: docker-demo
    version: v1.0.1
  type: NodePort
```
执行验证
```
●✚  ls
demo-pod-v1.0.0.yml demo-pod-v1.0.1.yml demo-service.yml

●✚  kubectl apply -f .
pod/docker-demo-v1.0.0 created
pod/docker-demo-v1.0.1 created
service/docker-demo created

●✚  kubectl get all
NAME                     READY   STATUS    RESTARTS   AGE
pod/docker-demo-v1.0.0   1/1     Running   0          5m19s
pod/docker-demo-v1.0.1   1/1     Running   0          5m19s

NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/docker-demo   NodePort    10.96.92.104   <none>        3000:31080/TCP   5m19s
service/kubernetes    ClusterIP   10.96.0.1      <none>        443/TCP          43m

●✚  minikube service docker-demo --url
http://192.168.99.129:31080

●✚  curl http://192.168.99.129:31080/version
1.0.1%

# 修改svc的selector 指向1.0.0
●✚  kubectl apply -f .
pod/docker-demo-v1.0.0 unchanged
pod/docker-demo-v1.0.1 unchanged
service/docker-demo configured
●✚  curl http://192.168.99.129:31080/version
1.0.0%
```


### svc的rs集群部署

上面讲了单节点单个pod的部署，现在讲下多节点。
多个node节点需要虚拟机，通过token加入，minikube帮我们做了这点，只需要在最开始minikube start后面加上参数`--node=3`
```
minikube start --cpus=2 --memory=2048mb --registry-mirror=https://t65rjofu.mirror.aliyuncs.com --driver=virtualbox --nodes=3
```
同时pod的yml文件也要做修改
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: docker-demo
spec:
  replicas: 3 #指定rs数量
  selector:
    matchLabels:
      app: docker-demo
  template: #定义每个pod的类容
    metadata:
      labels:
        app: docker-demo
    spec:
      containers:
        - name: docker-demo
          image: sundacheng/docker_demo:v1.0.1
          env:
            - name: MY_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: MY_POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP

```
执行部署验证
```
●✚  kubectl get no
NAME           STATUS   ROLES    AGE     VERSION
minikube       Ready    master   8m28s   v1.19.0
minikube-m02   Ready    <none>   7m33s   v1.19.0
minikube-m03   Ready    <none>   6m39s   v1.19.0
#查看node节点

●✚  kubectl get all
NAME                    READY   STATUS    RESTARTS   AGE
pod/docker-demo-5mcx4   1/1     Running   0          6m32s
pod/docker-demo-qgt9r   1/1     Running   0          6m32s
pod/docker-demo-rgdzq   1/1     Running   0          6m32s

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/docker-demo   NodePort    10.103.56.181   <none>        3000:31080/TCP   6m32s
service/kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP          15m

NAME                          DESIRED   CURRENT   READY   AGE
replicaset.apps/docker-demo   3         3         3       6m32s
#部署成功 

●✚  kubectl get pod -o wide
NAME                READY   STATUS    RESTARTS   AGE     IP           NODE           NOMINATED NODE   READINESS GATES
docker-demo-5mcx4   1/1     Running   0          7m10s   172.17.0.2   minikube-m03   <none>           <none>
docker-demo-qgt9r   1/1     Running   0          7m10s   172.17.0.2   minikube-m02   <none>           <none>
docker-demo-rgdzq   1/1     Running   0          7m10s   172.17.0.3   minikube       <none>           <none>
#查看pod和node的绑定关系
```

### deployment 部署
K8S提供了一种更加简单的更新RC和Pod的机制，叫做Deployment。通过在Deployment中描述你所期望的集群状态，Deployment Controller会将现在的集群状态在一个可控的速度下逐步更新成你所期望的集群状态。Deployment主要职责同样是为了保证pod的数量和健康，90%的功能与Replication Controller完全一样，可以看做新一代的Replication Controller。

功能：
* 事件和状态查看：可以查看Deployment的升级详细进度和状态。
* 回滚：当升级pod镜像或者相关参数的时候发现问题，可以使用回滚操作回滚到上一个稳定的版本或者指定的版本。
* 版本记录: 每一次对Deployment的操作，都能保存下来，给予后续可能的回滚使用。
* 暂停和启动：对于每一次升级，都能够随时暂停和启动。
* 多种升级方案：Recreate----删除所有已存在的pod,重新创建新的; RollingUpdate----滚动升级，逐步替换的策略，同时滚动升级时，支持更多的附加参数，例如设置最大不可用pod数量，最小升级间隔时间等等。

```
#创建命令：kubectl create -f deployment.yaml --record
#使用rollout history命令，查看Deployment的历史信息：kubectl rollout history deployment docker-demo
#使用rollout undo回滚到上一版本: kubectl rollout undo deployment docker-demo
#使用–to-revision可以回滚到指定版本:  kubectl rollout undo deployment docker-deployment --to-revision=2
```

### config map

ConfigMap 允许你将配置文件与镜像文件分离，以使容器化的应用程序具有可移植性。 本页提供了一系列使用示例，这些示例演示了如何创建 ConfigMap 以及配置 Pod 使用存储在 ConfigMap 中的数据。

config map yml
```
```
对应pod yml修改
```
```
更新config map,查看deployment

重启deployment
```
kubectl replace --force -f demo-deployment.yml
```


### 水平扩展HPA

压测
```
kubectl run  -i --rm  --tty load-generator --image=busybox /bin/sh\n

while true; do wget -q -O- http://php-hpa-service; done
```


### todo ingress