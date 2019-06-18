---
title: kubectl label
layout: 2017/sheet
prism_languages: [go, bash]
weight: -3
tags: [Featured]
category: Kubernetes
updated: 2017-11-19
---

{: .-one-column}

## node label规划

| Shortcut       | Description                      |
| ---            | ---                              |
| `department`   | 部门名称                          |
| `edgenode`     | 是否为边缘节点                     |
| `dedicated`    | 节点用途                          |


设置参数

| Shortcut       | Description                      |
| ---            | ---                              |
| `department`   | `devops`,`o2o`,                          |
| `edgenode`     | `true`                     |
| `dedicated`    | `online`,`edgenode`,`model`,`app`                 |

实现分配pod到node的方法，通过node label selector实现约束pod运行到指定节点,有两种方法 `nodeSelector`以及`affinity`

{: .-one-column}
## nodeSelector

`Pod.spec.nodeSelector`是通过kubernetes的label-selector机制进行节点选择，由scheduler调度策略`MatchNodeSelector`进行label匹配，调度pod到目标节点，该匹配规则是强制约束。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-test
  labels:
    k8s-app: nginx
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        department: devops 
      containers:
      - name: nginx
        image: nginx:1.7.9
```

{: .-one-column}
## affinity 

### 类型
类型包括： 
- `NodeAffinity`, node affinity

- `PodAffinity`, pod亲和性（Inter-pod affinity）

- `PodAntiAffinity` 反亲和性（anti-affinity）

限制方式： 

- `requiredDuringSchedulingIgnoredDuringExecution` 表示pod必须部署到满足条件的节点上，如果没有满足条件的节点，就不停重试。其中IgnoreDuringExecution表示pod部署之后运行的时候，如果节点标签发生了变化，不再满足pod指定的条件，pod也会继续运行。

- `preferredDuringSchedulingIgnoredDuringExecution` 表示优先部署到满足条件的节点上，如果没有满足条件的节点，就忽略这些条件，按照正常逻辑部署。

- > 软策略和硬策略的区分是有用处的，硬策略适用于 pod 必须运行在某种节点，否则会出现问题的情况，比如集群中节点的架构不同，而运行的服务必须依赖某种架构提供的功能；软策略不同，它适用于满不满足条件都能工作，但是满足条件更好的情况，比如服务最好运行在某个区域，减少网络传输等。这种区分是用户的具体需求决定的，并没有绝对的技术依赖。

匹配逻辑label

- In: label的值在某个列表中
- NotIn：label的值不在某个列表中
- Exists：某个label存在
- DoesNotExist：某个label不存在
- Gt：label的值大于某个值（字符串比较）
- Lt：label的值小于某个值（字符串比较）
如果nodeAffinity中nodeSelector有多个选项，节点满足任何一个条件即可；如果matchExpressions有多个选项，则节点必须同时满足这些选项才能运行pod 。需要说明的是，node并没有anti-affinity这种东西，因为NotIn和DoesNotExist能提供类似的功能。



### NodeAffinity

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-test
  labels:
    k8s-app: nginx
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - preference:
              matchExpressions:
              - key: dedicated
                operator: In
                values:
                - model
            weight: 1
          - preference:
              matchExpressions:
              - key: dedicated
                operator: In
                values:
                - app
            weight: 5
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: department
                operator: In
                values:
                - devops
      containers:
      - name: nginx
        image: nginx:1.7.9
```
这个 pod 同时定义了`requiredDuringSchedulingIgnoredDuringExecution`和 `preferredDuringSchedulingIgnoredDuringExecution`两种nodeAffinity。第一个要求 pod 运行在特定 devops 的节点上，第二个希望节点最好有对应的 cmratio:0.5和instance-type:compute标签, 根据权重决定了顺序
`NodeSelectorTerms`可以有多个，之间是或的关系，满足任意一个既满足，`MatchExpressions`也可以有多个，他们之间是且的关系 必须都满足
`preferredDuringSchedulingIgnoredDuringExecution`值为列表，根据权重决定顺序`MatchExpressions`值为列表 关系为且，必须都满足


### PodAffinity && PodAntiAffinity
PodAffinit是根据通过已运行在节点上的pod的标签而不是node的标签来决定被调度pod的运行节点，因为pod运行在指定的namespace所以需要自己指定运行pod的namesapce
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
  labels:
    app: pod-affinity-pod
spec:
  containers:
  - name: with-pod-affinity
    image: nginx
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - busybox-pod
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - node-affinity-pod
          topologyKey: kubernetes.io/hostname
```
上面这个例子中的 POD 需要调度到某个指定的主机上，至少有一个节点上运行了这样的 POD：这个 POD 有一个app=busybox-pod的 label。podAntiAffinity则是希望最好不要调度到这样的节点：这个节点上运行了某个 POD，而这个 POD 有app=node-affinity-pod的 label。根据前面两个 POD 的定义，我们可以预见上面这个 POD 应该会被调度到一个busybox-pod被调度的节点上，而node-affinity-pod被调度到了该节点以外的节点

## 污点（Taints）与容忍（tolerations）
对于nodeAffinity无论是硬策略还是软策略方式，都是调度 POD 到预期节点上，而Taints恰好与之相反，如果一个节点标记为 Taints ，除非 POD 也被标识为可以容忍污点节点，否则该 Taints 节点不会被调度pod。

比如用户希望把 Master 节点保留给 Kubernetes 系统组件使用，或者把一组具有特殊资源预留给某些 POD，则污点就很有用了，POD 不会再被调度到 taint 标记过的节点。taint 标记节点举例如下：
```bash
$ kubectl taint nodes kube-node key=value:NoSchedule
node "kube-node" tainted
```
如果仍然希望某个 POD 调度到 taint 节点上，则必须在 Spec 中做出Toleration定义，才能调度到该节点，举例如下：
```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```
effect 共有三个可选项，可按实际需求进行设置：

- NoSchedule：POD 不会被调度到标记为 taints 节点。
- PreferNoSchedule：NoSchedule 的软策略版本。
- NoExecute：该选项意味着一旦 Taint 生效，如该节点内正在运行的 POD 没有对应 Tolerate 设置，会直接被逐出

## example

{:.-three-column}

### traefik 部署

设置label 和 taint, edgenode为特殊属性的节点所以需要设置taint

```bash 
kubectl taint nodes node  edgenode=true:NoSchedule --overwrite 
kubectl  label nodes node department=devops edgenode=true service=traefik 
```

资源文件

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-test
  labels:
    k8s-app: nginx
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: dedicated
                operator: In 
                values: 
                - edgenode

              - key: department
                operator: In
                values:
                - devops

      tolerations: 
      - key: "edgenode"
        operator: "Exists"
        effect: "NoSchedule"

      containers:
      - name: nginx
        image: nginx:1.7.9
```

### 运维应用部署

运维服务部署在固定的几台主机上， 每个主机设置`department`的label， 部署服务时指定该label

设置label

```bash
kubectl  label nodes node department=devops
```

资源文件
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-test
  labels:
    k8s-app: nginx
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity: 
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
                - key: department
                operator: In
                values:
                - devops
      containers:
      - name: nginx
        image: nginx:1.7.9
```

### pod尽可能的分配到多个节点上

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                  - key: app
                    operator: In
                    values:
                    - nginx
                topologyKey: kubernetes.io/hostname

      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

### pod强制分配到不同的node节点上

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                  - key: app
                    operator: In
                    values:
                    - nginx
                topologyKey: kubernetes.io/hostname

      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

### 不同应用就近部署， 如app=nginx 和 app=frontend部署到相同节点

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                  - key: app
                    operator: In
                    values:
                    - nginx
                topologyKey: kubernetes.io/hostname

      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

## 参考

- [Kubernetes 的亲和性调度](https://www.qikqiak.com/post/understand-kubernetes-affinity/)
- [K8S调度之节点亲和性](https://www.cnblogs.com/breezey/p/9101666.html)