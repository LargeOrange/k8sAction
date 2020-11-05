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