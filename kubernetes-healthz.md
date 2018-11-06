---
title: Kubernetes healthz
layout: 2017/sheet
prism_languages: [bash,yaml]
weight: -3
tags: [Featured]
updated: 2017-11-19
category: Kubernetes
---

## Kubernetes traefik

{: .-three-column}

### **容器健康监测**

kubernetes提供了 liveness probes来检查我们的应用程序。它是由节点上的kubelet定期执行的。

当前kubelet拥有两个检测器，他们分别对应不通的触发器(根据触发器的结构执行进一步的动作)：

- Liveness Probe：表示container是否处于live状态。如果 LivenessProbe失败，LivenessProbe将会通知kubelet对应的container不健康了。随后kubelet将kill掉 container，并根据RestarPolicy进行进一步的操作。默认情况下LivenessProbe在第一次检测之前初始化值为 Success，如果container没有提供LivenessProbe，则也认为是Success；
- ReadinessProbe：表示container是否以及处于可接受service请求的状态了。如果ReadinessProbe失败，endpoints controller将会从service所匹配到的endpoint列表中移除关于这个container的IP地址。因此对于Service匹配到的 endpoint的维护其核心是ReadinessProbe。默认Readiness的初始值是Failure，如果一个container没有提供 Readiness则被认为是Success。

### **Pod的整个生命阶段**

- Pending：表示集群系统正在创建Pod，但是Pod中的container还没有全部被创建，这其中也包含集群为container创建网络，或者下载镜像的时间；
- Running：表示pod已经运行在一个节点上，并且所有的container都已经被创建。但是并不代表所有的container都运行，它仅仅代表至少有一个container是处于运行的状态或者进程出于启动中或者重启中；
- Succeeded：所有Pod中的container都已经终止成功，并且没有处于重启的container；
- Failed：所有的Pod中的container都已经终止了，但是至少还有一个container没有被正常的终止(其终止时的退出码不为0)

对于liveness probes的结果也有几个固定的可选项值：

- Success：表示通过检测
- Failure：表示没有通过检测
- Unknown：表示检测没有正常进行

Liveness Probe的种类：
- ExecAction：在container中执行指定的命令。当其执行成功时，将其退出码设置为0；
- TCPSocketAction：执行一个TCP检查使用container的IP地址和指定的端口作为socket。如果端口处于打开状态视为成功；
- HTTPGetAcction：执行一个HTTP默认请求使用container的IP地址和指定的端口以及请求的路径作为url，用户可以通过host参数设置请求的地址，通过scheme参数设置协议类型(HTTP、HTTPS)如果其响应代码在200~400之间，设为成功。
对于LivenessProbe和ReadinessProbe用法都一样，拥有相同的参数和相同的监测方式。
- initialDelaySeconds：用来表示初始化延迟的时间，也就是告诉监测从多久之后开始运行，也就是容器启动后第一次执行探测需要等待多少秒
- timeoutSeconds: 用来表示监测的超时时间，如果超过这个时长后，则认为监测失败
- periodSeconds：执行探测的频率。默认是10秒，最小1秒。
- successThreshold：探测失败后，最少连续探测成功多少次才被认定为成功。默认是1。对于liveness必须是1。最小值是1
- failureThreshold：探测成功后，最少连续探测失败多少次才被认定为失败。默认是3。最小值是1。

当前对每一个Container都可以设置不同的restartpolicy，有三种值可以设置：

- Always: 只要container退出就重新启动
- OnFailure: 当container非正常退出后重新启动
- Never: 从不进行重新启动

> 如果restartpolicy没有设置，那么默认值是Always。如果container需要重启，仅仅是通过kubelet在当前节点进行container级别的重启。

 



### 以traefik daemeset为例进行说明

```yaml
readinessProbe:
    tcpSocket:
    port: 80
    failureThreshold: 1
    initialDelaySeconds: 10
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 2
livenessProbe:
    tcpSocket:
    port: 80
    failureThreshold: 3
    initialDelaySeconds: 10
    periodSeconds: 10
    successThreshold: 1
    timeoutSeconds: 2
```
> livenessProbe 部分：
>
> ​	使用tcpSocket 方式监测容器80端口是否可用， initialDelaySeconds 容器启动后10s开始监测， `periodSeconds` 每10s监测一次， `successThreshold` 成功一次认为服务是可以正常对外提供服务的， `failureThreshold` 失败3次认为服务不能对外提供服务， 如果检测2s没有没有响应返回超时。
>
> readinessProbe部分：
>
> 和上面解释基本相同，唯一不同认为失败一次认定改容器不能对外提供服务， 从service中删除

### 完整示例
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-test
  labels:
    k8s-app: nginx
    traffic-type: external
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx  
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        readinessProbe:
          tcpSocket:
            port: 80
          failureThreshold: 1
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 2
        livenessProbe:
          tcpSocket:
            port: 80
          failureThreshold: 3
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 2

```


### HTTPGetAcction 使用示例
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-test
  labels:
    k8s-app: nginx
    traffic-type: external
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx  
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            port: 80
            path: /healthz
          initialDelaySeconds: 1
          periodSeconds: 5
          timeoutSeconds: 4
          successThreshold: 2
          failureThreshold: 3
```