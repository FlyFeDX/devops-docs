# Kubernetes 应用指南

> Kubernetes 是谷歌开源的容器集群管理系统 是用于自动部署，扩展和管理 Docker 应用程序的开源系统，简称 K8S。
>
> 关键词： `docker`

<!-- TOC depthFrom:2 depthTo:2 -->

- [一、K8S 简介](#一k8s-简介)
- [二、K8S 命令](#二k8s-命令)
- [参考资料](#参考资料)

<!-- /TOC -->

## 一、K8S 简介

K8S 主控组件（Master） 包含三个进程，都运行在集群中的某个节上，通常这个节点被称为 master 节点。这些进程包括：`kube-apiserver`、`kube-controller-manager` 和 `kube-scheduler`。

集群中的每个非 master 节点都运行两个进程：

- kubelet，和 master 节点进行通信。
- kube-proxy，一种网络代理，将 Kubernetes 的网络服务代理到每个节点上。

### K8S 功能

- 基于容器的应用部署、维护和滚动升级
- 负载均衡和服务发现
- 跨机器和跨地区的集群调度
- 自动伸缩
- 无状态服务和有状态服务
- 广泛的 Volume 支持
- 插件机制保证扩展性

### K8S 核心组件

Kubernetes 主要由以下几个核心组件组成：

- etcd 保存了整个集群的状态；
- apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API 注册和发现等机制；
- controller manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler 负责资源的调度，按照预定的调度策略将 Pod 调度到相应的机器上；
- kubelet 负责维护容器的生命周期，同时也负责 Volume（CVI）和网络（CNI）的管理；
- Container runtime 负责镜像管理以及 Pod 和容器的真正运行（CRI）；
- kube-proxy 负责为 Service 提供 cluster 内部的服务发现和负载均衡

![img](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LDAOok5ngY4pc1lEDes%2F-LpOIkR-zouVcB8QsFj_%2F-LpOIpZIYxaDoF-FJMZk%2Farchitecture.png?generation=1569161437087842&alt=media)

### K8S 核心概念

K8S 包含若干抽象用来表示系统状态，包括：已部署的容器化应用和负载、与它们相关的网络和磁盘资源以及有关集群正在运行的其他操作的信息。

![img](https://raw.githubusercontent.com/dunwu/images/dev/cs/os/kubernetes/pod.svg)

- `Pod` - K8S 使用 Pod 来管理容器，每个 Pod 可以包含一个或多个紧密关联的容器。Pod 是一组紧密关联的容器集合，它们共享 PID、IPC、Network 和 UTS namespace，是 K8S 调度的基本单位。Pod 内的多个容器共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。
- `Node` - Node 是 Pod 真正运行的主机，可以是物理机，也可以是虚拟机。为了管理 Pod，每个 Node 节点上至少要运行 container runtime（比如 docker 或者 rkt）、`kubelet` 和 `kube-proxy` 服务。
- `Namespace` - Namespace 是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。常见的 pods, services, replication controllers 和 deployments 等都是属于某一个 namespace 的（默认是 default），而 node, persistentVolumes 等则不属于任何 namespace。
- `Service` - Service 是应用服务的抽象，通过 labels 为应用提供负载均衡和服务发现。匹配 labels 的 Pod IP 和端口列表组成 endpoints，由 kube-proxy 负责将服务 IP 负载均衡到这些 endpoints 上。每个 Service 都会自动分配一个 cluster IP（仅在集群内部可访问的虚拟地址）和 DNS 名，其他容器可以通过该地址或 DNS 来访问服务，而不需要了解后端容器的运行。
- `Label` - Label 是识别 K8S 对象的标签，以 key/value 的方式附加到对象上（key 最长不能超过 63 字节，value 可以为空，也可以是不超过 253 字节的字符串）。Label 不提供唯一性，并且实际上经常是很多对象（如 Pods）都使用相同的 label 来标志具体的应用。Label 定义好后其他对象可以使用 Label Selector 来选择一组相同 label 的对象（比如 ReplicaSet 和 Service 用 label 来选择一组 Pod）。Label Selector 支持以下几种方式：
  - 等式，如 `app=nginx` 和 `env!=production`
  - 集合，如 `env in (production, qa)`
  - 多个 label（它们之间是 AND 关系），如 `app=nginx,env=test`
- `Annotations` - Annotations 是 key/value 形式附加于对象的注解。不同于 Labels 用于标志和选择对象，Annotations 则是用来记录一些附加信息，用来辅助应用部署、安全策略以及调度策略等。比如 deployment 使用 annotations 来记录 rolling update 的状态。

## 二、K8S 架构说明

![K8S功能模型](../../../images/modules/kubernetes功能全景图.png)

按照这幅图的线索，我们从容器这个最基础的概念出发，首先遇到了容器间“紧密协作”关系的难题，于是就扩展到了 Pod；有了 Pod 之后，我们希望能一次启动多个应用的实例，这样就需要 Deployment 这个 Pod 的多实例管理器；而有了这样一组相同的 Pod 后，我们又需要通过一个固定的 IP 地址和端口以负载均衡的方式访问它，于是就有了 Service。

可是，如果现在两个不同 Pod 之间不仅有“访问关系”，还要求在发起时加上授权信息。最典型的例子就是 Web 应用对数据库访问时需要 Credential（数据库的用户名和密码）信息。那么，在 Kubernetes 中这样的关系又如何处理呢？

Kubernetes 项目提供了一种叫作 Secret 的对象，它其实是一个保存在 Etcd 里的键值对数据。这样，你把 Credential 信息以 Secret 的方式存在 Etcd 里，Kubernetes 就会在你指定的 Pod（比如，Web 应用的 Pod）启动时，自动把 Secret 里的数据以 Volume 的方式挂载到容器里。这样，这个 Web 应用就可以访问数据库了。

除了应用与应用之间的关系外，应用运行的形态是影响“如何容器化这个应用”的第二个重要因素。

为此，Kubernetes 定义了新的、基于 Pod 改进后的对象。比如 Job，用来描述一次性运行的 Pod（比如，大数据任务）；再比如 DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务；又比如 CronJob，则用于描述定时任务等等。

如此种种，正是 Kubernetes 项目定义容器间关系和形态的主要方法。

可以看到，Kubernetes 项目并没有像其他项目那样，为每一个管理功能创建一个指令，然后在项目中实现其中的逻辑。这种做法，的确可以解决当前的问题，但是在更多的问题来临之后，往往会力不从心。

相比之下，在 Kubernetes 项目中，所推崇的使用方法是：

- 首先，通过一个“编排对象”，比如 Pod、Job、CronJob 等，来描述你试图管理的应用；

- 然后，再为它定义一些“服务对象”，比如 Service、Secret、Horizontal Pod Autoscaler（自动水平扩展器）等。这些对象，会负责具体的平台级功能。

这种使用方法，就是所谓的“声明式 API”。这种 API 对应的“编排对象”和“服务对象”，都是 Kubernetes 项目中的 API 对象（API Object）。

实际上，过去很多的集群管理项目（比如 Yarn、Mesos，以及 Swarm）所擅长的，都是把一个容器，按照某种规则，放置在某个最佳节点上运行起来。这种功能，我们称为“调度”。

而 Kubernetes 项目所擅长的，是按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。这种功能，就是我们经常听到的一个概念：编排。

所以说，Kubernetes 项目的本质，是为用户提供一个具有普遍意义的容器编排工具。不过，更重要的是，Kubernetes 项目为用户提供的不仅限于一个工具。它真正的价值，乃在于提供了一套基于容器构建分布式系统的基础依赖。

## 三、K8S 命令

### 客户端配置

```bash
# Setup autocomplete in bash; bash-completion package should be installed first
source <(kubectl completion bash)

# View Kubernetes config
kubectl config view

# View specific config items by json path
kubectl config view -o jsonpath='{.users[?(@.name == "k8s")].user.password}'

# Set credentials for foo.kuberntes.com
kubectl config set-credentials kubeuser/foo.kubernetes.com --username=kubeuser --password=kubepassword
```

### 查找资源

```bash
# List all services in the namespace
kubectl get services

# List all pods in all namespaces in wide format
kubectl get pods -o wide --all-namespaces

# List all pods in json (or yaml) format
kubectl get pods -o json

# Describe resource details (node, pod, svc)
kubectl describe nodes my-node

# List services sorted by name
kubectl get services --sort-by=.metadata.name

# List pods sorted by restart count
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# Rolling update pods for frontend-v1
kubectl rolling-update frontend-v1 -f frontend-v2.json

# Scale a replicaset named 'foo' to 3
kubectl scale --replicas=3 rs/foo

# Scale a resource specified in "foo.yaml" to 3
kubectl scale --replicas=3 -f foo.yaml

# Execute a command in every pod / replica
for i in 0 1; do kubectl exec foo-$i -- sh -c 'echo $(hostname) > /usr/share/nginx/html/index.html'; done
```

### 资源管理

```bash
# Get documentation for pod or service
kubectl explain pods,svc

# Create resource(s) like pods, services or daemonsets
kubectl create -f ./my-manifest.yaml

# Apply a configuration to a resource
kubectl apply -f ./my-manifest.yaml

# Start a single instance of Nginx
kubectl run nginx --image=nginx

# Create a secret with several keys
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Secret
metadata:
 name: mysecret
type: Opaque
data:
 password: $(echo "s33msi4" | base64)
 username: $(echo "jane"| base64)
EOF

# Delete a resource
kubectl delete -f ./my-manifest.yaml
```

### 监控和日志

```bash
# Deploy Heapster from Github repository
kubectl create -f deploy/kube-config/standalone/

# Show metrics for nodes
kubectl top node

# Show metrics for pods
kubectl top pod

# Show metrics for a given pod and its containers
kubectl top pod pod_name --containers

# Dump pod logs (stdout)
kubectl logs pod_name

# Stream pod container logs (stdout, multi-container case)
kubectl logs -f pod_name -c my-container
```

## 参考资料

- **官方**
  - [Kubernetes Github](https://github.com/kubernetes/kubernetes)
  - [Kubernetes 官网](https://kubernetes.io/)
- **更多资源**
  - [awesome-kubernetes](https://github.com/ramitsurana/awesome-kubernetes)
