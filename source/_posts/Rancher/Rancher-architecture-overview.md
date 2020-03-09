---
layout: post
title: "Rancher Architecture Overview"
description: "本篇文章簡單介紹 Rancher 的架構 & 內部元件"
date: 2020-03-10 06:30:00
comments: true
published: true
tags: 
  - Rancher
  - Kubernetes
categories: 
  - Rancher
---


# Rancher Server Architecture

![Rancher Architecture Overview](/blog/images/rancher/rancher-arch-overview.png)

從上面的示意圖可以看出幾點：

- Rancher server 可以同時管理多個下游的 k8s cluster (可以是透過 RKE 佈署的 k8s，也可以是 AWS EKS)

- 使用者可以透過 Authentication Proxy，對多個下游 k8s cluster 進行管理

- Rancher 本身也是一個小型的 k8s cluster，可以安裝在單一節點上(測試環境)或是帶有 HA 的 k8s cluster(生產環境) 中

- Rancher server 所運行的節點務必與所管理的 k8s cluster 分開


# Rancher server 如何與下游的 k8s cluster 通訊

以下是個簡單的示意圖：

![How Rancher Communicate](/blog/images/rancher/how-rancher-communicate.png)

上圖中有幾大重要的部份：

## [Authentication Proxy](https://rancher.com/docs/rancher/v2.x/en/overview/architecture/#1-the-authentication-proxy)

- 透過 Rancher server 進行 request 的使用者，會經過 Authentication Proxy 的認證後，由 Rancher server 將 request 轉到要管理的 cluster 上。

- 可以整合 GitHub, ActiveDirectory ...等服務進行驗證

- Rancher server 是透過 service account 來作為管理下游 k8s cluster 的識別

- 預設 Rancher server 會提供一份完整的 kubeconfig 檔案作為透過 Authentication Proxy 來管理下游 cluster 之用

## Cluster Controller

每個由 Rancher server 管理的 k8s cluster 都會有一對 Cluster Agent & Controller 來搭配負責進行管理，而 Cluster Controller 負責以下工作：

- 監控下游 k8s cluster 的變更

- 確保下游的 k8s cluster 有達到 desired state

- 將存取權限設定套用到下游的 cluster & project

- 與 RKE or GKE .... 等服務搭配，進行 provision 的工作

## Cluster Agent (cattle-cluster-agent)

預設每個下游的 k8s cluster 都會有一個 agent，並建立一條 tunnel 連回 Rancher server；若沒有，Cluster Controller 會直接連到 **Node Agent**。

而 Cluster Agent 負責以下工作：

- 負責連到由 Rancher 啟動的 k8s cluster 中的 API server

- 為每個 cluster 管理 workload，包含 pod, deployment ... 等等

- 將 role & binding .... 等全域設定套用到 cluster 中

- 回報 event、stat、node info, health ... 等資訊給 Rancher server


## Node Agent (cattle-node-agent)

- 若 cluster agent 無法使用，cluster 中的某一個 node agnet 就會負責與 Rancher server 通訊。

- node agnet 是以 DaemonSet 的形式安裝在由 Rancher 佈署的下游 k8s cluster 中

- 負責進行 node level 的相關操作，例如：更新 k8s 版本、還原 etcd snapshot ... 等等


## Authorized Cluster Endpoint

透過 RKE 安裝的 k8s cluster 中會包含一個名稱為 `kube-api-auth` 的 pod，允許使用者可以不透過 Authentication Proxy 與下游的 k8s cluster 直接進行通訊，主要目的有兩個：

1. 避免 Rancher server 發生故障倒至於無法存取 cluster

2. 若 Rancher server 與下游的 k8s cluster 距離很遠，可以透過此 endpoint 降低延遲

> Rancher server 可以產生讓使用者可以直連下游 k8s cluster 的 kubeconfig (建議此類的 kubeconfig 可以先匯出)


Reference
=========

- [Architecture - Rancher](https://rancher.com/docs/rancher/v2.x/en/overview/architecture/)