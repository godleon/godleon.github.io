---
layout: post
title:  "[Kubernetes] 如何取得合法可用的權限，讓 pod 與 API server 溝通"
description: "本篇文章的主題在介紹在 Kubernetes 中，若要開發一支程式並與 API server 溝通，如何設定並取得合法可用的權限"
date: 2019-05-22 14:30:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Security
---


Preface
=======

最近同事在修改 [telegraf plugin](https://github.com/influxdata/telegraf/blob/release-1.10/plugins/inputs/kube_inventory/README.md) 要用來取得 Kubernetes cluster level resource status，大概包含以下內容：

- daemonsets

- deployments

- nodes

- persistentvolumes

- persistentvolumeclaims

- pods (containers)

- statefulsets

遇到了一些權限上的問題，後來花了點時間解決，趁還沒忘記前作個筆記，或許可以幫助到有需要的人。

> 以下都是在 RBAC 的環境下所作的操作 & 說明


如何取得合法 & 可用的權限 ?
=======================

我們要解決的問題是：

> 我寫了一支程式，要取得 Kubernetes cluster level 的資訊，要如何取得合法權限 ?

首先，這些資訊要向 API server 取得，因此**要有合法的 token** 才可以向 API server 問到所需要的資訊，但另外一個問題來了：

> token 在哪? 要如何取得?

在 Kubernetes 中，token 是存在於 secret 中，而 secret 是存在於 service account 中，但 service account 有多少權限，則是透過 `Role`, `ClusterRole`, `RoleBinding`, `ClusterRoleBinding` .... 等 resource 來定義，相互關係可以參考下圖：

![Kubernetes - Service Account, Role, RoleBinding, ClusterRole, ClusterRoleBinding](/blog/images/kubernetes/k8s-serviceaccount-permission.png)

以 [telegraf Kube Inventory Plugin][telegraf Kube Inventory Plugin] 的說明為例，需要以下的權限：

```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: influx:cluster:viewer
  labels:
    rbac.authorization.k8s.io/aggregate-view-telegraf: "true"
rules:
- apiGroups: [""]
  resources: ["persistentvolumes","nodes"]
  verbs: ["get","list"]

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: influx:telegraf
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-view-telegraf: "true"
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-view: "true"
rules: [] # Rules are automatically filled in by the controller manager.
```

上面的設定是透過 `aggregationRule` 的方式，讓最上面所定義的 `influx:cluster:viewer` 可以被 reuse；有了權限的設定後，我們要把它指定給某個 service account：

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: telegraf

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: influx:telegraf:viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: influx:telegraf
subjects:
- kind: ServiceAccount
  name: telegraf
```

此時 service account `telegraf` 中已經有合法 & 可用的 token 可以用來向 API server 取得我們所需要的資訊。



如何使用 Service Account 中的 Token?
==================================

目前知道有一個名稱為 `telegraf` 的 service account，裏面有個可用的 token，但是要怎麼使用這個 token 呢?

答案是要在 pod definition 中加上 `serviceAccountName` 的設定，例如：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: "telegraf"
  labels:
    k8s-app: "telegraf"
spec:
  selector:
  ...
  template:
    ...
    spec:
      serviceAccountName: "telegraf"
      containers:
      ...
```

如此一來就是指定透過上面這個 DaemonSet 產生的 pod，會使用所指定的 service account，但隨之而來的問題是：

> 指定了 service account 後，token 在哪裡?

在 k8s 中，當使用者在 pod 中指定要使用特定的 service account，k8s 會自動將 service account 掛載到該 pod 中，而 service account 中的 token 就會存在於該 pod 的 `/var/run/secrets/kubernetes.io/serviceaccount/token` 這個位置，因此程式中只要指定使用上面這個位置的 token 內容，就可以跟 API server 溝通，並取得所需要的資料了!

所以要開發運行在 k8s 的程式且有與 API server 溝通的需求，建議將 `/var/run/secrets/kubernetes.io/serviceaccount/token` 這個位置設定為 bearer token 的預設路徑，如此一來，使用者只要有設定好正確的 service account 並將其指定到 pod definition 中，程式理當來說就可以與 API server 溝通了!



Reference
=========

- [了解 Kubernetes 中的授權機制 | 小信豬的原始部落](https://godleon.github.io/blog/Kubernetes/k8s-API-Authorization/)

- [Kubernetes权限管理之RBAC - OrcHome](http://orchome.com/1308)

- [telegraf Kube Inventory Plugin][telegraf Kube Inventory Plugin]



[telegraf Kube Inventory Plugin]: https://github.com/influxdata/telegraf/blob/release-1.10/plugins/inputs/kube_inventory/README.md "telegraf Kube Inventory Plugin"
