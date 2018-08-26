---
layout: post
title:  "[Kubernetes] Cluster Architecture"
description: "本篇文章介紹 Kubernetes Cluster Architecture 以及內部元件的功能"
date: 2018-08-26 17:35:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - CKA
---

本篇文章從 high level 的角度介紹 Kubernetes Cluster Architecture 以及內部元件功能的大略說明。


Kubernetes 的前身 - Google Borg
==============================

Kubernetes 源自於 Google 內部的 Borg，因此看看原生的 Borg 會有助於從大方向了解 Kubernets 架構中的各個 component 是如何協同運作的，以下是 Borg 的系統架構：

![Google Borg System Architecture](/blog/images/kubernetes/cka-k8s-borg-architecture.png)

從上面的架構圖可以看出幾個重點：

### BorgMaster

BorgMaster 是整個系統的大腦，用來負責與系統中的不同元件進行溝通，確保系統可以正常運作。


### Borglet

這是在每個 host 中負責管理 container 生命周期的元件，接收來自 BorgMaster 的命令並進行相對應的操作。


### Scheduer

負責進行任務的調度，根據不同應用的需求，將 workload 調度到不同的機器去執行。


### Paxos (persistent store)

由於整體系統在設計上就是預期會有某些 node 在某些時候會發生故障，因此當要儲存系統運作狀態時，就不會考慮儲存在自身的儲存空間上；因此 Paxos 就是在整個系統中負責儲存系統運作狀態的地方。

### borgcfg

用來操作 Borg 系統的 command line tool


Kubernets 架構
=============

Kuberntes 的架構設計上也極為類似 Borg，如下圖所示：

![Kuberntes Architecture](/blog/images/kubernetes/CKA-CoreConcept-k8s-cluster-architecture-high-level.jpg)

在把上面複雜的架構圖抽象化一點：

![Abstract Kuberntes Architecture](/blog/images/kubernetes/kubernetes-whole-arch.png)

從上圖可以看出以下結論：

1. 不論是透過 CLI or UI，都只能透過呼叫 API 的方式與 Kuberntes 溝通

2. master node 收到 API request 後，針對使用者需求對後面的 worker node 進行指定的工作

3. worker node 若是要佈署 pod，則需要從 image registry 中拉取特定的 image 來進行

接著以下用不同 node type 的角度來切入進行細部檢視：

## Master Node

![Abstract Kuberntes Architecture](/blog/images/kubernetes/kubernetes-master-arch.png)

在 master node 上有幾個相當重要的 component 分別是：

### API server

此為所有資源操作的唯一入口，並負責 Authentication、Authorization、Access Control、API registration & discovery。

### Scheduler

負責進行任務的調度，根據不同應用的需求，將 workload 調度到不同的機器去執行。

### Controller Manager

負責維護 cluster 的狀態，包含 failure detection、auto scale in/out、rolling update .... 等等。

### etcd

負責保存整個 cluster 的運行狀態

> 若 cluster 的 node 數量龐大，etcd 也有可能會被獨立於 master 之外來單獨安裝。


## [Worker Node](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md#the-kubernetes-node)

在 Kuberntes 中，worker node 也稱為 **minion**，可以是實體機或是 VM，以下是 worker node 的架構圖：

![Abstract Kuberntes Architecture](/blog/images/kubernetes/kubernetes-node-arch.png)

### Container Runtime

負責 container image 的管理並透過 [CRI(Container Runtime Interface)](https://github.com/kubernetes/community/blob/master/contributors/devel/container-runtime-interface.md) 來管理 pod 的運行。 

大部份時候會以 Docker 作為 container runtime，但其實可以作為 container runtime 的除了 Docker 之外，還有以下幾個選項：https://github.com/rkt/rkt

- [rkt - the pod-native container engine](https://github.com/rkt/rkt) 

- [CRI-O - OCI-based implementation of Kubernetes Container Runtime Interface](https://github.com/kubernetes-incubator/cri-o)

- [Frakti - The hypervisor-based container runtime for Kubernetes](https://github.com/kubernetes/frakti)


### kubelet

負責接收來自 API server 的命令、維護 container 的生命周期，並負責 CSI(Container Storage Interface) & CNI(Container Network Interface) 的管理。

### kube-proxy

在 Kuberntes 中，**[service](https://kubernetes.io/docs/concepts/services-networking/service/)** 將多個 pod 抽象化成一個單一群組，並為此群組提供一個 virtual IP，並提供其他 pod or service 進行存取。

而 kube-proxy 就是負責在 host 中針對 service 與 pod 的配置設定，產生相對應的 iptables rule，讓送到 service virtual IP 的流量可以正確的導向 service 後端的 pod 中。

因此總結來說，kube-proxy 就是負責為 service 提供 cluster 內部的 service discovery & load balance


Add-Ons
=======

除了上面在 master & worker node 上的核心元件外，還有一些 add-on 可以讓 Kuberntes 運作的更好，例如：

- **DNS**：目前以 [CoreDNS](https://github.com/coredns/coredns) 為主，以前則是 [kube-dns](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns/kube-dns)

- **Ingress controller**：負責讓外部的流量可以正確的導到 Kubernetes 內部的 service

- **Resource Monitoring**：原先是由 [Heapster](https://github.com/kubernetes/heapster) 來負責，但由於 [Prometheus](https://prometheus.io/) 從 CNCF 變成正式專案後，在 resource monitoring 這個部份開始推薦使用 Prometheus 來實現

- **Dashboard**：為了提供使用者更好的體驗，除了 [Kuberntes 官方正式維護的 Dashboard](https://github.com/kubernetes/dashboard/) 之外，也可以考慮其他的選項，例如：[Cockpit](https://cockpit-project.org/guide/latest/feature-kubernetes.html) or [Rancher](https://rancher.com/)(透過 import Kuberntes Clusters 的功能)


當然上面的 add-on 並不是全部，詳細的 add-on 清單可以到 [Kubernetes 的專案](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons)中去查詢。


References
==========

- [Kubernetes Architecture - Kubernetes](https://kubernetes.io/docs/concepts/architecture/)

- [Kubernetes Design and Architecture](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md)

- [Kubernetes架构 · Kubernetes Handbook - Kubernetes中文指南/云原生应用架构实践手册 by Jimmy Song(宋净超)](https://jimmysong.io/kubernetes-handbook/concepts/)

- [架构原理 · Kubernetes Handbook](https://kubernetes.feisky.xyz/zh/architecture/architecture.html)

- [Kubernetes指南 - 架构原理 · Kubernetes Handbook](https://kubernetes.feisky.xyz/zh/architecture/architecture.html)