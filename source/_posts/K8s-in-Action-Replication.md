---
title: K8s Replication and other controllers 
categories: 
- K8s
date: 2021-04-04 21:30:00
---

K8s可以在进程crash的时候重启容器，但如果进程没有crash，只是抛出错误或者进程出现了没有预料到的错误，比如死循环，死锁之类的该怎么办？因此要求可以从外部探查应用的健康状况。
有三种机制的探针：HTTP GET，要求返回状态码应该是2XX或者3XX；TCP socket，要求可以建立连接；运行容器内的一条命令，要求exit status code为0。  
不存在一个liveness探针这样的资源，而是在创建pod的时候附带创建探针。
```
apiVersion: v1
kind: pod
metadata:
name: kubia-liveness spec:
containers:
- image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```
这个pod的yaml带了一个**httpGet**探针，要求K8s定期发送HTTP GET请求到8080端口来监测服务是否健康。  
查看restart的pod重启之前的日志用```kubectl logs <pod_name> --previous```  
这个容器的进程会每5次返回500一次，如果对这个pod用describe可以在Events看到因为探针失败而重启**容器**的记录。还可以看到上次被terminated的exit code，这个code 128+x，x是引发terminate的信号编号。  
Liveness探针的参数可配，比如超时时间，失败数目上限，还有探针在容器启动后多久开始工作(initialDelaySeconds)。 
探针的配置的一些注意点：
1 最好有一个/health之类的路径专门用来处理探针请求，监测整个服务的健康状况。
2 探针应该只检查服务的内部状态而不受外部应用影响，比如接数据库的web server，如果数据库坏了，server的探针不应该报失败。
3 探针应该轻量，因为探针会被高频次地运行，且探针的计算资源开销是计入整个pod的资源开销的。
4 探针自己不需要重试，因为K8s会自动重试探针。

因为探针失败而重启容器这件事是由承载这个pod的node上的kubelet负责的，K8s Control Plane不参与这个行为。但是如果这个node坏了，必须由Control Plane来为这个crash的node上面的所有pod找一个新的宿主机，但是手动创建的pod不由Control Plane管理，只有kubelet管理。因此需要Replication Controller之类的机制。

## Replication Controller
rc是一种可以为pod维护指定replica数目的资源。rc会持续不断地监测符合某一个label selector的pod的数量，保证这个数量和设置的一样。  
一个rc由三个重要部分组成：
1 一个label selector，决定这个rc管理什么pod
2 一个replica count
3 一个pod template，用于创建新pod
更改rc的label selector和pod template对于正在运行的pod没有影响(变更label selector使得一些pod不受rc管理了应该也算影响才对)。

一个pod实例不会被挪到别的node，rc只会在一个新node上创建一个全新的pod来替代。  

用yaml来创造rc
```
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia 
template:
  metadata:
    labels:
      app: kubia
  spec:
    containers:
      - name: kubia
        image: luksa/kubia 
        ports:
        - containerPort: 8080
```
Pod template中的label必须和label selector中的一致，否则rc会不停地创建pod，所以个API server会检查这个点，如果不一致视为配置出错。可以不指定label selector，这种情况下会自动用template里面的label来作为selector。  

rc对于监控pod的变动是通过API server实现的，API server允许用户watch资源的变化，当rc注意到有新pod被创建或者删除的时候，这个创建或删除并不触发rc的去管理pod。rc会先用label selector查一遍自己管理的pod数目再作出反应。  

rc和pod不是强绑定的，rc只会管理符合label selector的pod。如果改变了一个pod的label，就可以实现将其移入移出rc的功能。不过虽然pod和rc不绑定，但是还是可以通过pod的```metadata.ownerReferences```查询到它属于什么rc。  

命令```kubectl edit rc <rc_name>```可以更改rc的yaml definition。这样可以更改rc的三个部分。但是如果单纯向扩展。可以用命令```kubectl scale rc <rc_name> --replica= ```。  

删除一个rc会一起删除这个rc管理的pod，但也可以不删除这些pod，这一点由```--cascade```选项控制。  

## Replication Set
replication set是用来代替rc的，它几乎和rc一样，但是有更丰富的label selector支持。但是rs和rc一般也不都直接被创建，而是由一种更高级的资源deployment(Ch9)来自动创建它们。  

一个rs 的yaml definition。
```
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
```
从这个yaml可以看出几点，一是rs的API版本不是v1而是apps/v1beta，apps是API group，v1beta是API version；其次是label selector用的是一个matchLabels。现在这个rs和之前的rc没有任何区别。但是rs提供下面的label selector已支持更丰富的label匹配。  
```
selector:
  matchExpressions:
    - key: app 
      operator: In 
      values:
	- kubia
```
这个operator可以在```In```, ```NotIn```, ```Exitsts```, ```DoesNotExist```中取值。如果使用的多个表达式和matchLabel，那么必须同时满足所有表达式才算匹配。  

## DaemonSets
rc和rs只能控制pod的数目，但是如果希望每个node都能部署一个且仅一个pod这种需求它们满足不了。DaemonSets可以解决这个问题。  
DaemonSets创建的pods因为已经有了target node所以不需要Scheduler调度。  

DaemonSets也可以只在部分nodes上创建pod，这一项由pod template中的nodeSelector来管理。DaemonSets创建的pod忽略node的unschedulable状态，因为DaemonSets创建的pod不走Scheduler。而且这也很合理，因为DaemonSet管理的一般是系统服务，这些服务不关心这个node上是不是有pod。

```
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: ssd-monitor 
spec:
  selector:
    matchLabels:
      app: ssd-monitor 
template:
    metadata:
      labels:
        app: ssd-monitor 
spec:
      nodeSelector:
        disk: ssd
      containers:
        - name: main
          image: luksa/ssd-monitor
```

## Job 
之前的pod都是持续运行的，如果希望一个pod执行完工作就终止就需要Job这种资源。  
Job的容器当进程正常结束之后不会重启，而是被视为完成了。如果因为node崩溃这种情况造成由job管理的pod崩溃，job会在另外的node上重启一个pod重新做任务。如果是pod内的进程执行失败，job可能会重试任务或者什么也不做。  
```
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job 
spec:
  template:
    metadata:
      labels:
        app: batch-job 
    spec:
      restartPolicy: OnFailure 
      containers:
        - name: main
          image: luksa/batch-job
```
restartPolicy的默认值的```Always```，这是所有Pod的默认值。但是由Job管理的Pod不能设置为该值，因为它设计的目的是为一次性任务服务。可以配置为```OnFailure```或者```Never```。  
一个Job可以管理多个pod，这样可以并行或者顺序地执行多个任务，通过设置Job spec的```completions```或者```parallelism```来做到。如果设置```completions=5```，那么job就会顺序地运行5个pod。可以同时设置这两个属性使得Job并行地运行多个任务。还可以在Job工作期间热更新并行数。
```
kubectl scale job multi-completion-batch-job --replicas 3
```
限制Job中的任务的完成时间通过设置spec的```activeDeadlineSeconds```做到，重试上限通过```backoffLimit```设置。  

Job资源在创建之后会立刻工作，如果由定期或者希望未来完成的任务，可以使用CronJob。
```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes 
spec:
  schedule: "0,15,30,45 * * * *"
  startingDeadlineSeconds: 15 \\因为CronJob不能精确地在设置时间执行，这里给一个延迟上限，超过会视为失败 
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job 
        spec:
  	  restartPolicy: OnFailure 
          containers:
 	    - name: main
	      image: luksa/batch-job
```
一般情况下，CronJob只会创建一个Job。但存在两个Job被同时创建或者有时候没有Job被创建的情况。因此需要保证你的任务的幂等的，并且任务可以完成上一轮没有做的工作。

