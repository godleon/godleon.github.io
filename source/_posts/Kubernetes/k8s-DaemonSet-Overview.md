---
layout: post
title:  "[Kubernetes] DaemonSet Overview"
description: "本篇文章的主題在介紹 Kubernetes 中的 DaemonSet resource object 是如何運作 & 相關細節"
date: 2018-10-04 05:00:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Core Concept
  - CKA
  - CKA Core Concept
---


What is DaemonSet ?
===================

DaemonSet 是確保在 Kubernetes 中的每一個 node 上都會有一個指定的 pod 來運行特定的工作，當有新的 node 加入到 Kubernetes cluster 後，系統會自動在那個 node 上長出相同的 DaemonSet pod，當有 node 從 Kubernetes cluster 移除後，該 node 上的 DaemonSet pod 就會自動被清除掉。

> 也可以讓 DaemonSet 佈署在特定幾個 node 上，透過 scheduling 的機制

了解以上 DaemonSet 的特色後，則會有一些比較標常見的應用例如：

- 在每個 node 上運行 storage daemon，例如：ceph, glusterd

- 在每個 node 上運行收集 log 用的 daemon，例如：fluentd, logstash

- 在每個 ndoe 上運行監控用的 daemon，例如：[Prometheus Node Exporter](https://github.com/prometheus/node_exporter), [collectd](https://collectd.org/), [Datadog](https://www.datadoghq.com/) .... 等等

以上是一個 DaemonSet 作為單一用途的 daemon 時的範例，當然也有可能會有需要同時多個 DaemonSet 來達成某種複雜用途時的場景，這時候就可以還會包含到使用不同的 flag，或是針對不同的硬體型態有不同的 cpu & memory 的資源需求。



撰寫第一個 DaemonSet
==================

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  # .spec.selector 一旦定義後就無法再變更了
  # 必須與 .spec.template.metadata.labels 的定義相同
  # 這裡可以使用 matchLabels 或是 matchExpressions(用來處理較為複雜的 label 組合)
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  # 此為必要欄位，與 pod template 相同
  # 用來定義 DaemonSet 的內容應該要長什麼樣子
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    # 由於是 DaemonSet 的關係，因此 .spec.template.spec.restartPolicy 永遠是 "Always"
    # 預設值為 "Always"
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

有幾個地方在定義 DaemonSet 的時候需要比較注意的：

- `.spec.selector` 一旦定義後就不建議再修改，若是修改可能會造成某些 pod 變成孤兒(因為 DaemonSet controller 無法管理到正確的 pod)

- `.spec.selector` 的定義必須與 `.spec.template.metadata.labels` 相同，否則會被 API server 拒絕套用

- 不可以再建立(或是透過其他 controller 建立，例如：**Deployment**)帶有與 DaemonSet 相同 label 組合的 pod，否則會被 DaemonSet 認為是自己所產生的 



誰負責 DaemonSet 中的 Pod Scheduling?
===================================

這個部份在 v1.12 之後已經有所改變。

## DaemonSet controller (v1.12 之前)

一般來說決定 pod 要在哪個 node 上運行是 k8s scheduler 的工作，但其實 DaemonSet controller 跟 k8s schedule 之間的運作有時就是會有矛盾之處，因為：

- 若是使用者為特定的 node 設定 [unschedulable](https://kubernetes.io/docs/concepts/architecture/nodes/#manual-node-administration)，這通常就會跟 DaemonSet controller 的規則衝突

- DaemonSet controller 可以在 k8s scheduler 還沒啟動前就佈署 pod，有時對於 k8s cluster bootstrap 是很有用途的


## default scheduler (v1.12 之後)

而為了解決上述的矛盾，在 v1.12 之後，DaemonSet pod 預設是回到由 k8s scheduler 統一來處理分派的工作，藉此避免以下狀況的發生：

- 不一致的 Pod 生成行為：一般的 pod 在等待被分派前，被建立後會先進入 `Pending` 狀態，但 DaemonSet Pod 不會先進入 Pending 狀態，這樣的過程可能會讓使用者混淆

- 即使 [Pod preemption](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/) 被啟用(這是由 k8s scheduler 所負責處理的一個功能)，也不會被 DaemonSet controller 考慮進來

而讓 k8s scheduler 取代 DaemonSet controller 的方法很簡單，只要移除在 DaemonSet 中的 `spec.nodeName` 的部份，並加入類似以下的 [Node Affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity) 設定即可：

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchFields:
      - key: metadata.name
        operator: In
        values:
        - target-host-name
```

若是原有的 DaemonSet pod 已經有 node affinity 的設定，上面新增的設定則會會覆蓋掉原本舊有的部份。

此外，針對 DaemonSet 設定的變更的部份，若是變動的部份是 `spec.template`，DaemonSet controller 並不會理會，因此變更就會無效；但 k8s scheduler 則還是會協助此變更的進行。


## 關於 Taints & Tolerations

然而 scheduling 這件事情由 DaemonSet controller 回到了 k8s scheduler 身上，因此 `Taint` & `Toleration` 這兩個機制就會對 DaemonSet 就顯得相對重要了，因為 `Noschedule` 的設定會影響 DaemonSet pod 的分派。

> 關於 `Taint` & `Toleration` 機制的說明，可以參考[此文章(深入kubernetes調度之Taints和Tolerations — VF的部落格)](http://cloudnil.com/2017/06/08/Schedule-taints-tolerations/)

而在原本由 DaemonSet controller 負責 scheduling 的情況下，若是 node 被設定為 **unschedulable**，也是同樣會被忽略的，但回到 k8s scheduler 之後，就必須額外增加 `Toleration` 的設定來達到相同的效果，以下是會被自動加入到 DaemonSet pod 中的 toleration 清單：

| Toleration Key | Effect | Version | Description |
|----------------|:------:|:-------:|-------------|
| `node.kubernetes.io/not-ready` | `NoExecute` | `1.8+` | 若 `TaintBasedEvictions` 啟用時，pod 不會因為 node 發生問題而被驅離 |
| `node.kubernetes.io/unreachable` | `NoExecute` | `1.8+` | 若 `TaintBasedEvictions` 啟用時，pod 不會因為 node 發生問題而被驅離 |
| `node.kubernetes.io/disk-pressure` | `NoSchedule` | `1.8+` |   |
| `node.kubernetes.io/memory-pressure` | `NoSchedule` | `1.8+` |   |
| `node.kubernetes.io/unschedulable` | `NoSchedule` | `1.12+` | node 上的 unschedulable 屬性對 DaemonSet pod 會失效 |
| `node.kubernetes.io/network-unavailable` | `NoSchedule` | `1.12+` | node 上的 network-unavailable 屬性對 DaemonSet pod(使用 host network) 會失效 |



如何與 DaemonSet Pod 溝通?
========================

其實這跟一般的 pod 的運作方式沒什麼太大不同，不外乎就以下幾種：

## Push

在這種模式中，沒有 client 存在的必要性，因為是 pod 本身被設定為自動對外回報 or 更新資訊。

## NodePort

這就是一般的 host IP + port 的概念，外部的 client 可以透過 IP + port 的型式直接存取 pod

## DNS

可以跟 StatefulSet 一樣，建立一個 [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) 搭配 pod selector，讓特定的 domain name 在 k8s cluster 內部可以直接被解析為 pod IP

## Service

這就是一般的 service，透過 service VIP 與後面的 pod 溝通，但缺點就是無法指定存取特定的 pod



更新 DaemonSet
=============

關於更新

- 若是 node label 變更了，DaemonSet 可能會在該 node 新增對應的 pod 或是移除不符合 label set 的 pod

- 可以直接修改 DaemonSet pod，但並不是 pod 的每一個欄位都可以修改

- 承上，即使 pod 被修改，下次有新的 node 加入的時候，DaemonSet 依然會用原本的 template 來產生 pod (因為 template 並沒有變動)

- 若是想刪除 DaemonSet 卻想保留 DaemonSet pod，可以使用 `kubectl` + `--cascade=false` 來完成

- 承上，若是接著有另一個帶有不同 template 卻相同 label selector 的 DaemonSet 被建立時，原有現存的 pod 不會被修改，但也不會產生新的 pod，此時就**必須要手動將原有的 pod 刪除，讓新的 pod 可以自動被產生**

- v1.6 之後對 DaemonSet 進行 [rolling update](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/)



DeamonSet 的替代方案?
===================

由於 DeamonSet 的用途就是在每個(or 特定的) node 上運行一個 pod，確保每個 node 都可以執行特定的工作，例如：monitoring。但這樣子的工作只有 DeamonSet 可以完成嗎? 答案當然不是。

但為什麼要使用 DeamonSet? 帶來的優點是什麼? 以下列出幾個替代方案來比較：

## Init Script

這是直接在 node 上啟動一個 daemon process(並非 pod)，例如：`init`, `upstartd`, `systemd` ... 等等，但使用 DeamonSet 可以帶來其他好處：

- 可以跟其他的 application 一樣，具備有監控 & 管理 daemon log 的能力

- 像其他 application 一樣使用相同的配置方式 & 語言 (例如: pod template, kubectl)

- 透過在 DeamonSet pod 上設定 resource limit，將 DeamonSet pod 儘量不佔用到 application pod 的資源 

## Bare Pod

這是直接在 node 上建立一個 pod，跟 DaemonSet 是相同的。

但 DeamonSet 是自動完成這件事情，而且在發生像是 node failure 或是進行維護狀態(例如: kernel update) 的事件時，DeamonSet 也會自動的刪除 & 對應的 pod。

## Static Pod

在 k8s 中提供了一個特殊的機制，稱為 [static pod](https://kubernetes.io/docs/concepts/cluster-administration/static-pod/)，可以透過在 node 中的某個目錄寫入一個檔案後，由 kubelet 來產生對應的 pod。

但 static pod 的問題在於不受 API server 或是任何的 client 所管控，因此通常只適用在 cluster bootstrap 的時候用。

> 在更後面的版本中，static pod 這個機制可能會被移除


## Deployment

Deployment 則是跟 DeamonSet 很相似，同樣都是建立 pod，並且持續的監控這個 pod 確保持續存在。

但 Deployment 比較適合用在 Stateless 的服務(例如 web server)，而這一類的服務的重點在於確保 replica 的數量會一直維持在使用者所提供的 desired status，並非**每一個 node 都要有一個特定的 pod**上。

因此若是目的是要達成在每個 node 上都運行特定的 pod，DeamonSet 會是比 Deployment 更好的選擇。



References
==========

- [DaemonSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

- [DaemonSet · Kubernetes Handbook - Kubernetes中文指南/云原生应用架构实践手册 by Jimmy Song(宋净超)](https://jimmysong.io/kubernetes-handbook/concepts/daemonset.html)