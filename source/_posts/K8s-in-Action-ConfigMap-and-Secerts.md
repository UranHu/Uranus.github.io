---
title: K8s ConfigMaps and Secrets
categories: 
- K8s
date: 2021-04-06 21:30:00
---
如何将配置项传给K8s中运行的App  
1 命令行配置参数或者配置文件
2 通过环境变量

在Docker中使用配置文件会很麻烦，如果把配置文件集合到imgae里面，那么不安全，所有对image有权限的人都能看到配置文件内容，其次不好管理，因为要修改配置参数就要重新打镜像。用volume也差不多，需要保证容器拉起来前文件被写到volume了。用gitRepo可能好点，但没必要。因为有ConfigMaps。  
但是配置项里面可能还有一些敏感的参数，比如私钥等需要保密的。K8s提供了另外一种一等对象Secret来管理这些东西。  
## 给容器传入命令行参数
在容器中执行的命令分为两部分，*command*和*argument*。在一个Dockerfile中，用  
**ENTRYPOINT**描述容器启动时应该运行什么命令  
**CMD**描述应该传给**ENTRYPOINT**的参数  
虽然**CMD**也可以用来运行命令，但是它正确的用法是给**ENTRYPOINT**传入参数。Docker镜像可以通过
```
docker run <image>
```
直接默认参数运行，也可以通过说明参数来覆盖**CMD**的值。
```
docker run <image> <arguments>
```

并且，这两种指令都支持**shell**和**exec**两种形式。
**shell**: ENTRYPOINT node app.js
**exec**: ENTRYPOINT ["node", "app,js"]  
它们两个的区别在于命令是否在一个shell里运行。  
如果shell中运行，那么PID 1的进程不是命令而是shell。  


