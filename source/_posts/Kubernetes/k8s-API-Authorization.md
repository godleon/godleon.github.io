---
layout: post
title:  "了解 Kubernetes 中的授權機制"
description: "本篇文章介紹在 Kubernetes 中存取 API 時的授權機制"
date: 2018-06-19 16:30:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - CKA Security
---


前言
===

在進行授權(Authorization) 之前，一定要先完成認證(Authentication) 的過程，而認證的說明可以參考[上一篇文章](https://godleon.github.io/blog/2018/06/19/Kubernetes/k8s-API-Authentication/)。

k8s 對於 REST API request 的屬性要求是通用的，因此可以與企業中 or 公有雲平台中現存的管理控制系統進行整合。

而在 k8s 中，負責對 API request 進行授權的是 API server，藉由檢查所有的 **request attribute** 並查詢所有的 policy 來決定是否允許或拒絕 API request 的存取。

> 預設情況下所有權限都是預設關閉的

k8s 支援同時多個授權機制，並且為循序式的進行檢查，若有任何一個授權機制通過檢查，就會馬上允許存取，若是沒有通過任何的授權機制檢查，此 API request 就會被拒絕存取；而被拒絕存取的 API request 都會收到 HTTP 403 error。



名詞釋疑
======

## NameSpace

用來進行資源的虛擬隔離之用，在 k8s 剛建立好的初期會有預設的 `default` & `kube-system` 兩個 namespace


## User Account & Service Account

User Account & Service Account 是用來區別作用的範圍，但會有以下的差別：

1. User Account 是用來辨識使用者，而 Service Account 則是用來辨識 k8s 中的各種服務

2. User Account 對應人的身份，因此是跨 namespace；而 Service Account 對應的是 service 的身份，因此是與 namespace 有相關



Request Attributes
==================

上面有提到，API server 會檢查 **API request attribute** 來決定是否進行授權存取，在 k8s 有以下 API request attributes：

- user, group, extra(額外的 key/value 資訊)

- API、Request Path(/api, /healthz), API request verb(get, list, create, update, patch, watch ... etc)

- HTTP request verb(get, post, put, delete)

- Resource, Subresource (存取 resource 時需要的資訊)

- Namespace

- [API group](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups) (存取 resource 時需要的資訊)



Authorization Modules
=====================

目前 k8s 支援以下幾種 Authorization Modules：

## [Node](https://kubernetes.io/docs/reference/access-authn-authz/node/)

此模組是為了授權在每一個 node 上的 kubelet 所發出的 API request 所設計出來的，讓 kubelet 的 API request 可進行特定的權限控制；例如對於以下的 resource 的讀取的權限：

- services

- endpoints

- nodes

- pods

- secrets / configmaps / presistent volume (claim)

另外透過 NodeRestriction 的 plugin，可以針對以下 resource 進行寫入：

- node (status)

- pod (status)

- event

> 未來可能還會繼續透過 Node Authorization 來對於 kubelet 進行更細膩的權限控管


## [ABAC (Attribute-based access control)](https://kubernetes.io/docs/reference/access-authn-authz/abac/)

此種方式就是在 master node 上保留一份 policy 文件，指定不同的使用者對於 resource 的存取權限，不彈性也不容易擴充，修改了 policy 文件之後還需要重新啟動 master node。


## [RBAC (Role-based access control)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

RBAC 是目前 k8s 預設會啟動的 authorization module。

在 RBAC API 中定義了 resource target，用來描述使用者以及 resource 之間的權限關係：

- Role：定義在特定 namespace 下的 resource 的存取權限

- RoleBinding： 設定哪些使用者(or service account)與 role 綁定而擁有存取權限

- ClusterRole：定義在整個 k8s cluster 下的 resource 的存取權限

- ClusterRoleBinding：設定哪些使用者(or service account)與 role 綁定而擁有存取權限


接著以下以 role **kubernetes-dashboard-minimal** 做個簡單的示範：

```bash
root@kube-master0:~# kubectl get role --all-namespaces
NAMESPACE     NAME                                             AGE
default       pod-reader                                       4d
kube-public   system:controller:bootstrap-signer               5d
kube-system   extension-apiserver-authentication-reader        5d
kube-system   kubernetes-dashboard-minimal                     5d
..... (略)


root@kube-master0:~# kubectl get rolebinding --all-namespaces
NAMESPACE     NAME                                             AGE
kube-public   system:controller:bootstrap-signer               5d
kube-system   kubernetes-dashboard-minimal                     5d
..... (略)


#
# 檢視 role "kubernetes-dashboard-minimal" 中有那些 resource 的存取權限
#
root@kube-master0:~# kubectl describe role/kubernetes-dashboard-minimal -n kube-system
Name:         kubernetes-dashboard-minimal
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"rbac.authorization.k8s.io/v1","kind":"Role","metadata":{"annotations":{},"name":"kubernetes-dashboard-minimal","namespace":"kube-system"...
PolicyRule:
  Resources       Non-Resource URLs  Resource Names                     Verbs
  ---------       -----------------  --------------                     -----
  configmaps      []                 [kubernetes-dashboard-settings]    [get update]
  configmaps      []                 []                                 [create]
  secrets         []                 [kubernetes-dashboard-certs]       [get update delete]
  secrets         []                 [kubernetes-dashboard-key-holder]  [get update delete]
  secrets         []                 []                                 [create]
  services        []                 [heapster]                         [proxy]
  services/proxy  []                 [heapster]                         [get]
  services/proxy  []                 [http:heapster:]                   [get]
  services/proxy  []                 [https:heapster:]                  [get]

#
# 檢視目前與 role "kubernetes-dashboard-minimal" 相關連的使用者(名稱為 "kubernetes-dashboard" 的 service account)
#
root@kube-master0:~# kubectl describe rolebinding/kubernetes-dashboard-minimal -n kube-system
Name:         kubernetes-dashboard-minimal
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"rbac.authorization.k8s.io/v1","kind":"RoleBinding","metadata":{"annotations":{},"name":"kubernetes-dashboard-minimal","namespace":"kube-...
Role:
  Kind:  Role
  Name:  kubernetes-dashboard-minimal
Subjects:
  Kind            Name                  Namespace
  ----            ----                  ---------
  ServiceAccount  kubernetes-dashboard  kube-system
```

接著以 clusterole **system:controller:deployment-controller** 做示範：

```bash
#
# 檢視目前 k8s cluster 中的 ClusteRole 資訊
#
root@kube-master0:~# kubectl get clusterrole
NAME                                                                   AGE
admin                                                                  5d
calico-node                                                            5d
cluster-admin                                                          5d
..... (略)
system:controller:deployment-controller                                5d
..... (略)


#
# 檢視 "system:controller:deployment-controller" ClusteRole 的 resource 存取權限
#
root@kube-master0:~# kubectl describe clusterrole/system:controller:deployment-controller
Name:         system:controller:deployment-controller
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate=true
PolicyRule:
  Resources                          Non-Resource URLs  Resource Names  Verbs
  ---------                          -----------------  --------------  -----
  events                             []                 []              [create patch update]
  pods                               []                 []              [get list update watch]
  deployments.apps                   []                 []              [get list update watch]
  deployments.apps/finalizers        []                 []              [update]
  deployments.apps/status            []                 []              [update]
  replicasets.apps                   []                 []              [create delete get list patch update watch]
  deployments.extensions             []                 []              [get list update watch]
  deployments.extensions/finalizers  []                 []              [update]
  deployments.extensions/status      []                 []              [update]
  replicasets.extensions             []                 []              [create delete get list patch update watch]


#
# 檢視目前 k8s cluster 中的 ClusteRoleBinding 資訊
#
root@kube-master0:~# kubectl get clusterrolebinding
NAME                                                   AGE
..... (略)
system:controller:deployment-controller                5d
..... (略)


#
# 檢視目前 k8s cluster 中與 "system:controller:deployment-controller" ClusteRoleBinding 連結的使用者(or service account)
#
root@kube-master0:~# kubectl describe clusterrolebinding/system:controller:deployment-controller
Name:         system:controller:deployment-controller
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate=true
Role:
  Kind:  ClusterRole
  Name:  system:controller:deployment-controller
Subjects:
  Kind            Name                   Namespace
  ----            ----                   ---------
  ServiceAccount  deployment-controller  kube-system
```

API server 已經預先建立了一系列的 ClusterRole & ClusteRoleBinding，名稱都以 **system** 開頭，並涵蓋了維持系統正常服務所需要的所有權限，因此整個 k8s cluster 都可以在 RBAC 的基礎下進行權限的管理與授權。

> 這也代表著隨意對 system 開頭的 ClusteRole 修改，可能會造成系統無法正常運作。


## [Webhook](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)

此模式是管理者在外部提供 HTTPS 授權服務，並設定 API server 透過與外部服務互動的方式進行授權。





檢查 API 存取權限
===============

kubectl 提供了 `auth can-i` 命令，可用來查詢當前的使用者有沒有特定 resource 的存取權限：

```bash
#
# 檢查有無在 "dev" namespace 中建立 deployment 的權限
#
root@kube-master0:~# kubectl auth can-i create deployments --namespace dev
yes

#
# 檢查有無在 "prod" namespace 中建立 deployment 的權限
#
root@kube-master0:~# kubectl auth can-i create deployments --namespace prod
yes

#
# 檢查有無在 "dev" namespace 中以 dave 的身份來列出 secret 的權限
#
root@kube-master0:~# kubectl auth can-i list secrets --namespace dev --as dave
yes
```


References
==========

- [Authorization Overview - Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)

- [Kubernetes-- 漫谈kubernetes 中的认证 & 授权 & 准入机制 | Solar](https://zhangchenchen.github.io/2017/08/17/kubernetes-authentication-authorization-admission-control/)

- [kubernetes 權限管理 – Cizixs Writes Here](http://cizixs.com/2017/06/16/kubernetes-authentication-and-authorization)

- [https://k8smeetup.github.io/docs/admin/authorization/rbac/](https://k8smeetup.github.io/docs/admin/authorization/rbac/)
