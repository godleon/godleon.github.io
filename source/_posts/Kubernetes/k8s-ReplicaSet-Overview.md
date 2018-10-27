---
layout: post
title:  "[Kubernetes] ReplicaSet 介紹"
description: "本篇文章的主題圍繞在 Kubernetes 中的 ReplicaSet resource object，並對其概念 & 特性作概略的介紹"
date: 2018-09-07 09:00:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - CKA
  - CKA Core Concept
---


本篇文章的主題圍繞在 Kubernetes 中的 ReplicaSet resource object，並對其概念 & 特性作概略的介紹


What is ReplicaSet ?
====================

![ReplicaSet](/blog/images/kubernetes/k8s-replicaset.png)

ReplicaSet 是用來確保在 k8s 中，在資源允許的前提下，指定的 pod 的數量會跟使用者所期望的一致，也就是所謂的 desired status。

而 ReplicaSet 其實是 ReplicationController 的進化版，其中的差別僅在於 ReplicaSet 支援 set-based label selector，而 ReplicationController 僅支援 equality-based label selector。



如何使用 ReplicaSet?
==================

官方建議 ReplicaSet 要搭配 Deployment 一起來使用，原因如下：

- 若是有 [rolling update](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#rolling-update) 的需求 只有在 Deployment 有相關的 kubectl 指令可以協助，單單使用 ReplicaSet 是沒有的

- Deployment 是個更上層的抽象概念，也因此支援了更多好用的 feature，因此官方才會建議不要單獨使用 ReplicaSet，而是使用 Deployment，並將 ReplicaSet 的資訊設定到 Deployment 的 spec 中



ReplicaSet 定義說明
=================

## 範例介紹

以下用一個簡單的範例來介紹 ReplicaSet 的結構定義：

```yaml
# apiVersion, kind, metadata 是必備欄位
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  # replicaset 也可以定義 label
  # 一般會與 .spec.template.metadata.labels 設定相同，但不同其實也沒差
  labels:
    app: guestbook
    tier: frontend
# 以下透過 spec 設定 replicaset 的規格 
spec:
  # 要產生幾份副本(沒設定則預設為 1)
  replicas: 3
  # 設定 label selector，用來選擇產生副本用的 pod
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  # .spec.template 是 .spec 中唯一的必要欄位
  # 其實就是 pod template
  template:
    metadata:
      # .spec.template.metadata.labels 必須符合 .spec.selector 中的設定才行
      # 否則 API server 會拒絕產生此物件
      labels:
        app: guestbook
        tier: frontend
    spec:
      # 所有 .spec.template.spec.restartPolicy 的設定，僅能設定為 Always (同時也是預設值)
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below.
          # value: env
        ports:
        - containerPort: 80
```

## 使用上需要注意的事情

在設定 ReplicaSet 的時候，有幾點是必須注意的：

- `.spec.template.metadata.labels` 的設定必須符合 `.spec.selector`，否則 API server 會拒絕產生此物件

- 從 v1.9 後，apiVersion 已經從 `apps/v1beta2` 改為 `apps/v1`

- 當 ReplicaSet 物件建立後，不要建立帶有完全相同 label 組合設定的 pod, deployment 或是其他 replicaset，這樣會造成運作上的混淆 (k8s 並不會阻止你這麼做....) 

- 如果有建立多個帶有相同 label selector 的 controller，**刪除**這件事情你就要自己動手了

- `.spec.replicas` 若沒有設定就預設為 **1**

- 如果要運行一個執行完工作就自動終止的 pod，就不要使用 ReplicaSet，而是要改用 [Job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)

- 如果要運行一個 machine level 的功能(例如：monitoring, logging)，確保 pod lifetime 與 machine lifetime 一致，且希望這個 pod 可以比其他 pod 更早啟動，也不要使用 ReplicaSet，而是要改用 [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)



ReplicaSet 相關操作
==================

## 刪除 ReplicaSet & 相關的 Pod

透過 [kubectl delete](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete) 刪除，k8s 中的 [Garbage Controller](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/) 會自動的刪除相對應的 pod。

但若是透過 REST API k的方式，則必須要將 `propagationPolicy` 設定為 **Foreground** 才可以：

```bash
$ kubectl proxy --port=8080

$ curl -X DELETE  'localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/frontend' \
> -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
> -H "Content-Type: application/json"
```


## 僅刪除 ReplicaSet

若只要刪除 ReplicaSet 而不影響 pod，透過 [kubectl delete](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete) 搭配 `--cascade=false` 參數就可以完成。

若使用 REST API，則必須要將 `propagationPolicy` 設定為 **Orphan**：

```bash
$ kubectl proxy --port=8080

$ curl -X DELETE  'localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/frontend' \
> -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
> -H "Content-Type: application/json"
```


## 將 ReplicaSet 與 Pod 隔離

通常會這麼做的目的大多在於進行 debugging 或是 data recovery。

作法不困難，只要修改 pod label，讓 ReplicaSet 的 label selector 選不到該 pod 即可；但同時要搭配調整 ReplicaSet 中的 `.spec.replicas` 的設定，不然新的 pod 又會被產生出來。


## ReplicaSet Scaling

調整 `.spec.replicas` 設定即可，k8s 會自動維護使用者所指定的 desired status。


## 與 HPA(Horizontal Pod Autoscaler Target) 的搭配

透過與 [Horizontal Pod Autoscaler Target(HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 搭配，可以指定 ReplicaSet Scaling 的上下限範圍，在資源使用率達到指定門檻時，讓 k8s 自動進行 scale out/in，以下是個簡單範例：

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  # 指定要搭配的 resource object，這裡是 ReplicaSet
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  # 指定 scale in/out 的 pod 數量為 3~10 個
  minReplicas: 3
  maxReplicas: 10
  # 資源使用率門檻為 CPU 使用率 50%
  targetCPUUtilizationPercentage: 50
```


References
==========

- [ReplicaSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)