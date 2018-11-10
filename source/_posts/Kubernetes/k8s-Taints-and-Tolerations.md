---
layout: post
title:  "[Kubernetes] Taints and Tolerations"
description: "本篇文章的主題在介紹在 Kubernetes 中，'taint' & 'toleration' 這兩種機制的設計會如何影響 scheduler & k8s 相關自動化維運的行為"
date: 2018-10-31 04:30:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Scheduling
  - CKA
  - CKA Scheduling
---


前言
===

`taint` 跟[上一篇](k8s-Assigning-Pod-to-Nodes)提到的 node affinity 雖然都是屬於 scheduling 的一部份，但要達成的目的其實完全相反：

- `node affinity`：設計如何讓 pod 被分派到某個 worker node

- `taint`：設計讓 pod 如何**不要**被分派到某個 worker node

而 taint 並非單獨運作，而是與 `toleration` 共同搭配使用，目的就是要避免讓 pod 被分派到不正確 or 不合適的 worker node 上，運作原理大概如下：

> 如果有特定的 node 被加上了 taint(汙點)，pod 就不會被分派到上面，除非 pod spec 有設定 toleration(容忍) 來接受這些 taint (必須全部 taint 都接受才行)


設定 Taint & Toleration
======================

運作規則要分為以下幾個部份說明，分別是：

- 如何為 node 設定 taint，避免 pod 被分派到上面

- 如何為 node 移除 taint

- 如何在 pod spec 中設定 toleration，讓 pod 可以分派到有 taint 的 node 上

## 如何設定 node taint

每個 taint 都有以下 3 個屬性：

1. `Key`

2. `Value`

3. `Effect`：共有三種，分別是 **NoSchedule**, **PreferNoSchedule** & **NoExecute**

而設定 node taint 很簡單，透過 kubectl 執行以下指令將 3 個屬性給入即可即可：

> kubectl taint nodes node1 key=value:NoSchedule

以下是一個實際操作範例：

```bash
$ kubectl get node
NAME              STATUS   ROLES    AGE   VERSION
... (略)
leon-k8s-node03   Ready    node     15d   v1.12.1
leon-k8s-node04   Ready    node     15d   v1.12.1
leon-k8s-node05   Ready    node     15d   v1.12.1

# 檢視 node 狀態細節，查看 taint 設定狀態
# 可以看出目前並沒有任何的 taint 設定
$ kubectl describe node/leon-k8s-node03
Name:               leon-k8s-node03
Roles:              node
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/hostname=leon-k8s-node03
                    node-role.kubernetes.io/node=true
... (略)
Taints:             <none>
Unschedulable:      false
... (略)

# 為 node03 加上 taint
$ kubectl taint nodes leon-k8s-node03 key=value:NoSchedule
node/leon-k8s-node03 tainted

# 重新檢視 node 狀態細節，察看 taint 設定狀態
# 目前已經多了一個 taint 的設定
$ kubectl describe node/leon-k8s-node03
Name:               leon-k8s-node03
Roles:              node
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/hostname=leon-k8s-node03
                    node-role.kubernetes.io/node=true
... (略)
Taints:             key=value:NoSchedule
Unschedulable:      false
... (略)
```

當以上步驟完成後，後續新增進來的 pod 就不會被分派到這個 node 上。

## 如何移除 taint

移除的語法很簡單，只要在 taint 的 Key:Effect 後面加上 `-` 即可：

> kubectl taint nodes node1 key:NoSchedule-

## 設定 pod toleration

pod toleration 是設定在 pod spec 中，我們可以設定以下的 pod toleration 讓 pod 可以接受經過上面指令所產生的 node taint：

```yaml
# 表示可以接受"帶有 key=value & effect=NoSchedule" 的 taint
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

也可以是下面這樣設定：

```yaml
# 表示可以接受"存在 key(不論 value 為何) & effect=NoSchedule" 的 taint
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

在 toleration 的設定中，有一些比較特別的案例需要說明一下：

- operator 若是沒設定，預設值就是 `Equal`

- 若僅有設定 operator 為 Exists，卻沒有設定 key，那表示會 tolerate(容忍) 所有的 taint

```yaml
tolerations:
- operator: "Exists"
```

- 僅有設定 key，沒有設定 effect，表示只有帶有相同 key 的 taint 都會被容忍

```yaml
tolerations:
- key: "key"
  operator: "Exists"
```



Effect 運作規則說明
=================

由於 key & value 這兩個部份是比較容易理解的，而比較有差異的部份主要落在 `Effect` 的部份，因此以下就 Effect 進行比較詳細的說明。

Taint Effect 共有分成三種，分別是：

- `NoSchedule`

- `PreferNoSchedule`

- `NoExecute`

由於 taint & toleration 是互相抵消的關係，若 node 上有設定 taint，而 pod spec 中又有設定符合的 toleration，就會互相抵消；而 k8s scheduler 在判斷時會以最後剩下的 taint 來進行工作分派的依據。

而 effect `NoExecute` 的 taint 被加到 node 的時候也會影響到目前正在該 node 上運作的 pod。

## NoSchedule

假設最後某個 node 上留下的 taint 的 effect 為 `NoSchedule`，那 k8s 就不會把該 pod 分派到該 node 上，但不影響正在運作中的 pod。

## PreferNoSchedule

假設最後某個 node 上留下的 taint 的 effect 為 `PreferNoSchedule`，那 k8s 就儘量不會把該 pod 分派到該 node 上(最後要是沒辦法的時候還是會破功)，但不影響正在運作中的 pod。

## NoExecute

假設某個 node 被設定了 effect 為 `NoExecute` 的 taint，那 k8s 還會把已經存在該 node 上的 pod 趕走，也不會把該 pod 分派到該 node 上。

> 設定 `tolerationSeconds` 可以表示在 taint 被增加之後，**帶有相對應 toleration 的 pod 還可以在該 node 上存在多久**



實際上如何運用?
============

上面提到了 taint & toleration 的運作規則，也說明了不同的 taint effect 會有哪些不同的影響，那實際上在什麼樣的情況可以用 taint & toleration 機制來處理呢?

基本上，taint 機制設計的目的，就是**不要讓 pod 被分派到某個 node 上**，而 toleration 則是在上述的前提下可以進行例外的設定，因此就可以有類似以下的 use case 產生：

## 專用的 Node

如果希望某個 node 只給特定群組的使用者使用，可以使用以下的方式增加 taint：

> kubectl taint nodes nodename dedicated=groupName:NoSchedule

有了上面的 taint 設定後，一般的 pod 就不會被分派到上面；而該群組的使用者只要在 pod spec 加入對應的 toleration 即可讓 pod 分配到專用的 node 上，而若是覺得每次都要設定 pod spec 很麻煩，或許可以考慮 k8s [Admission Controller][admission controller] 來讓這件事情變得簡單些。

> Admission Contoller 可以自動幫 pod 加上 toleration
 
 > 設定 pod toleration 並不是保證一定會被分派到專屬的 node 上，也有可能還是被分派到到其他沒有 taint 的 node 上；因此若是要讓 pod scheduler 可以更精確的把 pod 放到專屬的 node，可以搭配 **Node Label + nodeSelector** 一起使用


## 帶有特殊硬體的 Node

由於特殊硬體(例如：GPU, FPGA ... etc)一般來說都不便宜，因此在一個 cluster 中可能會僅有某些 node 安裝特殊硬體，而通常這一類的特殊硬體還是多少都會消耗 CPU & memory 資源，因此在合理的思考下，就不會讓一般不需要用到到特殊硬體被分派到這些 node 上。

因此，可以在帶有這些特殊硬體的 node 上設定類似以下的 taint：

```bash
$ kubectl taint nodes nodename special=true:NoSchedule

$ kubectl taint nodes nodename special=true:PreferNoSchedule

$ kubectl taint nodes nodename gpu=true:NoSchedule

$ kubectl taint nodes nodename fpga=true:NoSchedule
```

而 pod toleration or [Kubernetes Admission Controller][admission controller] 的設定當然也不能或缺；但若是真的使用特殊硬體，也可以考慮使用 [Extended Resource](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#extended-resources) + [ExtendedResourceToleration admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#extendedresourcetoleration) 的機制來處理 & 運用。

> 透過 `Extended Resource` + `ExtendedResourceToleration admission controller` 可以確保 pod 可以被自動的加上被分派到帶有特殊硬體的 node 上所需要的 toleration 設定


## 當 Node 發生問題時

當 node 發生問題時(或是任何其他會造成該 node 無法繼續提供服務的情況)，管理者需要考慮驅逐目前在上面運行中的 pod，可以透過加上 taint(Effect=`NoExecute`) 的方式達成，k8s 就會自動幫忙處理後續的作業。



Taint based Evictions
=====================

首先再度說明當 node 被加上 taint(Effct=`NoExecute`) 後會自動的發生以下行為：

1. 沒有設定對應 toleration 的 pod 會被馬上驅離

2. 有設定相對應 toleration 的 pod 則會繼續留在該 node 上

3. 有設定對應 toleration 的 pod，也設定 `tolerationSeconds`，會在該 node 上停留一陣子(根據設定值)後就會被驅離


由於以上這些行為都是自動完成的，因此在 Kubernetes v1.6 之後，就可以開始利用這個機制來執行當特定 node 發生問題時的處理。

> 由於 k8s 中有 node controller 會持續監控每個 node 的狀態並回報，因此當它發現某些 node 有狀況時，可以透過為這個 node 增加 taint 的方式，將上面正在運作的 pod 驅離到其他 node 上去執行

為了自動化的完成這些事情，k8s 設計了幾個專用的 taint 來描述 node 的相關問題：

- `node.kubernetes.io/not-ready`：Node is not ready

- `node.kubernetes.io/unreachable`：node controller 無法聯繫到該 node

- `node.kubernetes.io/out-of-disk`：磁碟空間不足

- `node.kubernetes.io/memory-pressure`：記憶體快要耗盡

- `node.kubernetes.io/disk-pressure`：磁碟空間快要耗盡

- `node.kubernetes.io/network-unavailable`：網路有問題

- `node.kubernetes.io/unschedulable`：無法分派 pod 到該 node

- `node.cloudprovider.kubernetes.io/uninitialized`：node 尚未初始化完成，還無法使用 (這是給 public cloud provider 用的 taint)

可能會有人有疑問....如果 node 一旦被偵測出有問題，上面的 pod 就會馬上被驅離，但可能只是斷線一下就回復了，馬上驅離的作法妥當嗎?

因此為了處理這樣的狀況，當 node 發生問題時，除了在該 node 上增加一個 taint 的設定外，還會在該 node 上的每個 pod 加上相對應的 toleration 設定，並設定 tolerationSeconds=300，這表示每個 pod 都還可以留在該 node 上 5 分鐘。

> tolerationSeconds 的設定是 k8s 自動給進去的，若要更改預設值 300，可以透過 [DefaultTolerationSeconds admission controller](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission/defaulttolerationseconds)

此外，要讓 taint based eviction 運作，必須要開啟 `TaintBasedEvictions` feature gate (預設是關閉的)

> 設定這功能需要注意 node controller 中的 `--node-eviction-rate`(預設 0.1，表示 10 秒驅離一個 pod) 設定，必須很小心的設定這個值，避免大規模驅離 pod 時造成整個 cluster 崩潰的連鎖效應 (特別是當 master node 發生崩潰的時候)

最後，官網文件有提到 DaemonSet 最自動加上以下 toleration(Effect=`NoExecute`)：

- **node.alpha.kubernetes.io/unreachable**

- **node.kubernetes.io/not-ready**

但在 v1.12 的環境下檢視已經存在的 DaemonSet，並沒有看到這樣的設定，這部份有帶後續釐清。



Taint Nodes by Condition
========================

如果我們希望當 node 發生特定問題的時候，不要自動被加上 taint，避免 pod 無法被分派到該 node 時，可以怎麼做呢? 答案是透過 `TaintNodesByCondition` feature。

在 v1.12 後，`TaintNodesByCondition` feature 已經變成 beta 功能且預設開啟，管理者可以透過這個功能**選擇性**的忽略某些 node 的狀況，而 `TaintNodesByCondition` 實際運作狀況是這樣的：

- 若 node 發生狀況時，k8s 會自動加上 Effect=`NoSchedule` 的 taint 到該 node 上

- 若 TaintNodesByCondition 有設定忽略特定狀況，就不會加上 taint

> 跟 TaintBasedEviction 不同，一個是增加 Effect=`NoExecute`(TaintBasedEviction) 的 taint，一個是增加 Effect=`NoSchedule`(TaintNodesByCondition) 的 taint

最後，官網文件有提到 DaemonSet controller 會為所有的 DaemonSet 自動增加以下 toleration(Effect=`NoSchedule`)：

- **node.kubernetes.io/memory-pressure**

- **node.kubernetes.io/disk-pressure**

- **node.kubernetes.io/out-of-disk** (只有對重要的 pod)

- **node.kubernetes.io/unschedulable** (v1.10 之後)

- **node.kubernetes.io/network-unavailable** (若設定使用 hoet network 時)

但在 v1.12 的環境下檢視已經存在的 DaemonSet，並沒有看到這樣的設定，這部份有帶後續釐清。



References
==========

- [Taints and Tolerations - Kubernetes](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/)

- [Kubernetes中的Taint和Toleration（污点和容忍） - 宋净超的博客|Cloud Native|云原生布道师](https://jimmysong.io/posts/kubernetes-taint-and-toleration/)

- [Kubernetes Pod调度进阶：Taints(污点)和Tolerations(容忍) — 青蛙小白](https://blog.frognew.com/2018/05/taint-and-toleration.html)



[admission controller]: https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/ "Kubernetes Admission Controllers"