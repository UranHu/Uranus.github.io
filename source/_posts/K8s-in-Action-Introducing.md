---
title: Introducing K8s 
categories: 
- K8s
date: 2021-04-01 21:30:00
description: Introducing Kubernetes 
---
为什么需要K8s?  
- 因为由单体式程序(monolithic)转向微服务式程序，因此需用像K8s这种工具帮助开发者和运维团队。  
- 微服务通过水平扩容来提高请求处理能力，要求能运维上能迅速扩容/缩容。
- 不同的微服务依赖于不同版本的库甚至环境，对于这些资源的隔离需用docker以及K8s这类工具。
- K8s能实现运维的自动化，docker减小了开发，测试和线上环境的差异，为应用提供一个更稳定的环境。
- ...

Docker通过Unix/Linux的命名空间(namespace)实现比VM更轻量的资源隔离。Docker的主要概念包括下面三个。
- images: 将应用和环境打包的产物
- registries: 存储Docker image的仓库
- containers: 通过Docker image创建的Linux容器

Docker images是由多层(layers)组成的，不同的images可以包含相同的层。image的层是只读的，因此共享同一层的不同容器可以看到相同的文件系统，但是不能看到对方的修改。修改会作为一个新层加上去。

## K8s的结构
K8s由master和worker node(s)组成。Master上运行着control plane，控制和管理整个K8s系统，worker nodes真正运行部署的应用。  
Control Plane
- Kubernetes API Server: 用户和control plane其他组件交互的工具。
- Scheduler: 为需要部署的应用调度worker nodes。
- Controller Manager: 执行集群级别的工作，例如复制组件，追踪worker nodes，处理节点失败等情况。
- etcd: 一个强一致配置中心。

Nodes
- container runtime: docker, rkt等运行容器的。
- Kubelet: 和API Server交互，管理本node上的容器。
- kube-proxy: 负责管理应用之间的网络。

用户通过声明需要部署什么应用，各需要多少各replica，每个应用的image应该如何获取。K8s会自己决定应该把各个应用实例调度到那个node上。
![K8s的基本运作方式](/images/K8s_basic_overview.png)
