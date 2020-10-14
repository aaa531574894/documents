Kubernetes使用指南

k8s 中文社区官方教程 [链接](https://www.kubernetes.org.cn/kubernetes%e8%ae%be%e8%ae%a1%e6%9e%b6%e6%9e%84)

## 0.k8s架构图

![architecture](./images/architecture.png)







## 1.名词释义

### Cluster

即K8s集群，包括master 和 node。

Cluster由多个node节点组成集群，master是一个特殊的node，一般情况下，应用pod不会被调度到master节点上。



### Master

是cluster的大脑，它的主要职责是调度，决定应用在哪个worker上运行；为了实现高可用，可以运行多个master组成集群。master节点上主要运行下列服务：

* api server  提供restful api
* scheduler  负责决定将Pod放在哪个node上运行。
* Controller 负责管理cluster的各种资源。
* etcd           分布式可靠存储服务



### Node

node是工作节点，是k8s中参与计算的机器，由master节点管理。master节点中可以有多个pod，k8s master节点会自动在node集群中调度pod，主节点的自动调度需要考虑worker节点上的资源使用率。每个node节点上都至少运行着两部分内容：

1. kubelet，负责k8s主节点和工作节点之间的通信过程；管理pod和机器上运行的容器。
2. docker，负责从仓库中提取容器镜像，运行容器。
3. pod网络   如flannel





### 资源

namespace



### Pod

Pod是K8s的最小工作单元。每个Pod包含一个或多个容器。Pod中的容器会作为一个整体被Master调度到一个Node上运行。

K8s引入Pod主要基于下面两个目的：

1. 可管理性

   有些容器之间天生就是紧密联系，一起工作的。Pod提供了比容器更高层级的抽象，将它们封装到一个部署单元中。K8s以Pod为最小单元进行调度、扩展、共享资源。

2. 通信和资源共享

   Pod中的所有容器都使用同一个网格namespace，即相同的ip地址和Port空间。它们可以直接通过localhost来进行通信。同样的，存储空间也是，**当k8s挂在volume到Pod时，本质上是将volume挂在到Pod中的每一个容器。**

Pods有两种使用方式：

1. 运行单一容器。

   即简单的将单个容器简单封装为Pod。在这种情况下，k8s管理的仍是Pod而不是容器。

2. 运行多个容器。



### Controller

是k8s中用来管理pod的组件，是个抽象，有多种实现。k8s 通常不会直接创建Pod，而是通过Controller来管理Pod。Controller中定义了Pod的部署特性，比如有几个副本、在什么样的Node上运行。k8s提供了多种Controller，包括：

* Deployment

  最常用的Controller，可以管理pod的多个副本，并确保Pod按照期望的状态运行。

* ReplicaSet

  实现了Pod的多副本管理。使用deployment时会自动创建ReplicaSet，即deployment就是通过replicaSet来管理多个副本的。**一般我们不会直接使用此controller。**

* DaemonSet

  deployment可能会在一个node上运行多个pod副本；而daemonSet会**为每个Node均启动一个pod副本**。

  * 在每个node节点上运行日志收集daemon
  * 在每个node节点上运行监控daemon

  其实k8s 自身就在用daemontSet来运行一些系统组件，包括kube-proxy,kube-flanneld等。

  可以通过`kubectl get daemonset -n kube-system -o wide` 来查看。

* StatefulSet

  

* Job

  用于运行结束就删除的应用，其他Controller的Pod通常是长期运行的。

* 。。。。。。

### Service

一个Deployment可以部署多个副本，每个Pod都有自己的IP，外界如何访问这些副本呢？通过Pod的ip吗？no，答案是service。

k8s定义了外界访问一组特定Pod的方式。Service有自己的IP和端口，Service为Pod提供了负载均衡。



K8s运行容器与访问容器分别由Controller与Service这两个组件来负责执行。

 



### namespace

多个用户或多个项目组同时使用同一个k8s cluster集群时，通过namespace将他们创建的资源（Controller，pod）进行区分。

namespace可以将一个物理的cluster从逻辑上划分为多个虚拟的cluster，每个cluster就是一个namespace。不通namespace下的资源是完全隔离的。

k8s会默认创建两个namespace

1. default         创建资源时不指定namespace，则都会放到此namespace中。
2. kube-system  k8s创建的系统资源都放在这个namespace中。

> ps：可以通过 `kubectl get namespace` 查看已有的namespace资源列表。





### label

默认情况下，scheduler会将Pod调度至所有可用的Node。不过有些情况我们希望将Pod部署到指定的Node上。就可以通过对node添加label来实现。



```shell
#为node添加lable   key=value
kubectl lable node ${nodeName}  ${key}=${value}
#查看节点的lable 在kubectl get node 后加入  --show-lables 参数
kubectl get node   --show-lables  
#为node删除lable   即key后带`-`号即可
kubectl lable node ${nodeName} ${key}-

```





## 2.命令行

```bash
# 列出资源  nodes  pods  services 等
kubectl get  ${资源类型}
# 显示资源相关的详细信息
kubectl describe    deployment/pod
# 打印pod和其中容器的日志
kubectl logs
# 在pod中的容器上执行命令
kubectl exec 

# 启动一个deployment 使用指定镜像 数量为2
kubectl run httpd-app --image=httpd --replicas=2


# kubectl create delete apply edit
# 查看已存在的某个资源的配置文件
kubectl edit daemonset kube-proxy --namespace=kube-system






```







## 3.创建资源

k8s中有两种创建资源的方式：

* 通过命令行
* 通过yaml资源配置文件

### 1.基于命令行

基于命令行使用时比较简单、直观、容易上手，适合临时测试。

eg：

```
kubectl run httpd-app --image=httpd --replicas
```

详细使用可以使用  `kubectl run --help` 命令查看



### 2.基于资源文件

首先要准备一份yaml配置文件，然后通过`kubectl apply -f ${fileName}` 来完成资源创建的操作。

除了kubectl apply之外，还有 create、replace、edit、patch等等，一般用apply足矣。



列出一个标准的config文件，并对其进行说明：

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadate:
  name: nginx-deployment
spec:
  replicas: 6
  template: 
    metedata:
      labels:
        app: web_server
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
      nodeSelector:
        disktype: ssd
```





























