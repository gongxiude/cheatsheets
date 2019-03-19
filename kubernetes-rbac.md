---
title: kubernetes-rbac
layout: 2017/sheet
prism_languages: [go, bash]
weight: -3
tags: [Featured]
category: Kubernetes
updated: 2019-03-19
---
{: .-one-column}
## RBAC 基于角色的访问控制

基于角色的访问控制（Role-Based Access Control, 即”RBAC”）使用”rbac.authorization.k8s.io” API Group实现授权决策，允许管理员通过Kubernetes API动态配置策略。

截至Kubernetes 1.6，RBAC模式处于beta版本。

要启用RBAC，请使用`--authorization-mode=RBAC`启动API Server。

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ name }}
  namespace: {{ namespace }
secrets:
- name: {{secret-name}}
---
apiVersion: v1
kind: Secret
metadata:
  name: {{ secret-name }}
  annotations:
    kubernetes.io/service-account.name: {{ secret-name}
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ name }}-role
rules:
- apiGroups: [""]
  resources: ["pods", "nodes", "replicationcontrollers", "events", "limitranges", "services"]
  verbs: ["get", "delete", "list", "patch", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: {{ name }}-role-binding
roleRef:
  kind: ClusterRole
  name: {{ name }}-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: {{ name }}
  namespace: {{ namespace }
```