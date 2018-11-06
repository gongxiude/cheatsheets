---
title: Kubernetes ingress
layout: 2017/sheet
prism_languages: [bash,yaml]
weight: -3
tags: [Featured]
updated: 2017-11-19
category: Kubernetes
---

## Kubernetes traefik
# K8s Ingress基本概念

​ [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) & [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 都会各自拥有一个 IP address 供读取，但这些 IP 仅在 K8s cluster 內部才有办法读取的到，但若要在 K8s cluster 上提供对外服务呢?

而目前可以外部存取 K8s 上服务的方式主要有三种：

1. **Service NodePort**: 这方法会在每个 master & worker node 上都开启指定的 port number (这样其实造成不少资源浪费)， 不方便进行端口管理

2. **Service LoadBalancer**: 只有在 GCP or AWS 这类的 public cloud 平台才有用的功能

3. **Ingress** 就是被设计來处理这类的问题


   在沒有 Ingress 之前，从外面到 K8s cluster 的流量会如下图：

   ```
     internet
         |
   ------------
   [ Services ]
   ```

   > 但在原有的模式下，如果是在公有云 使用 K8s 的话，还可以搭配 LoadBalancer 的 service type 动态取得 LB 对外提供服务；但如果是自己架设 K8s，那就只能透过 Service NodePort 的方式让使用者从外部访问运行在 K8s 上的服务。

   Ingress 就是集群内所有服务的入口，rule 的集合，让外面进來的网路流量可以正确的被导到后方的 Service，架构如下图：

   ```
   internet
     	|
   [ Ingress ]
   --|-----|--
   [ Services ]
   ```

Ingress 可以负责以下工作：
  
- 基于http-header 的路由
- 基于 path 的路由
- 登录验证
- cros
- 请求速率limit
- rewrite 规则
- ssl

## Service、Ingress与Ingress Controller的作用与关系

- Service 是后端真实服务的抽象，一个 Service 可以代表多个相同的后端服务
- Ingress 是反向代理规则，用来规定 HTTP/S 请求应该被转发到哪个 Service 上，比如根据请求中不同的 Host 和 url 路径让请求落到不同的 Service 上
- Ingress Controller 就是一个反向代理程序，它负责解析 Ingress 的反向代理规则，如果 Ingress 有增删改的变动，所有的 Ingress Controller 都会及时更新自己相应的转发规则，当 Ingress Controller 收到请求后就会根据这些规则将请求转发到对应的 Service。

请求流程： 
Ingress Controller 收到请求，匹配 Ingress 转发规则，匹配到了就转发到后端 Service，而 Service 可能代表的后端 Pod 有多个，选出一个转发到那个 Pod，最终由那个 Pod 处理请求。


## ingress 的暴漏方式

- Ingress Controller 用 Deployment 方式部署，给它添加一个 Service，类型为 LoadBalancer，这样会自动生成一个 IP 地址，通过这个 IP 就能访问到了，并且一般这个 IP 是高可用的（前提是集群支持 LoadBalancer，通常云服务提供商才支持，自建集群一般没有）
- 使用集群内部的某个或某些节点作为边缘节点，给 node 添加 label 来标识，Ingress Controller 用 DaemonSet 方式部署，使用 nodeSelector 绑定到边缘节点，保证每个边缘节点启动一个 Ingress Controller 实例，用 hostPort 直接在这些边缘节点宿主机暴露端口，然后我们可以访问边缘节点中 Ingress Controller 暴露的端口，这样外部就可以访问到 Ingress Controller 了
- Ingress Controller 用 Deployment 方式部署，给它添加一个 Service，类型为 NodePort，部署完成后查看会给出一个端口，通过 kubectl get svc 我们可以查看到这个端口，这个端口在集群的每个节点都可以访问，通过访问集群节点的这个端口就可以访问 Ingress Controller 了