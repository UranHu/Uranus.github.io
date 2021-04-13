---
title: K8s Pods 
categories: 
- K8s
date: 2021-04-03 21:30:00
---
Pod是K8s的中心概念，K8s的其他资源不是管理，暴露Pod，就是被Pod使用。  
Pod是一组容器，但大多数情况下pod里只有一个容器。如果pod中含有多个容器，则它们一定运行在同一个worker node上。  
多个运行单进程的**容器**优于运行多进程的单容器，因为容器没有管理进程的功能，如果诸多进程中的一个挂了，容器不能自行把它拉起来。另外多个进程在一个容器里会共用很多资源。例如stdout，这些进程的log会都挤在一起，无法分辨。  
容器设计之时就是为了运行单进程的，除非是一个进程自己引发的子进程的情况。因为必须保证单个容器单个进程，但由存在需要多个进程协同工作的情况。因此pod通过囊括多个容器来填补这个空白。  
K8s通过令一个pod上的所有容器使用相同的Linux namespace来模拟多个容器运行在一台物理机的情况。但是每个容器拥有其各自的文件系统，可以通过K8s的Volume令其共享一套文件系统。另外因为一个pod的多个容器相当于运行在一台物理机上，因此它们应该注意端口冲突的问题。  
Pods间的网络是平的，即它们可以通过ip地址直接找到对方，不需要通过NAT。无论是同一worker node上的pods还是不同worker nodes上的pods都是如此。

Pod是K8s扩展的最小单位，而不是容器。因此应该把不同的进程放在不同的pod中，这样扩展比较方便。比如无状态的web server和后端数据库就应该分开，这样web server扩容时，数据库不应该随之增减实例。  
但是如果是一个主要进程和若干个负责辅助其工作的进程，那么就可以放在一个pod中。  

总之，牢记尽量避免把多个进程放在一个容器，以及把多个容器放在一个pod就可以了。

## 通过YAML或JSON来创建pod
[K8s API doc](http://kubernetes.io/docs/reference/)

通过```kubectl get po pod-name -o yaml```可以看到这个pod完整的yaml definition。一个yaml definition的主要内容包括
1 本yaml文件使用的K8s API version以及本yaml文件描述的资源类型
2 Metadata: 关于这个pod的信息，如name，namespace，label等
3 Spec: pod实际内容的描述，比如这个pod有什么容器，volume等
4 Status: 包含正在运行的pod的状态, 比如它现在运行正常吗，它的内部ip等其他基础信息

Pod的yaml definition内容很丰富，在用yaml创建pod的时候不用提供4。下面是一个创建pod的yaml的样例。
```
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual 
spec:
  containers: //这个pod包含1个容器
  - image: luksa/kubia
    name: kubia //给这个容器一个名字
    ports:
    - containerPort: 8080
      protocol: TCP
```
在yaml文件中指明port与否对于能不能通过这个端口个连上容器里面的服务没有影响，只不过可以让看yaml的人看得更明白。

在撰写manifest的时候，除了可以参考K8s API的文档，还可以利用```kubectl explain```命令查看各字段的含义。例如
```
kubectl explain pods
kubectl explain pod.spec
```

若想通过yaml文件创建pod，运行```kubectl create -f kubia-manual.yaml```即可。

在部署应用后希望查询应用的日志，调用```kubectl logs pod_name```或者登陆到worker node运行```docker logs <container_id>```。容器中的进程一般把日志直接写到stderr和stdout，而不是日志文件中。因为这样对于K8s和Docker可以直接获得日志。日志每天以及每10MBrotate一次，```kubectl logs```展示的是从上一次rotate到现在的日志。  
如果一个pod里面有多个容器，那么查询日志时应该指定容器名，即
```kubectl logs <pod_name> -c <container_name>```  
K8s只能查到还存在的容器的日志，如果想要保留被删除的容器的日志，需要一个cluster维度的日志中心(Ch17)。  

虽然pod创建成功，但是其中的服务还是无法访问。除了之前说的service，端口转发(port forwarding)也可以让用户可以访问到pod中的服务。端口转发是让用户通过本地的端口访问服务，service是给pod暴露了一个持久稳定的ip，无需端口，仅用ip即可。  
```
kubectl port-forward kubia-manual 8888:8080
```
运行了该命令后，kubectl会拦截本地8888端口的请求，转发到kubia-manual这个pod的8080端口。

## 用label管理pods
Label是用户给K8s资源打上的任意键值对。一般会在创建资源的时候指定label，但是之后也可以热增加or热修改label。  
Label可以在些yaml文件的时候就写好。
```
...
metadata:
  name: kubia-manual-v2
  labels:
    creation_method: manual
    env: prod
...
```
运行下面命令查看label。
```
kubectl creata -f kubia-manual-v2
kubectl get po --show-labels 
kubectl get po -L creation_method,env \\ -L选项指定labels
```
修改或增加label的命令
```
kubectl label po kubia-manual-v2 env=debug --overwrite //必须带--overwrite选项才能修改已有label
```

### label selector
使用```-l```选项加上条件可以通过label筛选资源。  
```
kubectl get po -l creation_method=manual / env / '!env' / != / in (a, b) / notin (a,b) ...
```

同样也可以通过label管理worker node。  
```
kubectl label node <node_name> gpu=true
```
使用```nodeSelector```可以把pod指定调度到符合某个label selector的worker nodes上。
```
spec:
  nodeSelector:
    gpu: "true" //这个pod只会被调度到包含gpu=true的node
  containers:
```
每个node都有一个名为kubernetes.io/hostname的label，这个label的值是node是主机名。但是没有必要不要使用这个label来调度pod，因为这样使pod对单个node依赖。

## Annotation 注解 pod
Annotating与label的区别在于
1 它的值可以很长，因为它是用来说明注解pod的键值对
2 它对于调度无影响
Annotations经常也被用于引入新feature。新的feature在beta阶段往往在annotation中出现。
```
kubectl annotate pod <pod_name> <key>=<value>
```

## 使用命名空间对资源进行分组
Namespace的作用是把资源分组，资源的名称只需在命名空间内唯一即可。但是cluster级别的资源不受namespace控制，比如node。除了资源隔离外，namespace还可以用来做访问控制和资源管理。

可以用yaml文件创建namespace，也可以用命令。用命令比简单。
```
kubectl create namespace custom-namespace
```
切换namespace
```
kubectl config set-context $(kubectl config current-context) --namespace <namespace_name>
```

Namespace提供的隔离不是低级别的，即不同的ns的pod是可以互相看见对方的。ns的网络隔离依赖与K8s的网络策略，如果K8s没有做ns间的网络隔离，那么就没有隔离。

## 停止和移除pod
```
kubectl delete po <pod_name>
kubectl delete po -l <label selector>
```
删除pod，K8s会终止这个pod的所有容器，K8s会向进程发送SIGTERM信号并等待一段时间。如果进程没有关掉，会再发送SIGKILL。因此应该保证容器中的进程可以处理SIGTERM信号。

也可以通过删除整个namespace来删除pod，删除ns可以删除ns上的所有pod。或者用```--all```选项。  
如果使用来```kubeclt delete all --all```来删除所有资源，第一个all就是表示所有资源。但是仍然会有一些资源不会被这个命令删除，需要显式删除，比如Secret。K8s service也会被这个命令删除，但是会再重新拉起来一个。
