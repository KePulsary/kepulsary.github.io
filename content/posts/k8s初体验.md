---
title: "k8s初体验"
date: 2023-02-26 12:46:22
url: /2023/02/26/k8s初体验/
updated: 2023-02-27 15:13:43
tags:
  - k8s
  - 笔记
description: "k8s入门"
---

# k8s初体验

## 什么是k8s

Kubernetes 是一个开源的容器编排引擎，用来对容器化应用进行自动化部署、 扩缩和管理。

## 为什么需要k8s

通过现代的 Web 服务，用户希望应用程序能够 24/7 全天候使用，开发人员希望每天可以多次发布部署新版本的应用程序。 容器化可以帮助软件包达成这些目标，使应用程序能够以简单快速的方式发布和更新，而无需停机。Kubernetes 帮助你确保这些容器化的应用程序在你想要的时间和地点运行，并帮助应用程序找到它们需要的资源和工具。Kubernetes 是一个可用于生产的开源平台，根据 Google 容器集群方面积累的经验，以及来自社区的最佳实践而设计。

## k8s的使用

### 使用 Minikube 创建集群

minikube start 时因为国内墙问题会有start失败的情况，自行百度

一个 Kubernetes 集群包含两种类型的资源:

- **Master** 调度整个集群。 Master 协调集群中的所有活动，例如调度应用、维护应用的所需状态、应用扩容以及推出新的更新。
- **Nodes** 负责运行应用。Node 是一个虚拟机或者物理机，它在 Kubernetes 集群中充当工作机器的角色

![img](https://cdn.jsdelivr.net/gh/L1aovo/blogpic@main/img/module_01_cluster.svg)

1. **minikube version** - 确定minikube已经安装 [https://minikube.sigs.k8s.io/docs/start/](https://minikube.sigs.k8s.io/docs/start/)
2. **minikube start** - 启动minikube
3. **kubectl version** - 确定kubectl已安装
4. **kubectl cluster-info** - 查看集群详细信息
5. **kubectl get nodes** - 查看集群中的节点

### 使用 kubectl 创建 Deployment

一旦运行了 Kubernetes 集群，就可以在其上部署容器化应用程序。 为此，你需要创建 Kubernetes **Deployment** 配置。Deployment 指挥 Kubernetes 如何创建和更新应用程序的实例。创建 Deployment 后，Kubernetes master 将应用程序实例调度到集群中的各个节点上。

![img](https://cdn.jsdelivr.net/gh/L1aovo/blogpic@main/img/module_02_first_app.svg)

1. **kubectl version** - 确定kubectl已安装
2. **kubectl get nodes** - 查看集群中的节点
3. kubectl create deployment deployment-name –image=deployment-image
4. **kubectl get deployments** - 列出所有deployment
5. **kubectl proxy** - 代理api出来使得能够被访问

### 查看 pod 和工作节点

在创建 Deployment 时, Kubernetes 添加了一个 **Pod** 来托管你的应用实例。Pod 是 Kubernetes 抽象出来的，表示一组一个或多个应用程序容器（如 Docker），以及这些容器的一些共享资源。这些资源包括:

- 共享存储，当作卷
- 网络，作为唯一的集群 IP 地址
- 有关每个容器如何运行的信息，例如容器镜像版本或要使用的特定端口。

![img](https://cdn.jsdelivr.net/gh/L1aovo/blogpic@main/img/module_03_pods.svg)

一个 pod 总是运行在 **工作节点**。工作节点是 Kubernetes 中的参与计算的机器，可以是虚拟机或物理计算机，具体取决于集群。每个工作节点由主节点管理。工作节点可以有多个 pod ，Kubernetes 主节点会自动处理在集群中的工作节点上调度 pod 。 主节点的自动调度考量了每个工作节点上的可用资源。

每个 Kubernetes 工作节点至少运行:

- Kubelet，负责 Kubernetes 主节点和工作节点之间通信的过程; 它管理 Pod 和机器上运行的容器。
- 容器运行时（如 Docker）负责从仓库中提取容器镜像，解压缩容器以及运行应用程序。

![img](https://cdn.jsdelivr.net/gh/L1aovo/blogpic@main/img/module_03_nodes.svg)

使用 kubectl 进行故障排除。 最常见的操作可以使用以下 kubectl 命令来获取有关已部署的应用程序及其环境的信息：

- **kubectl get** - 列出资源
- **kubectl describe** - 显示有关资源的详细信息
- **kubectl logs** - 打印 pod 和其中容器的日志
- **kubectl exec** - 在 pod 中的容器上执行命令

详细的命令示例

```bash
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-fb5c67579-4whtb   1/1     Running   0          2m46s

$ kubectl describe pods
Name:         kubernetes-bootcamp-fb5c67579-4whtb
Namespace:    default
Priority:     0
Node:         minikube/10.0.0.30
Start Time:   Sun, 26 Feb 2023 10:22:42 +0000
Labels:       app=kubernetes-bootcamp
              pod-template-hash=fb5c67579
Annotations:  <none>
Status:       Running
.......................

$ export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
$ echo Name of the Pod: $POD_NAME
Name of the Pod: kubernetes-bootcamp-fb5c67579-4whtb    @ 实际是就是pod的name

$ curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-fb5c67579-4whtb | v=1
$ kubectl logs $POD_NAME
Kubernetes Bootcamp App Started At: 2023-02-26T10:22:44.413Z | Running On:  kubernetes-bootcamp-fb5c67579-4whtb 

Running On: kubernetes-bootcamp-fb5c67579-4whtb | Total Requests: 1 | App Uptime: 2260.095 seconds | Log Time: 2023-02-26T11:00:24.509Z

$ kubectl exec $POD_NAME -- env # 类似docker exec  容器 命令 
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubernetes-bootcamp-fb5c67579-4whtb
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBERNETES_SERVICE_HOST=10.96.0.1
NPM_CONFIG_LOGLEVEL=info
NODE_VERSION=6.3.1
HOME=/root

$ kubectl exec -ti $POD_NAME -- bash # 以交互式执行命令执行的bash即进入命令行...
```

### 使用 Service 暴露你的应用

Kubernetes [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 是转瞬即逝的。 Pod 实际上拥有 [生命周期](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/)。 当一个工作 Node 挂掉后, 在 Node 上运行的 Pod 也会消亡。 [ReplicaSet](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/replicaset/) 会自动地通过创建新的 Pod 驱动集群回到目标状态，以保证应用程序正常运行。 换一个例子，考虑一个具有3个副本数的用作图像处理的后端程序。这些副本是可替换的; 前端系统不应该关心后端副本，即使 Pod 丢失或重新创建。也就是说，Kubernetes 集群中的每个 Pod (即使是在同一个 Node 上的 Pod )都有一个唯一的 IP 地址，因此需要一种方法自动协调 Pod 之间的变更，以便应用程序保持运行。

Kubernetes 中的服务(Service)是一种抽象概念，它定义了 Pod 的逻辑集和访问 Pod 的协议。Service 使从属 Pod 之间的松耦合成为可能。 和其他 Kubernetes 对象一样, Service 用 YAML [(更推荐)](https://kubernetes.io/zh-cn/docs/concepts/configuration/overview/#general-configuration-tips) 或者 JSON 来定义. Service 下的一组 Pod 通常由 *LabelSelector* (请参阅下面的说明为什么你可能想要一个 spec 中不包含`selector`的服务)来标记。

尽管每个 Pod 都有一个唯一的 IP 地址，但是如果没有 Service ，这些 IP 不会暴露在集群外部。Service 允许你的应用程序接收流量。Service 也可以用在 ServiceSpec 标记`type`的方式暴露

- *ClusterIP* (默认) - 在集群的内部 IP 上公开 Service 。这种类型使得 Service 只能从集群内访问。
- *NodePort* - 使用 NAT 在集群中每个选定 Node 的相同端口上公开 Service 。使用`<NodeIP>:<NodePort>` 从集群外部访问 Service。是 ClusterIP 的超集。
- *LoadBalancer* - 在当前云中创建一个外部负载均衡器(如果支持的话)，并为 Service 分配一个固定的外部IP。是 NodePort 的超集。
- *ExternalName* - 通过返回带有该名称的 CNAME 记录，使用任意名称(由 spec 中的`externalName`指定)公开 Service。不使用代理。这种类型需要`kube-dns`的v1.7或更高版本。

另外，需要注意的是有一些 Service 的用例没有在 spec 中定义`selector`。 一个没有`selector`创建的 Service 也不会创建相应的端点对象。这允许用户手动将服务映射到特定的端点。没有 selector 的另一种可能是你严格使用`type: ExternalName`来标记。

**Service 和 Label**

![img](https://cdn.jsdelivr.net/gh/L1aovo/blogpic@main/img/module_04_services.svg)

Service 通过一组 Pod 路由通信。Service 是一种抽象，它允许 Pod 死亡并在 Kubernetes 中复制，而不会影响应用程序。在依赖的 Pod (如应用程序中的前端和后端组件)之间进行发现和路由是由Kubernetes Service 处理的。

Service 匹配一组 Pod 是使用 [标签(Label)和选择器(Selector)](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels), 它们是允许对 Kubernetes 中的对象进行逻辑操作的一种分组原语。标签(Label)是附加在对象上的键/值对，可以以多种方式使用:

- 指定用于开发，测试和生产的对象
- 嵌入版本标签
- 使用 Label 将对象进行分类

![img](https://cdn.jsdelivr.net/gh/L1aovo/blogpic@main/img/module_04_labels.svg)

1. **kubectl get pods** - 列出存在的pods
2. **kubectl get services/svc** - 列出存在的service
3. **kubectl expose** (-f FILENAME | TYPE NAME) [–port=port]
[–protocol=TCP|UDP|SCTP] [–target-port=number-or-name] [–name=name]
[–external-ip=external-ip-of-service] [–type=type] [options] - 创建新服务并将其暴露给外部流量
4. **kubectl describe services/服务名** - 查看服务的详情
5. curl $(minikube ip):$NODE_PORT 请求 这里的ip是哪个ip呢，node上面？应该是service上面
6. **kubectl describe deployment** - deployment的详细内容
7. 通过 -l app=xxxx 来筛选指定标签的内容 pods or svc 等等
8. **kubectl label** pods $POD_NAME version=v1 - 要应用新标签，我们使用 label 命令，后跟对象类型、对象名称和新标签 我们便可以在对象详情的labels部分看见 version=v1

### 运行应用程序的多个实例

Deployment 仅为跑这个应用程序创建了一个 Pod。 当流量增加时，我们需要扩容应用程序满足用户需求。

**扩缩** 是通过改变 Deployment 中的副本数量来实现的。

扩展 Deployment 将创建新的 Pods，并将资源调度请求分配到有可用资源的节点上，收缩 会将 Pods 数量减少至所需的状态。Kubernetes 还支持 Pods 的[自动缩放](https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/)，。将 Pods 数量收缩到0也是可以的，但这会终止 Deployment 上所有已经部署的 Pods。

运行应用程序的多个实例需要在它们之间分配流量。服务 (Service)有一种负载均衡器类型，可以将网络流量均衡分配到外部可访问的 Pods 上。服务将会一直通过端点来监视 Pods 的运行，保证流量只分配到可用的 Pods 上。

![img](https://cdn.jsdelivr.net/gh/L1aovo/blogpic@main/img/module_05_scaling1.svg)

1. **kubectl get deployments** - 列出 deployments
2. **kubectl get rs**查看 Deployment 创建的 ReplicaSet ReplicaSet的名字为 [DEPLOYMENT-NAME]-[RANDOM-STRING] 随机字符串使用pod-template-hash作为随机数种子
3. **kubectl scale** deployment type –replicas=4 使用 kubectl scale 命令扩展副本，然后是部署类型、名称和所需的实例数 通过设置实例数来增加/减少pod数量

```bash
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1/1     1            1           24s
$ kubectl get rs
NAME                            DESIRED   CURRENT   READY   AGE
kubernetes-bootcamp-fb5c67579   1         1         1       110s
$ kubectl scale deployments/kubernetes-bootcamp --replicas=4
deployment.apps/kubernetes-bootcamp scaled
$ kubectl scale deployments/kubernetes-bootcamp --replicas=4
deployment.apps/kubernetes-bootcamp scaled
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   4/4     4            4           10m
$ kubectl get pods -o wide
NAME                                  READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
kubernetes-bootcamp-fb5c67579-bfp5g   1/1     Running   0          89s   172.18.0.9   minikube   <none>           <none>
kubernetes-bootcamp-fb5c67579-c7h8c   1/1     Running   0          89s   172.18.0.7   minikube   <none>           <none>
kubernetes-bootcamp-fb5c67579-dmksw   1/1     Running   0          89s   172.18.0.8   minikube   <none>           <none>
kubernetes-bootcamp-fb5c67579-v5zx5   1/1     Running   0          10m   172.18.0.3   minikube   <none>           <none>
$ kubectl describe deployments/kubernetes-bootcamp
Name:                   kubernetes-bootcamp
Namespace:              default
CreationTimestamp:      Sun, 26 Feb 2023 12:00:22 +0000
Labels:                 app=kubernetes-bootcamp
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=kubernetes-bootcamp
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=kubernetes-bootcamp
  Containers:
   kubernetes-bootcamp:
    Image:        gcr.io/google-samples/kubernetes-bootcamp:v1
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   kubernetes-bootcamp-fb5c67579 (4/4 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  10m   deployment-controller  Scaled up replica set kubernetes-bootcamp-fb5c67579 to 1
  Normal  ScalingReplicaSet  101s  deployment-controller  Scaled up replica set kubernetes-bootcamp-fb5c67579 to 4
```

### 执行滚动更新

用户希望应用程序始终可用，而开发人员则需要每天多次部署它们的新版本。在 Kubernetes 中，这些是通过滚动更新（Rolling Updates）完成的。 **滚动更新** 允许通过使用新的实例逐步更新 Pod 实例，零停机进行 Deployment 更新。新的 Pod 将在具有可用资源的节点上进行调度。在 Kubernetes 中，更新是经过版本控制的，任何 Deployment 更新都可以恢复到以前的（稳定）版本。

![img](https://cdn.jsdelivr.net/gh/L1aovo/blogpic@main/img/module_06_rollingupdates1.svg)

滚动更新允许以下操作：

- 将应用程序从一个环境提升到另一个环境（通过容器镜像更新）
- 回滚到以前的版本
- 持续集成和持续交付应用程序，无需停机

1. **kubectl get** deployments - 获取deployment
2. kubectl get pods - 获取pods
3. **kubectl set image** deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2 设置deployments/kubernetes-bootcamp的镜像 —> 更新镜像 会自己升级
4. kubectl describe pods - 查看应用的版本
5. kubectl rollout status deployments/kubernetes-bootcamp - 还可以通过运行 rollout status 命令来确认更新
6. kubectl rollout undo deployments/kubernetes-bootcamp - 要将部署回滚到上一个工作版本，请使用 rollout undo 命令：

```bash
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   4/4     4            4           36s
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-fb5c67579-8k8n2   1/1     Running   0          26s
kubernetes-bootcamp-fb5c67579-8mpqr   1/1     Running   0          26s
kubernetes-bootcamp-fb5c67579-fw2wj   1/1     Running   0          26s
kubernetes-bootcamp-fb5c67579-hnk24   1/1     Running   0          26s
$ kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
deployment.apps/kubernetes-bootcamp image updated
$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
kubernetes-bootcamp-7d44784b7c-9pb8f   0/1     ContainerCreating   0          1s
kubernetes-bootcamp-7d44784b7c-fgxlb   1/1     Running             0          4s
kubernetes-bootcamp-7d44784b7c-lfskb   1/1     Running             0          4s
kubernetes-bootcamp-7d44784b7c-v95wl   0/1     ContainerCreating   0          1s
kubernetes-bootcamp-fb5c67579-8k8n2    1/1     Terminating         0          42s
kubernetes-bootcamp-fb5c67579-8mpqr    1/1     Terminating         0          42s
kubernetes-bootcamp-fb5c67579-fw2wj    1/1     Terminating         0          42s
kubernetes-bootcamp-fb5c67579-hnk24    1/1     Running             0          42s
```

## k8s配置

在 Kubernetes 中，为 docker 容器设置环境变量有几种不同的方式，比如： Dockerfile、kubernetes.yml、Kubernetes ConfigMaps、和 Kubernetes Secrets。
