---
title: Kubernetes ingress
layout: 2017/sheet
prism_languages: [bash,yaml]
weight: -3
tags: [Featured]
updated: 2017-11-19
category: Kubernetes
---
{: .-there-column}
## Kubernetes traefik

# K8s Ingress基本概念

​ [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) & [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 都会各自拥有一个 IP address 供读取，但这些 IP 仅在 K8s cluster 內部才有办法读取的到，但若要在 K8s cluster 上提供对外服务呢?

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

{: .-there-column}
#  示例
#### [x] Traffik 多https证书支持, 
后期配置tls证书， 此证书只允许具有相同namespace ingress使用

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik-ui.minikube
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
  tls:
    - secretName: traefik-ui-tls-cert
```
在ingress 创建的空间内创建secert

```
kubectl -n kube-system create secret tls traefik-ui-tls-cert --key=tls.key --cert=tls.crt
```
#### 定义后端的分发策略

>  这里支持多种负载均衡方法：
>
>  - `wrr`: 加权轮询
>  - `drr`: 动态轮询: 这会为表现比其他服务器好的服务器增加权重。当服务器表现有变化的时，它也会会退到正常权重。

定义在service 资源中， 不能定义在ingress资源中

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx
  annotations:
traefik.ingress.kubernetes.io/load-balancer-method: drr
spec:
  selector:
app: nginx
  ports:
- protocol: TCP
  port: 80
  targetPort: 80
```

#### session 粘滞

> 所有的负载平衡器都支持粘滞会话(sticky sessions)。当粘滞会话被开启时，会有一个名称叫做`_TRAEFIK_BACKEND`的cookie在请求被初始化时被设置在请求初始化时。在随后的请求中，客户端会被直接转发到这个cookie中存储的后端（当然它要是健康可用的），如果这个后端不可用，将会指定一个新的后端。
>
> 开启的方法为添加`traefik.ingress.kubernetes.io/affinity: "true"` 的annotations

定义在service 资源中， 不能定义在ingress资源中

```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx
  annotations:
traefik.ingress.kubernetes.io/affinity: "true"
traefik.ingress.kubernetes.io/load-balancer-method: drr
spec:
  selector:
app: nginx
  ports:
- protocol: TCP
  port: 80
  targetPort: 80
```

请求header 如下

> ```bash
> ➜  ~ curl -H "Host: ngx09.gxd88.cn" http://internal/api/ -v
> GET /api/ HTTP/1.1
> Host: ngx09.gxd88.cn
> User-Agent: curl/7.51.0
> Accept: /
> < HTTP/1.1 200 OK
> < Accept-Ranges: bytes
> < Content-Length: 612
> < Content-Type: text/html
> < Date: Sun, 05 Aug 2018 04:07:11 GMT
> < Etag: "54999765-264"
> < Last-Modified: Tue, 23 Dec 2014 16:25:09 GMT
> < Server: nginx/1.7.9
> < Set-Cookie: _c43d4=http://172.20.0.162:80; Path=/.   cookie 记录后端服务IP
> < Vary: Accept-Encoding
> ```



####  自定义日志格式, 添加字段

> https://docs.traefik.io/configuration/logs/ 
>
> 日志文件添加请求类型： 是内部访问还是外部访问
>
> 请求所属的部门
>
> 

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx0901
  namespace: default
  labels: 
traffic-type: internal
  annotations:
traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip
ingress.kubernetes.io/custom-request-headers: traffic-type:internal||team:devops
spec:
  rules:
  - host: ngx09.gxd88.cn
http:
  paths:
  - path: /api 
    backend:
      serviceName: nginx
      servicePort: 80
```


##### **rewrite 相关实现**

> rewrite 实现使用`traefik.ingress.kubernetes.io/rewrite-target `annotations , 具体实现如下， 实现的效果为 `/api/(.*)` 转发为`/api/$1` 其中`(.*)`和 `$1`为自动添加， 不支持绝对的路径匹配（如访问／api）会提示404

- 访问`/api/a/(.*) `的请求rewrite为`/api/b/$1`
- 访问`/api/a/(.*)` 的请求全部转发给后端为`/a/$1`

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx06
  namespace: default
  labels: 
  traffic-type: internal
  annotations:
  traefik.ingress.kubernetes.io/rewrite-target: /api/b

spec:
  rules:
  - host: ngx06.gxd88.net
  http:
    paths:
    - path: /api/a
    backend:
      serviceName: stilton
        servicePort: http
  
```

> 具体的路由规则为`PathPrefix:/stilton;ReplacePathRegex: ^/stilton/(.*) /api/b/$1` 
> `PathPrefix:/api/a;ReplacePathRegex: ^/api/a/(.*) /api/b/$1` 

当前ingress下的所有path 都会添加相同的ReplacePathRegex

- `/api/a`转发为`/a` 删除路径前缀

  > 使用`traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip` annotations
  >
  > 或者使用`traefik.ingress.kubernetes.io/rewrite-target` annotations， 但是不支持路径本身
  >
  > 或者使用`traefik.ingress.kubernetes.io/request-modifier: AddPrefix: /api`, path只监听`/a`， 路由规则为`PathPrefixStrip /api`

  ```yaml
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: nginx09
    namespace: default
    labels: 
      traffic-type: internal
    annotations:
      traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip
  spec:
    rules:
    - host: ngx09.gxd88.cn
      http:
        paths:
        - path: /api 
          backend:
            serviceName: nginx
            servicePort: 80
  ```

  > 请求方法 
  >
  > ```bash
  > curl -H "Host: ngx09.gxd88.cn" http://internal/api/asd
  > ```
  >
  > 请求到后端为
  >
  > ```bash
  > http://backend/asd
  > ```
  > 最终实现当前域名下的所有请求都会去掉/api




#### **http 强制跳转https**

```yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx05
  namespace: default
  labels: 
traffic-type: internal
  annotations:
traefik.ingress.kubernetes.io/redirect-entry-point: https
spec:
  rules:
  - host: ngx05.gxd88.cn
http:
  paths:
  - backend:
  serviceName: nginx
  servicePort: 80
```



#### **相同域名下托管多个站点**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
name: nginx04
  labels: 
    traffic-type: internal
spec:
  rules:
  - host: ngx04.gxd88.cn
    http:
      paths:
      - path: /stilton
        backend:
          serviceName: stilton
          servicePort: http
      - path: /cheddar
        backend:
          serviceName: cheddar
          servicePort: http
      - path: /wensleydale
        backend:
          serviceName: wensleydale
          servicePort: http
```