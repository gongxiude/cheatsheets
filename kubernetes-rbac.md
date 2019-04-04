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

