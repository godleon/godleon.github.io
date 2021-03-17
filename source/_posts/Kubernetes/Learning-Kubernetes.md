---
layout: post
title:  "Kubernetes 學習筆記"
description: "This article includes all the study notes when learning Kubernetes"
date: 2017-06-29 15:00:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
---

此篇文章為研究 Kubernetes 時所留下的學習筆記索引

Concepts
========

## kubeconfig

- [如何設定 kubeconfig 與 Kubernetes cluster 互動](https://github.com/godleon/learning_kubernetes/blob/master/concept/howto_configure_kubeconfig.md)

基本上，kubeconfig 內容的組成是由以下幾個部份組成：

- **clusters**：這裡定義了一個或多個 cluster control plane 的 name、endpoint、認證用資訊 ... 等等

- **users**：這裡定義了使用者的資訊

- **contexts**：此部份就是 user & cluster 的組合，並且會給一個 name

- **current-context**：指定目前預設使用的 context

![kubeconfig structure](/blog/images/kubernetes/k8s_kubeconfig-structure.png)

透過上面的組合，就可以在同一個 kubeconfig 中定義多個 cluster，並根據管理需求隨時切換 current-context 指定的 cluster

以下是常用指令：

```bash
# 檢視目前 kuebconfig 的內容
$ kubectl config view 可以

# 切換 current context
$ kubeconfig config use-context [CONTEXT_NAME]
```



## Overview & Components

- [組成元件概觀](https://github.com/godleon/learning_kubernetes/blob/master/concept/component_overview.md)

- [Kubernetes Overview](https://github.com/godleon/learning_kubernetes/blob/master/concept/overview.md)

## Networking

- [Service](https://github.com/godleon/learning_kubernetes/blob/master/concept/network/service.md)

- [Ingress](https://github.com/godleon/learning_kubernetes/blob/master/concept/network/ingress.md)

## Storage

- [Volume, PersistentVolume & PersistentVolumeClaim](https://github.com/godleon/learning_kubernetes/blob/master/concept/storage/volume.md)


-------------------------


Operating
=========

- [kubectl Cheat Sheet - Kubernetes](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

- [Kubernetes 基本操作](https://github.com/godleon/learning_kubernetes/blob/master/operation/basic.md)

## Persistent Volume

- [使用 Persistent Volume - 以 NFS 為例](https://github.com/godleon/learning_kubernetes/blob/master/operation/use_PersistentVolume_NFS.md)



學習資源
======

- [Kubernetes(K8S)中文文档_Kubernetes中文社区](http://docs.kubernetes.org.cn/)