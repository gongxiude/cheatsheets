---
title: Kubernetes apisix ingress controller
layout: 2017/sheet
prism_languages: [bash,yaml]
weight: -3
tags: [Featured]
updated: 2017-11-19
category: Kubernetes
---

## K8s Ingress基本概念
{: .-one-column}
​[Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) & [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 都会各自拥有一个 IP address 供读取，但这些 IP 仅在 K8s cluster 內部才有办法读取的到，但若要在 K8s cluster 上提供对外服务呢?

而目前可以外部存取 K8s 上服务的方式主要有三种：

1. **Service NodePort**: 这方法会在每个 master & worker node 上都开启指定的 port number (这样其实造成不少资源浪费)， 不方便进行端口管理

2. **Service LoadBalancer**: 只有在 GCP or AWS 这类的 public cloud 平台才有用的功能

3. **Ingress** 就是被设计來处理这类的问题在沒有 Ingress 之前，从外面到 K8s cluster 的流量会如下图：

```
  internet
      |
------------
[ Services ]
```

但在原有的模式下，如果是在公有云 使用 K8s 的话，还可以搭配 LoadBalancer 的 service type 动态取得 LB 对外提供服务；但如果是自己架设 K8s，那就只能透过 Service NodePort 的方式让使用者从外部访问运行在 K8s 上的服务。

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
{: .-one-column}
- Service 是后端真实服务的抽象，一个 Service 可以代表多个相同的后端服务
- Ingress 是反向代理规则，用来规定 HTTP/S 请求应该被转发到哪个 Service 上，比如根据请求中不同的 Host 和 url 路径让请求落到不同的 Service 上
- Ingress Controller 就是一个反向代理程序，它负责解析 Ingress 的反向代理规则，如果 Ingress 有增删改的变动，所有的 Ingress Controller 都会及时更新自己相应的转发规则，当 Ingress Controller 收到请求后就会根据这些规则将请求转发到对应的 Service。

请求流程： 
Ingress Controller 收到请求，匹配 Ingress 转发规则，匹配到了就转发到后端 Service，而 Service 可能代表的后端 Pod 有多个，选出一个转发到那个 Pod，最终由那个 Pod 处理请求。

{: .-one-column}
## ingress 的暴漏方式

- Ingress Controller 用 Deployment 方式部署，给它添加一个 Service，类型为 LoadBalancer，这样会自动生成一个 IP 地址，通过这个 IP 就能访问到了，并且一般这个 IP 是高可用的（前提是集群支持 LoadBalancer，通常云服务提供商才支持，自建集群一般没有）
- 使用集群内部的某个或某些节点作为边缘节点，给 node 添加 label 来标识，Ingress Controller 用 DaemonSet 方式部署，使用 nodeSelector 绑定到边缘节点，保证每个边缘节点启动一个 Ingress Controller 实例，用 hostPort 直接在这些边缘节点宿主机暴露端口，然后我们可以访问边缘节点中 Ingress Controller 暴露的端口，这样外部就可以访问到 Ingress Controller 了
- Ingress Controller 用 Deployment 方式部署，给它添加一个 Service，类型为 NodePort，部署完成后查看会给出一个端口，通过 kubectl get svc 我们可以查看到这个端口，这个端口在集群的每个节点都可以访问，通过访问集群节点的这个端口就可以访问 Ingress Controller 了

{: .-there-column}
##  ingress.class示例

### **通过ingress.class方式暴露服务**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: apisix-dashboard
  labels:
    app: apisix-dashboard
    department: ops
    dedicated: apisix
  annotations:
    kubernetes.io/ingress.class: apisix
spec:
  rules:
  - host: apisix-dashboard.example.com
    http:
      paths:
      - backend:
          serviceName: apisix-dashboard
          servicePort: 80
        path: /*
        pathType: Prefix
```

>  首先确定apifix-ingress-controller配置文件中ingress_class的值， 默认为apisix
>  注意如果要匹配跟下面的所有路径，需要将path配置为`/*`, 也可以配置`pathType: Prefix`会创建`/` `/*` 两个路径
>  其它的用法完全符合ingress的默认配置，annotation可配置参数参考[官方文档](https://apisix.apache.org/zh/docs/ingress-controller/concepts/annotations/)

{: .-there-column}
## **crd基础示例**

### **ApisixRoute基本用法**

先在集群中部署httpbin服务

```
kubectl run httpbin --image kennethreitz/httpbin --port 80
kubectl expose pod httpbin --port 80
```

创建httpbin-route.yaml文件，内容如下：

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
spec:
  http:
  - name: rule1
    match:
      hosts:
      - httpbin.example.com
      paths:
      - /*
    backends:
       - serviceName: httpbin
         servicePort: 80
```
> 可以使用两种路径类型prefix、exact,默认为exact，如果prefix需要，只需附加一个*，例如，/id/*匹配前缀为 的所有路径/id/。

测试过程如下：

```bash
➜  ~ curl -H "Host: httpbin.example.com" http://internal/headers/ -v
*   Trying internal...
* TCP_NODELAY set
* Connected to internal (internal) port 80 (#0)
> GET / HTTP/1.1
> Host: httpbin.example.com
> User-Agent: curl/7.64.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: text/html; charset=utf-8
< Content-Length: 9593
< Connection: keep-alive
< Date: Tue, 16 Aug 2022 09:07:36 GMT
< Access-Control-Allow-Origin: *
< Access-Control-Allow-Credentials: true
< Server: APISIX/2.15.0
```

### **ApisixTls基本用法**

ApisixTls主要用来维护https证书

创建httpbin-tls.yaml文件，内容如下：

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixTls
metadata:
  name: wildcard.example.com-ApisixTls
spec:
  hosts:
  - "*.example.com"
  secret:
    name: wildcard.example.com-tls-secret
    namespace: ingress-apisix
```

> 通过该方式创建的ssl证书，对apisix全局服务都有效，apisix会根据请求域名动态选择

### **ApisixUpstream基本用法**

ApisixUpstream 需要配置成和Kubernetes Service相同的名字，通过添加负载均衡、健康检查、重试、超时参数等使 Kubernetes Service 更加丰富。

如果不使用ApisixUpstream，则loadbalancer的类型为roundrobin

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixUpstream
metadata:
  name: httpbin
spec:
  loadbalancer:
    type: ewma
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
spec:
  selector:
    app: httpbin
  ports:
  - name: http
    port: 80
    targetPort: 8080
```
> 其中loadbalancer支持如下三种类型
> **Round robin:**  轮询，配置为`type: roundrobin`
> **Least loaded:** 最小链接,配置为`type: least_conn`
> **Peak EWMA:** 维护每个副本往返时间的移动平均值，按未完成请求的数量加权，并将流量分配到成本函数最小的副本。配置为`type: ewma`
> **chash:** 一致性哈希 配置为`type: chash`



{: .-there-column}
## **ApisixRoute使用进阶**

### **通过ApisixRoute methods方式路由**

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: method-route
spec:
  http:
    - name: method
      match:
        hosts:
        - httpbin.example.com
        paths:
        - /
        methods:
        - GET
      backends:
      - serviceName: foo
        servicePort: 80
```

HTTP Methods
- GET
- POST
- PUT
- HEAD
- DELETE
- PATCH
- OPTIONS
- CONNECT
- TRACE

### **通过ApisixRoute exprs方式路由**

```
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: method-route
spec:
  http:
    - name: method
      match:
        paths:
          - /
        exprs:
          - subject:
              scope: Query
              name: id
            op: Equal
            value: "2143"
      backends:
      - serviceName: foo
        servicePort: 80
```

subject.scope支持`Header` `Query` `Cookie` `Path` 
subject.op支持`Equal` `NotEqual` `GreaterThan` `LessThan` `In` `NotIn` `RegexMatch` `RegexNotMatch` `RegexMatchCaseInsensitive` `RegexNotMatchCaseInsensitive`
