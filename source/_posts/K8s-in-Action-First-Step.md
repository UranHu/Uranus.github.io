---
title: First steps with Docker and K8s 
categories: 
- K8s
date: 2021-04-02 21:30:00
---

## 简单使用Docker
运行一个Docker容器
```
docker run busybox echo "Hello world"
```
这条命令背后是
1 Docker检查本地是否存在busybox镜像
2 Docker从registry拉取busybox镜像
3 Docker在一个被隔离的容器里面运行''' echo "Hello world" '''

但是镜像一般是和想要运行的命令一起打包的，因此一般只要run一个打包进命令的image就可以了。
```
docker run <image>:<tag>
```
例如，如果有一个node.js编写的http server源文件app.js，想要这个http server在容器中运行。可以提供像这样的一个Dockerfile。这个Dockerfile应该和app.js在一个文件夹下
```
FROM node:7 // base image
ADD app.js /app.js //表示将当前目录的app.js加入镜像的根目录下
ENTRYPOINT ["node", "app.js"] //运行镜像时应该执行的命令
```
运行下面的命令制作一个镜像。
```
docker build -t kubia .
```
.表示基于当前目录制作image。在这个制作镜像的过程中，Docker将当前这个目录上传到Docker Daemon，Docker Daemon如果本地没有base image就去拉取，然后制作出新镜像。因此不要往文件夹里面放无用的文件。  

镜像不是一个大的二进制blob，而是由多层组成的。每个Dockerfile中的command都会在base image上再加一层。例如上面的Dockerfile中的ADD会给node:7增加一个layer，CMD会给它再增加一个layer。  

除了依靠Dockerfile，也可以手动制作镜像：拉起一个image，进去容器运行命令，退出后提交image即可。事实上Dockerfile也是这么做的，但是使用Dockerfile可以让以后制作同样镜像的行为自动化。

```
docker run --name kubia-container -p 8080:8080 -d kubia
```
这个命令让Docker运行一个名为kubia-container的容器，镜像为kubia。-d表示这个容器会与控制台分离(detached)，即在后台运行。宿主机的8080端口会映射到容器的8080端口。对于win和mac，docker的宿主机是运行Docker Daemon的VM，可以通过DOCKER_HOST环境变量查看它的IP地址。  

```
docker ps // list running containers
docker inspect container_name //输出一个json文档对容器进行详细描述
docker exec -it container_name bash // -i表示打开STDIN，-t表示分配一个伪终端(TTY)
docker stop container-name //仅仅是stop的，可以通过docker ps -a看到
docker rm container-name //彻底移除这个容器
```

容器中的pid使用的是自己的namespace，因此要运行的命令的pid为1。在宿主机上也能看到容器中运行的进程，但其pid必然不是1。

### 把image推到image registry上
按照Docker Hub要求，image的tag应该由Docker Hub ID开头，可以通过```docker tag kubia luksa/kubia```给这个image增加一个tag，这里并不是将image重命名了。通过```docker images```可以看到现在是同一个image拥有两个tag。

```
docker push luska/kubia
docker run -p 8080:8080 -d luksa/kubia //如果本地没有则从Docker Registry拉取镜像
```

## 简单使用K8s
首先应该有一个K8s集群，可以通过Minikube或者GKE等创建。  
如图，用户通过kubectl与K8s集群交互。
![用户与K8s集群交互](k8s_fig2.png)

### 通过K8s部署应用
```
kubectl run kubia --image=luska/kubia --port=8080 --generator=run/v1
```
这条命令创建了一个名为kubia的replication controller，K8s中不直接创建pod，而是用过创建replication set或者deployment来创建pod。选项```--genertator=run/v1```表示这里创建的是一个replication controller而不是deployment。```--port=8080```表示这个应用监听8080端口。

Pod是若干相关联的容器组成的组，因此一个pod中可以有多个容器。每个容器运行一个进程，这些进程就像运行在一台物理机上一样。Pod是K8s的基础构建模块，但K8s一般不直接创建pod。

在上面这个命令中发生了以下几步
1 kubectl发出一个RESTful call给API Server
2 master node创建一个pod并将其调度到一个worker node上
3 被调度的worker node的kubelet看到pod被调度来，驱使Docker去拉镜像
4 Docker运行image，创建容器


但刚刚部署的服务仅能通过pod的cluster ip访问到，但是pod是短命的，因此这个服务的ip随时可能变动。因此通过service为这个应用暴露一个持久稳定的ip地址。因为svc分配到的ip在它的整个生命周期都不会变。或者说，因为svc生命周期长，所以它的ip很稳定。svc代表的是运行同一类服务的1个or多个pods的一个静态地址。
```
kubectl expose rc kubia --type=LoadBalancer --name kubia-http
```

命令```kubectl expose```暴露了一个名为kubia-http的服务(service)，通过```kubectl get svc```可以看到这个svc的external ip，现在就可以通过这个external ip访问pod上的服务了。

在上面的例子中，```kubectl run```创建了一个replication controller，这个rc维护数量为1的pod。为了让这个pod能被K8s外界访问到，命令K8s将所有由这个rc管理的pod暴露为一个服务。
![一个rc，一个pod和一个svc]()

Replication Controller可以通过命令快速水平扩容/缩容。
```
kubectl scale rc kubia --relicas=3
```
