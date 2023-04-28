---
title: kubectl-output
layout: 2017/sheet
prism_languages: [go, bash]
weight: -3
tags: [Featured]
category: Kubernetes
updated: 2017-11-19
---

## 基本命令

{: .-three-column}

### 获取Node的IP地址地址

{:.-prime}

#### 单Node实利的IP地址


```bash
$ kubectl get node <node_name> -o=jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'
```


## 参考

- [JSONPath 支持](https://kubernetes.io/zh/docs/reference/kubectl/jsonpath/)