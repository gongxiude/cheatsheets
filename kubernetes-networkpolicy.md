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

> PS:   在podSelector为空对象(即值为`{}`)时,则是选择`该命名空间`所有的pod,如果`ingress/engress`为空对象(即值为`{}`)时,则允许所有流量进/出




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
> 没有配置ingress字段， 则表示没有允许的入口， 拒绝所有流量

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
> 配置了一个空的ingress， 表示所有的入口， 允许所有的进入 

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

禁止相同namespace或者其它namespace下的所有pod访问该服务

```
$ kubectl run web --image=nginx --labels app=web,env=prod --expose --port 80
$ kubectl run debug-tools --image=wcr.xxx.net/xxx/debug-tools:v1.0.1  --labels app=debug-tools
```
网络策略 
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

测试步骤： 

10.40.237.91 为pod的IP地址
```bash 
$ ➜  network-policy kubectl exec -it debug-tools -n net-policy-test  ping 10.40.237.91
PING 10.40.237.91 (10.40.237.91) 56(84) bytes of data. 
```

```bash 
➜  network-policy kubectl exec -it debug-tools -n net-policy-test01   ping 10.40.237.91
PING 10.40.237.91 (10.40.237.91) 56(84) bytes of data.
^C
--- 10.40.237.91 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms
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
  podSelector: {}
  policyTypes:
  - Ingress
  ingress: 
    - from: 
      - podSelector: {}
```


### 允许特定IP地址访问该pod

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ip-access-policy
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
      - ipBlock:
          cidr: 10.45.0.0/16 #禁止该IP地址段访问
      - ipBlock:
          cidr: 10.40.0.0/16 #禁止该IP段访问
          except:
          - 10.40.9.136/22   #排除IP地址

```

## 参考

- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [kubernetes network policy学习笔记](https://segmentfault.com/a/1190000012692009)
- [Kubernetes Network Policies](https://feisky.gitbooks.io/kubernetes/concepts/network-policy.html#%E7%A6%81%E6%AD%A2%E8%AE%BF%E9%97%AE%E6%8C%87%E5%AE%9A%E6%9C%8D%E5%8A%A1)