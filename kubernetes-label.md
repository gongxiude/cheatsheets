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
{: .-prime}
| Shortcut       | Description                      |
| ---            | ---                              |
| `department`   | 部门名称                          |
| `edgenode`     | 是否为边缘节点                     |
| `dedicated`    | 节点用途                          |
| `cmratio`      | CPU和内存的比例                    |
| `instance-type`| 实例类型                          |
{: .-shortcuts}

设置参数
| Shortcut       | Description                      |
| ---            | ---                              |
| `department`   | `devops`,`o2o`,                          |
| `edgenode`     | `true`                     |
| `dedicated`    | `online`,`edgenode`,`model`                   |
| `cmratio`      | `o.5`,`0.25`,`0.125`                    |
| `instance-type`| `compute`,`network`                         |
{: .-shortcuts}

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

{: .-there-column}
## affinity 

### 类型
类型包括： 
- `NodeAffinity`,
- `PodAffinity`,
- `PodAntiAffinity`

限制方式： 

- `requiredDuringSchedulingIgnoredDuringExecution` 表示pod必须部署到满足条件的节点上，如果没有满足条件的节点，就不停重试。其中IgnoreDuringExecution表示pod部署之后运行的时候，如果节点标签发生了变化，不再满足pod指定的条件，pod也会继续运行。
- `requiredDuringSchedulingRequiredDuringExecution` 表示pod必须部署到满足条件的节点上，如果没有满足条件的节点，就不停重试。其中RequiredDuringExecution表示pod部署之后运行的时候，如果节点标签发生了变化，不再满足pod指定的条件，则重新选择符合要求的节点。
- `preferredDuringSchedulingIgnoredDuringExecution` 表示优先部署到满足条件的节点上，如果没有满足条件的节点，就忽略这些条件，按照正常逻辑部署。
- `requiredDuringSchedulingRequiredDuringExecution` 表示优先部署到满足条件的节点上，如果没有满足条件的节点，就忽略这些条件，按照正常逻辑部署。其中RequiredDuringExecution表示如果后面节点标签发生了变化，满足了条件，则重新调度到满足条件的节点

软策略和硬策略的区分是有用处的，硬策略适用于 pod 必须运行在某种节点，否则会出现问题的情况，比如集群中节点的架构不同，而运行的服务必须依赖某种架构提供的功能；软策略不同，它适用于满不满足条件都能工作，但是满足条件更好的情况，比如服务最好运行在某个区域，减少网络传输等。这种区分是用户的具体需求决定的，并没有绝对的技术依赖。

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
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
                preference:
                matchExpressions:
                - key: instance-type
                    operator: In
                    values:
                    - compute
            - weight: 5
                preference:
                matchExpressions:
                - key: cmratio
                    operator: In
                    values:
                    - "0.5"
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
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            labelSelector:
            - matchExpressions:
              - key: department
                operator: In
                values:
                - devops
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
                preference:
                matchExpressions:
                - key: instance-type
                    operator: In
                    values:
                    - compute
            - weight: 5
                preference:
                matchExpressions:
                - key: cmratio
                    operator: In
                    values:
                    - "0.5"
      containers:
      - name: nginx
        image: nginx:1.7.9
```
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
              - key: service
                operator: In 
                values: 
                - traefik

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

