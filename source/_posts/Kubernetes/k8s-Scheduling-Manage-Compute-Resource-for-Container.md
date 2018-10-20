---
layout: post
title:  "[Kubernetes] 分配 & 管理 container 所使用到的計算資源"
description: "本篇文章介紹如何在 Kubernetes 中，對於 container 中所使用的 computing resource 進行分配 & 管理"
date: 2018-10-21 05:00:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - CKA
---


前言
===

當設定 pod spec 時，若要針對計算資源進行管控，可以加入 CPU & memory 的相關設定，藉此可以讓 scheduler 可以更做出更好的分配決定，讓 pod 可以落在更合適的 worker node 上。



Kubernetes 支援那些 Compute Resource Types?
=========================================

在 k8s 中預設支援可用的計算資源為 CPU & memory，其中：

- CPU 以 core 為單位

- Memory 以 byte 為單為

因此在設定 pod spec 時可以指定每一個 container 需要多少計算資源，k8s 就會在硬體資源足夠的前提下，自動分配資源來完成工作。

而設定 Compute Resource Request/Limitation 的方式則是透過以下 4 的參數：

- `spec.containers[].resources.limits.cpu`

- `spec.containers[].resources.limits.memory`

- `spec.containers[].resources.requests.cpu`

- `spec.containers[].resources.requests.memory`

> 目前 CPU & Memory 的 resource limit 僅可以套用在 container 上而不是 pod 上



CPU & Memory 的單位
==================

## CPU

CPU 的單位通常是 VM 中的 vCore or vCPU，或是實體機中的一個 thread(開啟 Hyperthread 的前提下)，無論 worker node 跑在 CPU core 數有多大的機器，設定所能取得都是一樣的。

其中設定的單位是 `m`，每 **1000m = 1 vCore**，也可以使用分數，因此設定的方式可以是：

- 1 (相當於 1000m)

- 0.5 (相當於 500m)

- 300m (相當於 0.3)

> 設定 1m 是不被允許的，官方建議最低從 100m 開始


## Memory

Memory 設定的單位最低則是從 byte 開始，而使用的單位可以是單一字母的 `E, P, T, G, M, K`，也可以是雙字母的 `Ei, Pi, Ti, Gi, Mi, Ki`(比較常見)，以下是幾個設定範例：

- 104857600 (相當於 100 MB = 100*1024*1024)

- 100M

- 100Mi


## 設定範例

有了上面的概念後，以下是個簡單的設定範例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    # 第 1 個 container 的 resource limit 設定
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    # 第 2 個 container 的 resource limit 設定
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```



硬體資源實際上如何被分配?
====================

以上講的部份，其實都僅是個抽象的概念，目的就是要讓使用者了解 k8s 支援 resource limit，以及 resource limit 如何定義 & 使用。

而硬體資源實際上如何被分配呢? 這件事情要從兩個角度來看：

## 帶有 Resource Request 的 Pod 如何被指派?

由於 k8s 中的每一個 worker node 都有其 CPU & Memory 上限，因此關於 pod 如何被指派到特定的 worker node 上運行，大致上有幾個原則：

- 將要被分配的 pod 中所有 container 所要求的資源一定會小於 worker node 目前剩餘資源

- 即使 container 要求的資源很低，若是 worker node 無法滿足，scheduler 也不會把 pod 分配過去運作(確保尖峰時段造成 workload peak 讓 worker node 無法負荷)


## 帶有 Resource Limit 的 Pod 是如何運作的?

由於最底層的 container 不是由 k8s 負責的，因此資源限制 & 使用這件事情上就得仰賴各個 container runtime(例如：docker, cri-o ... 等等)來處理，當然也是有規則來處理這件事情，以下用 Docker 為例來說明：

- `spec.containers[].resources.requests.cpu` 的值會被除以 1024 後，轉換為 core value 後傳給 docker，這就是在 **docker run** 時帶入 [`--cpu-shares`](https://docs.docker.com/engine/reference/run/#runtime-constraints-on-resources) 參數的效果

- `spec.containers[].resources.limits.cpu` 的值則會被除以 100，而出來的結果就是 container 可以在**每 100ms cpu time**中使用多少 cpu time

> k8s 預設使用的 quota period 為 100ms，但最低其實可以到 1ms

- `spec.containers[].resources.limits.memory` 則是類似在 **docker run** 中帶入 [`--momery`](https://docs.docker.com/engine/reference/run/#/user-memory-constraints) 參數的效果


## 如果 resource 使用額度爆了怎辦?

若是遇到 resource 使用額度爆掉或是目前資源已經不足的的情況，k8s 會根據以下準則來處理：

- 若是 container 使用超過所設定的 **memory limit**，可能就會被終止；如果這個 container 是可以被重新啟動的，kubelet 就會將其重新啟動

- 若是 container 使用超過所設定的 **memory request**，當負責運行該 container 的 worker node 若是遇到記憶體不足時，會整個 pod 一起被趕走

- container 使用超過 CPU 使用額度就不一定會被終止 (但因為 cgroup 限制的關係，其實也超過不了多少)

- 若 container 要求比目前 worker 都還更多的資源，那就會造成無法被分派到任一個 worker node 的狀況



遇到 Resource 相關問題怎麼解?
==========================

以下針對因為資源不足所可能造成的問題作一些說明:

## Pod 一直無法成功分派

若是發現 pod 一直無法被分派到任何一個 worker node，然後透過 `kubectl describe pod/xxxx` 指令又發現有 **PodExceedsFreeCPU** or **PodExceedsFreeCPU** 等關鍵字時，大概就是因為 worker node 資源不足造成的，此時可以透過以下幾種方式解決：

- 增加資源較為充足的 worker node

- 砍掉不需要的 pod，讓資源釋放出來

- 確認 pod 中的 container 所要求的資源沒有超過目前 worker node 所能提供的

> 透過 `kubectl describe node/xxxx` 可以檢視 node 詳細資訊(一共有多少資源，有那些 pod 在上面、消耗多少資源 .... etc)，其中 `Allocatable` 欄位會列出 worker node 可用的 CPU & Memory 資源


## Pod 被終止

若 pod 不斷被重啟，但又不是程式邏輯錯誤所造成時，可能就是因為資源不足所造成的，這時候可以透過指令 `kubectl get pod -o json`，可以仔細看看是不是有 `terminated:map` & `reason` 關鍵字。就可以看出 pod 被終止的真正原因。


可以限制磁碟使用空間嗎?
===================

也許有人會問，若是 container 沒有外掛 external storage，然後又無限制的使用磁碟空間的話，那 worker node 上的磁碟空間不就有可能爆掉? 是否可以透過磁碟使用的限制來避免這個問題的發生呢?

這個問題的答案是 "**Yes**"，在 v1.8 後，使用者可以開始針對 ephemeral storage 進行磁碟空間的限制，詳述如下：

## 什麼是 ephemeral storage?

這功能在原生的 container runtime 並沒有提供，因此 k8s 就必須自己來處理，因此在 v1.8 版後釋出了一個名稱為 `ephemeral-storage` 的 resource，用來管理 host 上的短暫儲存空間。

> ephemeral-storage 在 v1.10 後被列為正式功能，不需在透過 feature gate(`LocalStorageCapacityIsolation`) 開啟；但目前在 v1.12 中還是 beta，尚未 GA

在每個 k8s node 上的 kubelet 在運行時，總是會需要儲存一些資料在本地端，因此以下的其中兩個空間就會被拿來利用：

- kubelet root directory: `/var/lib/kubelet`

- log directory: `/var/log`

由於被 kubelet 管理到的部份，例如：pod log, image layer, container writable latyer，甚至是在 pod spec 中定義 `emptyDir`，都會利用到上面提到的這兩個空間，而 ephemeral-storage 就是針對上述這兩個空間進行管理

> 若是在安裝時額外將 images later or container writable layer 放到其他獨立的分割區中，ephemeral-storage 就管不著了 (只能管理到主分割區 `/`)

## 設定 ephemeral storage 使用限制

ephemeral storage 額度設定可透過以下兩個設定來完成：

- `spec.containers[].resources.limits.ephemeral-storage`

- `spec.containers[].resources.requests.ephemeral-storage`

設定的單位跟 memory 其實一樣，可以是單一字母的 `E, P, T, G, M, K`，也可以是雙字母的 `Ei, Pi, Ti, Gi, Mi, Ki`，以下是一個簡單的設定範例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  # 整個 pod 一共要求 4G 的 ephemeral storage 空間
  # 以及使用 ephemeral storage 的空間限制為 8G
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      # db 要求 2G 的 ephemeral storage 空間
      requests:
        ephemeral-storage: "2Gi"
      # db 使用 ephemeral storage 的空間限制為 4G
      limits:
        ephemeral-storage: "4Gi"
  - name: wp
    image: wordpress
    resources:
      # wp 要求 2G 的 ephemeral storage 空間
      requests:
        ephemeral-storage: "2Gi"
      # wp 使用 ephemeral storage 的空間限制為 4G
      limits:
        ephemeral-storage: "4Gi"
```

## 其他問題

其他比較常見的問題可能會有：

### 帶有 ephemeral-storage requests 的 pod 要怎麼被分派?

由於 kubelet 會運行在每一個 node 上，因此會定期回報 node 相關資訊，當然也包含磁碟容量。

> 每個 node 的 ephemeral storage 可透過 `kuebctl describe node/xxxx` 指令查詢，並尋找 **Allocatable > ephemeral-storage** 即可知道有多少磁碟空間可用

因此假設在 CPU & memory 都沒問題的情況下，**磁碟空間最大**且**符合 pod 中所有 container 對於磁碟空間要求**的 node 會被選到。

### 帶有 ephemeral-storage limit 的 pod 會如何運作

由於帶有 ephemeral-storage limit，因此 k8s 就會持續監控 pod 中的 container 的磁碟使用狀況是否有超過設定限制，pod 就會被趕走。

而以上是以 container level 的角度來看，但若是以 pod level 來看，只要所有 container 使用空間 & emptyDir volume 使用空間的加總超過使用限制，pod 也會被趕走。


其它擴充資源(Extended Resources)
==============================

由於資源這東西不會只有 CPU/Memoery/Disk 三種，總是會有其他的資源，例如：GPU。但這一類的資源並不是普遍存在在每一台機器上，因此在 k8s 就不會以內建支援，因此就必須改用擴充的方式進行資源管理。

> 額外擴充的資源就不會屬於 `kubernetes.io` domain，而是可以由管理者自己定義

既然 extended resource 需要由管理者自行定義，那要讓 container 使用到 extended resource，最基本就要完成兩件事情：

1. 管理者要告訴 k8s 有哪些 extended resource (透過 advertise 的方式)

2. 在 pod spec 中要有對於這些 extended resource 的 request 設定

而 extended resource 可以根據其所影響的範圍，可以分為 node-level & cluster-level：

## Node-level extended resources

node level 的資源即是跟 node 相關的資源，而 node-level extended resource 有分為兩種：

### Device plugin managed resources

由於愈來愈多硬體加速的需求在雲平台上面跑出來，因此在 k8s 就提出了 

這是因為通常會是特定的硬體，例如 k8s 中有提出 [Device Plugin][device_plugin] 的概念，讓使用者可以自訂在某些 node 上會有特定的硬體資源(例如：GPU, FPGA, InfiniBand .... 等等)。

### 其他資源

至於其他資源的方式，由於沒有一個明確的 resource object 來處理這個，因此可以透過發一個 HTTP `PATCH` request 給 API server，為特定 node 上的 `status.capacity` 資訊中增加特定的資源，完成了之後，透過 `kubectl describe node/xxxx` 就可以在 `status.allocatable` 資訊中看到所新增的資源。

一旦新增了 extended resource，kubelet 就會根據 pod 已經所消耗的數量，定期匯報資訊給 API server，因此 scheduler 在進行 pod 分派的時候就可以執行正確的決定。

以下用一個簡單的例子來說明如何為特定的 node 新增 extended resource：

```bash
$ kubectl get nodes
NAME              STATUS   ROLES    AGE   VERSION
leon-k8s-node00   Ready    master   8d    v1.12.1
leon-k8s-node01   Ready    master   8d    v1.12.1
leon-k8s-node02   Ready    master   8d    v1.12.1
leon-k8s-node03   Ready    node     8d    v1.12.1
leon-k8s-node04   Ready    node     8d    v1.12.1
leon-k8s-node05   Ready    node     8d    v1.12.1

# 檢視 node03 的 status.capacity 資訊
$ kubectl describe node/leon-k8s-node03
Name:               leon-k8s-node03
Roles:              node
....(略)
Capacity:
 attachable-volumes-azure-disk:  16
 cpu:                            4
 ephemeral-storage:              64989928Ki
 hugepages-1Gi:                  0
 hugepages-2Mi:                  0
 memory:                         4046000Ki
 pods:                           110
....(略)

# 使用 HTTP PATCH，在 node03 中新增一個 example.com/foo 的 extended resource，數量為 5
# 其中 ~1 是 "/" 字元經過 url encoding 之後的結果
$ curl --header "Content-Type: application/json-patch+json" \
--request PATCH \
--data '[{"op": "add", "path": "/status/capacity/example.com~1foo", "value": "5"}]' \
http://localhost:8080/api/v1/nodes/leon-k8s-node03/status

# 再一次檢視 node03 的 status.capacity 資訊
# 可以看到新增的 exmaple.com/foo 已經出現
$ kubectl describe node/leon-k8s-node03
Name:               leon-k8s-node03
Roles:              node
....(略)
Capacity:
 attachable-volumes-azure-disk:  16
 cpu:                            4
 ephemeral-storage:              64989928Ki
 example.com/foo:                5
 hugepages-1Gi:                  0
 hugepages-2Mi:                  0
 memory:                         4046000Ki
 pods:                           110
....(略)
```

## Cluster-level extended resource

cluster level 的資源就不會與特定的 node 綁定，因此就由更上層的 scheduler extender 來管理，這個部份等之後有接觸過 & 有更多實用的資訊後再來補....



未來發展方向
==========

從上面可以看出，其實 k8s 目前可以進行 resource limit 的能力 & 範圍還是相當有限，但 IT 的世界本來就是一直在進步的，因此有些概念被提了出來，未來可能會往以下幾個方向來改進：

- 現在的 resource limit 是設定在 container level，未來可能可以設定在 pod level

- 未來希望可以新增更多 resource type 的支援，當然也包含自訂的部份

- 支援 resource overcommitment，藉此來達成某種程度的 QoS

- 以目前來說，一個 CPU 其實在不同的 provider 之間的定義是不同的，未來會考慮將這個問題解決



References
==========

- [Managing Compute Resources for Containers - Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)

- [体验ephemeral-storage特性来对Kubernetes中的应用做存储的限制和隔离-博客-云栖社区-阿里云](https://yq.aliyun.com/articles/594066)



[device_plugin]: https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/ "Device Plugin"