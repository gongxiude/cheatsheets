---
title: kubectl network policy
layout: 2017/sheet
prism_languages: [go, bash]
weight: -3
tags: [Featured]
category: Kubernetes
updated: 2017-11-19
---


{: .-one-column}
## network policy 

`network policy`顾名思义就是对pod进行网络策略控制。 k8s本身并不支持，因为k8s有许多种网络的实现方式，企业内部可以使用简单的flannel、weave、kube-router等，适合公有云的方案则有calico等。不同的网络实现原理（vethpair、bridge、macvlan等）并不能统一地支持network policy。



使用`network policy`资源可以配置pod的网络，networkPolicy是namespace scoped的，他只能影响某个namespace下的pod的网络出入站规则。

- metadata 描述信息
- podSelector pod选择器，选定的pod所有的出入站流量要遵循本networkpolicy的约束
- policyTypes 策略类型。包括了Ingress和Egress，默认情况下一个policyTypes的值一定会包含Ingress，当有egress规则时，policyTypes的值中会包含Egress
- ingress 入站
- egress 出站



**example**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```



该例子的效果如下： 

1. `default` namespace下label包含`role=db`的pod，都会被隔绝，他们只能建立“满足networkPolicy的ingress和egress描述的连接”。即2-5点： 
2. 所有属于`172.17.0.0/16`网段的IP，除了`172.17.1.0/24`中的ip，其他的都可以与上述pod的6379端口建立tcp连接。 
3. 所有包含label：`project=myproject`的namespace中的pod可以与上述pod的6379端口建立tcp连接；
4. 所有`default` namespace下的label包含`role=frontend`的pod可以与上述pod的6379端口建立tcp连接；
5. 允许上述pod访问网段为`10.0.0.0/24`的目的IP的5978端口。

 


## namespace 隔离

### 描述
默认情况下，所有 Pod 之间是全通的。每个 Namespace 可以配置独立的网络策略，来隔离 Pod 之间的流量。

{: .-three-column}
### deny all ingress traffic

同namespace的pod，入站规则为全部禁止

```bash 
kubectl create -f deny-all.yaml -n net-policy-test
```

资源文件

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### allow all ingress traffic

namespace的pod，入站规则为全部开放

资源文件
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  ingress:
  - {}
```

### deny all egress traffic 
同namespace的pod，出站规则为全部禁止

```yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### deny all egress traffic 

同namespace的pod，出站规则为全部开放

```yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```

### deny all ingress and all egress traffic
同namespace的pod， 入站规则和出站规则全部禁止

```yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```


## pod 隔离

### 描述
通过使用标签选择器（包括 namespaceSelector 和 podSelector）来控制 Pod 之间的流量。比如下面的 Network Policy


### example

```yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: tcp
      port: 6379
```

1. 允许 default namespace 中带有 role=frontend 标签的 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口
2. 允许带有 project=myprojects 标签的 namespace 中所有 Pod 访问 default namespace 中带有 role=db 标签 Pod 的 6379 端口

### 禁止访问指定服务 
```
$ kubectl run web --image=nginx --labels app=web,env=prod --expose --port 80
```
** 网络策略 ** 
```yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-pod-ingress
  namespace: net-policy-test
spec:
  podSelector: 
    matchLabels: 
      app: web
      env: prod
  policyTypes:
  - Ingress
```

### 只允许指定 Pod 访问服务 

允许net-policy-test namespace下的app: debug-tools的pod访问app: web，app: web的pod

```yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-pod-access-pod
  namespace: net-policy-test
spec:
  podSelector: 
    matchLabels: 
      app: web
      env: prod
  policyTypes:
  - Ingress
  ingress: 
    - from:
        - podSelector: 
            matchLabels: 
              app: debug-tools
```

### 只允许指定 namespace 访问服务

创建的namespace时需要制定label
```yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: net-policy-test
  labels:
    name: net-policy-test
```

网络策略
```yaml 
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-namespace-access-pod
  namespace: net-policy-test
spec:
  podSelector:
    matchLabels:
      app: web
      env: prod
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: net-policy-test01
```

### 禁止不同的namespace之间相互访问

```yaml 
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-other-namespace-access-pod
  namespace: net-policy-test
spec:
  podSelector: 
    matchLabels: 

  policyTypes:
  - Ingress
  ingress: 
    - from: 
      - podSelector: {}
```



## 参考

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [kubernetes network policy学习笔记](https://segmentfault.com/a/1190000012692009)