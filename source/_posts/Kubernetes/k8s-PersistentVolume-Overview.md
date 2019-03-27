---
layout: post
title:  "[Kubernetes] Persistent Volume (Claim) Overview"
description: "本篇文章的主題在介紹 Kubernetes 中的 Persistent Volume (Claim) 的設計概念 & 使用方式"
date: 2019-02-08 12:50:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Storage
  - CKA
  - CKA Storage
---


Preface
=======

管理 storage 一直都是一個不簡單的課題，為了讓 developer 可以更專注在開發上，k8s 中提出了兩個概念(resource object)，分別是 `Persistent Volume(PV)` & `Persistent Volume Claim(PVC)`，透過這兩個概念將`連結 storage` 的過程抽象化，讓使用者不需要了解 storage 在底層是如何運作的，只要了解如何使用即可，大幅降低使用者操作 storage 的困難度。

`Persistent Volume(PV)` 由 storage 管理者負責產生，在 k8s 中就是一種可用的 storage resource，同樣也是一種 volume plugin，但有自己獨立的 lifecycle，且包含的就是實際與 storage 連結的實作細節。

> 目前 k8s 支援的 PV type 可以參考[此網址](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)

`Persistent Volume Claim(PVC)` 則是來自使用者的 storage request，就跟 pod 一樣都是要消耗特定的資源，PVC 消耗 PV 資源(pod 消耗 node 資源)，PVC 指定特定 size or access mode 的 PV(pod 可以指定 CPU, memory ... etc)。

由於 PVC 可以允許使用者自行指定所需要 PV 的相關屬性，因此除了 size, access mode 外，可能還會有 performance 的需求。而為了滿足不同的使用目的，cluster administrator 就必須要準備好不同的 PV 來處理不同 PVC 的需求；若是 PVC 數量不多且需求單純，手動產生 PV 可能還可以接受，但若是 PVC 的數量眾多且需求多變，那可能就需要 `[StorageClass][StorageClass]` 的協助了!



PV & PVC Interaction Lifecycle
==============================

有了上面對 PV & PVC 的基礎了解後，這個部份要說明從 PV 的產生到 PV & PVC 相互連結，一直到最後結束的這一段 lifecycle，會有以下的事件發生：

- Provisioning

- Binding

- Using

- Storage Object in Use Protection

- Reclaiming

- Expanding Persistent Volumes Claims

以下就仔細說明每個事件的細節。


## Provisioning

基本上，產生 PV 的方法有 `static` & `dynamic` 兩種，而這兩種分別說明如下：

### Static

static 很單純，就是 cluster administrator 預先建立幾個 PV，接著使用者在後續新增 PVC 的時候，可以相互 match 並開始使用 storage

### Dynamic

dynamic 不一樣的地方在於，**是當 k8s 中現存的 PV 沒有符合使用者的 PVC 需求時，會嘗試自動產生符合需求條件的 PV**，而這樣的自動化行為，則必須仰賴 [StorageClass][StorageClass] 來達成。

但要自動化產生 PV 來使用，必須要完成以下兩個條件：

- cluster administrator 必須預先設定好 StorageClass

- 在 PVC 設定中的 `spec.storageClassName` 必須設定已經存在的 StorageClass

- 若是要設定 default StorageClass，就必須要啟用 `DefaultStorageClass` admission controller (需要在 API server 的參數 `--enable-admission-plugins` 上加入 `DefaultStorageClass`)


## Binding

一般在 PVC 被建立之後，k8s 會自動在 cluster 中尋找是否有合適的 PV 可用(例如：容量, access mode ... etc)，若是尋找到就會將兩者連結(bind)起來；反之若是一直找不到合適的 PV，就會一直處在 `Unbound` 的狀態。

但如果是透過動態的方式所產生的 PV，就一定會與該 PVC 連結起來，產生出來的 PV 不一定會完全符合 PVC 需求(**可能會超過，但不會低於需求**)，而且只會是個一對一的結果。


## Using

在上個步驟中，cluster 會協助使用者的 PVC 需求找到相對應的 PV，然後 Pod 則會將 PVC 當作是 volume 一樣使用 & 掛載到特定目錄上。

- 關於 access mode 的部份，pod 會根據 PVC 中的定義來進行存取行為

- 當 PVC 已經與 PV 連結完成，pod 就可以根據需求一直使用，沒有使用時間的限制


## Reclaiming

當使用者已經不需要 volume 時，就可以將 PVC 從 cluster 刪除，此時與之連結的 PV 就會被釋放，並且可以提供其他 PVC 進行 reclaim，而 PV 中有個 reclaim policy，是用來告訴 cluster 當 PV 被釋放後，要如何處理已經存在的資料，目前支援的選項有 `Retained`, `Recycled` & `Deleted` 三種。

### Retained

若是 reclaim policy 設定為 Retain，當 PVC 刪除後(此時 PV 狀態為 `Released`)，cluster administrator 可以進行手動的 reclaim，步驟如下：

1. 刪除 PV (對應的實際資料並不會消失)

2. 清除實際資料 (optional, 如果有需求可做)

3. 若是使用已經存在的資料，那就再建立一個 PV，並指到原本的儲存空間

完成以上步驟後，就會有一個帶有原本資料且可以使用的 PV 了。

### Deleted

若是 volume plugin 支援刪除功能(例如：AWS EBS, GCE PD, Azure Disk, or Cinder volume)，使用帶有 reclaim policy `Delete` 的 PVC，當這個 PVC 被刪除後，相對應的 PV & 實際的 storage 空間也都會一併被刪除。

若是使用 StorageClass，透過 StorageClass 所產生出來的 PV 都會繼承來自 StorageClass 的設定(若沒有此設定，則預設為 `Delete`)，因此 cluster administrator 建立 StorageClass 時必須要根據使用者的實際期望來設定。

> 若是 PV 被建立後，reclaim policy 不是所預期的設定，還是可以透過 edit or patch 的方式修改

### Recycled

reclaim policy `Recycled` 未來將會被廢棄，若有同樣的需求，建議改用 dynamic provisioning 來取代。


## Expanding Persistent Volumes Claims

當 volume 容量不足時，擴充就變成了另一個需要面臨的問題，目前在 k8s 中，PVC 的擴充已經是預設支援的功能，而支援 PVC 擴充的 volume type 多屬於 public cloud，而 private cloud 可用的選項大概有：

- OpenStack Cinder

- Ceph RBD

- GlusterFS

- Portworx

但這功能也不是完全沒有限制，除了要上述的 volume type 之外，[Storage Class][StorageClass] 中也要預先有 `allowVolumeExpansion: true` 的定義。

而要擴充的方法也很簡單，透過 `kubectl edit pvc/xxxx` 的方式，設定個較大的 size value 後存檔即可，而這個行為不會產生新的 PV，而是對已經連結的 PV 進行擴充。

### 當 Volume 中包含檔案系統

若 PV 的源頭屬於 block device，因此就會包含一個使用者所需要的檔案系統，可能是 Ext4 or XFS，但在這情況下要進行容量擴充就會有限制：

- 目前僅有 XFS, EXt3, Ext4 的檔案系統可以進行擴充

- Pod 必須要以 ReadWrite mode 掛載

- 要擴充前必須將 pod 先移除，等待 volume 擴充完畢再重新建立 & 掛載

> 當 PVC status 變成 `FileSystemResizePending` 後就可以重新建立 pod 了

### 擴充使用中的 PVC

這功能目前還屬於 alpha 階段，使用上則會有以下幾點需要注意：

- 需要開啟 `ExpandInUsePersistentVolumes` feature gate

- 由於是擴充"**使用中**"的 PVC，因此 PVC 必須要被某個 pod 掛載才行



Persistent Volumes
==================

要了解 PV 的特性，可以從簡單的定義檔開始：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

上面是一個簡單使用 NFS 的 PV 定義檔，但卻可以從裏面得到很多資訊。

## Capacity

PV 都會有一個 `.spec.capacity` 的屬性，用來告訴使用者這個 PV 有多少容量可以使用，可用 Ei, Pi, Ti, Gi, Mi, Ki ... 等來標記。

> 目前只有 capacity 可作為 resource request 的依據，未來可能還會增加 IOPS, throughput .... 等等

## Volume Mode

在 v1.9 之前，所有的 volume plugin 都會自動在 PV 上建立 file system，但目前(v1.13)已經可以透過設定 `.spec.volumeMode: Block` 的方式來將 PV 作為 raw block device 使用。

> `Filesystem` 是預設值，而 `Block` 的功能目前(v1.13) 依然屬於 beta

## Access Modes

PV 可以透過 `ReadWriteOnce`(一個 node 可 read-write，可縮寫為 `RWO`), `ReadOnlyMany`(一個 node 可 write，多個 node 可 read-only，可縮寫為 `ROX`), `ReadWriteMany`(多個 node 可 read-write，可縮寫為 `RWX`) 三種存取模式提供掛載，但存取控制機制本身並非由 k8s 支援，而是由 PV resource provider 支援（例如上面的例子為 NFS，就可以支援 `ReadWriteMany`，但 Ceph RBD 則不行）。

此外，雖然 NFS 可以提供 `ReadWriteMany` access mode，但在 NFS share 本身的設定就有可能只是 RO，因此在 PV 上還是會有 Access Mode 的設定，主要的用意就是要用來告訴使用者目前這個 PV 所支援的 access mode 為何。

> 需要注意的是，每個 PV 掛載時只能使用一種 access mode，而每種 volume 支援的 access mode 可以參考[官網列表](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)

## Class

PV 可以擁有 class 的屬性，可透過 `spec.storageClassName` 來設定；若 PV 設定了 class，那就只能讓帶有相同 class request 的 PVC 連結；反之亦然。

## Reclaim Policy

透過 `spec.persistentVolumeReclaimPolicy` 來設定，上面已經有提到，有 `Retain`, `Recycle`, `Delete` 三種可以設定。

## Mount Options

k8s cluster 管理者可以在建立 PV 時，指定額外的 mount option，並在 volume 實際被掛載時套用；但需要注意的是，這功能並不是所有的 volume type 都支援，而且每個 volume type 可用的 mount option 也不盡相同，若是在不支援 mount option 的 volume type 中使用 mount option，就會無法掛載成功。

若要知道哪些 volume type 支援 mount option，可以參考[官網說明列表](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#mount-options)。


[StorageClass]: https://kubernetes.io/docs/concepts/storage/storage-classes/ "Kubernetes Storage Class"

## Node Affinity

在 PV 中可以設定 Node Affinity，目的在於限定特定的 node 才可以用來存取 volume，而使用帶有 Node Affinity 設定的 PV 的 pod，就僅會被分派到那些符合條件的 node 上面去。

## Phase

這個部份其實就類似 PV 的生命周期了，共包含4個階段：

- `Available`: 可被 PVC 使用的狀態

- `Bound`: 已經連結到特定的 PVC

- `Released`: PVC 已經被刪除，但 cluster 尚未進行 reclaim 的工作

- `Failed`: reclaim 失敗後會進入的狀態



Persistent Volume Claim
=======================

每一個 PVC 都包含了 spec 相關資訊，k8s 就是利用這些資訊協助尋找合適的 PV 並執行掛載工作，以下是一個 PVC 的範例：

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

## Access Modes

透過 `spec.accessModes` 設定，Access Mode 就跟 PV 的 Access Mode 相同，共有 `ReadWriteOnce`(`RWO`), `ReadOnlyMany`(`ROX`), `ReadWriteMany`(`RWX`) 三種。

## Volume Modes

透過 `spec.volumeMode` 設定，跟 PV 相同，volume mode 可以設定為 file system(`Filesystem`) 或是 raw block device(`Block`)。

## Resources

透過 `spec.resources` 設定，目前關於 storage 中可以設定的僅有 size 一項，以後可能還會支援 IOPS 或其他選項。

## Selector

透過 `spec.selector` 設定，就是透過 k8s 中的 label 機制，選擇合適的 PV 來繫結，可使用 `matchLabels` 做精準的配對，也可以透過 `matchExpressions` 作範圍性的配對。

## Class

透過 `spec.storageClassName` 設定，可用來指定已經存在的 storage class，因此若是設定了 storageClassName，就只有由指定的 storage class 所產生出來的 PV，才可以被此 PVC 繫結。

而 `spec.storageClassName` 若設定為空白，k8s 則會透過 label or resource request 來協助找到合適的 PV；但若是 k8s 管理者是將 [DefaultStorageClass](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass) admission plugin 開啟，則規則就會變得比較複雜些：

- 若管理者有設定 **DefaultStorageClass admission plugin**，PVC 就只能與透過 DefaultStorageClass 產生的 PV 做繫結

- 在既有的 storage class 中設定 annotation `storageclass.kubernetes.io/is-default-class: true` 的效果也等同於開啟 DefaultStorageClass admission plugin

- 若是有多個 storage class 都有設定 annotation `storageclass.kubernetes.io/is-default-class: true`，則 admission control 會禁止建立新的 PVC

- 若沒開啟 **DefaultStorageClass admission plugin** 也沒設定 annotation `storageclass.kubernetes.io/is-default-class: true`，則沒有設定 `spec.storageClassName`(設定空白字串也是相同意思) 的 PVC 就只能與沒有 class 資訊的 PV 繫結

- 若 PVC 同時設定了 selector & storageClassName，就只有同時符合兩個條件的 PV 可以與之繫結


Pod 如何使用 PV ?
===============

有了上面 PV & PVC 的概念後，那 pod 要如何使用 PV 呢? 答案是**必須透過 PVC，並將 PVC 當作 volume 使用**，以下是一個簡單的範例：

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      # 指定掛載的目錄
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      # 掛載 PVC "myclaim"
      persistentVolumeClaim:
        claimName: myclaim
```

因此，正確的繫結關係應該是 `Pod <--> PVC <--> PV`，所以已經成功與 PV 繫結的 PVC 是必要的。

此外，還有幾點觀念必須知道的：

- PVC 屬於 namespace level resource，因此 Pod 只能與同一個 namespace 中的 PVC 進行繫結 

- PV 屬於 cluster level resource，但若要支援 `ROX`(ReadOnlyMany) or `RWX`(ReadWriteMany)，就只限於同一個 namespace 中的 PVC 才可以



使用 raw Block Volume
====================

從 v1.13 開始以 beta 的形式支援 raw block volume，目前支援的 storage 還不是很多，詳細的清單可以查詢[官網列表](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#raw-block-volume-support)。

以下是一個使用 fiber channel block volume 的範例：

```yaml
# PV 定義
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  # 指定 volume mode 為 "Block"
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  # 以下是 fiber channel 連線相關資訊
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false


# PVC 定義
# 系統會自動將此 PVC 與上面的 PV(name='block-pv') 進行繫結
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi


# Pod 定義
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      # 將 block volume 掛載為 /dev/xvda
      # 因為是 block volume，因此要改成指定 devicePath
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```



如何撰寫可攜性高的 Storage Configuration ?
====================================== 

搞清楚上面 Pod, PVC, PV 相關概念後，若是真有弄清楚 k8s storage 的設計概念，就會大概有以下的概念：

- 並非每個 k8s cluster 都會有相同的 storage 設定

- 承上，這也是 storage class 會被設計出來的原因

因此要撰寫可攜性高的設定也就不是什麼太困難的事情，只要大概遵守以下幾個原則即可：

- 不要使用 volume，改用 PVC

- 不要包含 PV 的設定，因為在其他的 k8s cluster 很有可能會無法建立相同的 PV

- 給使用者指定 storage class (k8s 使用者應該知道有什麼 storage class 可用)，讓 PVC 使用 storage class

- 若使用者沒指定 storage class，那就應該會有 default storage class 的相關設定

- 若 PVC 一直處於無法與任何 PV 繫結的狀況，那就需要詢問 k8s cluster 管理者



Summary
=======

前一篇文章提到了 volume 的概念，在 k8s 中也同時支援了相當多的 cloud volume；而這裡指的 cloud volume 像是 `awsElasticBlockStore`, `gcePersistentDisk`...等，使用 clouod volume 的缺點在於使用者必須知道這些 volume 的設定細節，例如：volume ID, file system type...等資訊，在某些時候還是挺麻煩的，因此才會有 `persistent volume claim(PVC)` 這樣的 resource object 來抽象 & 隱藏這些設定的細節。

![Multiple Containers in a Pod](/blog/images/kubernetes/k8s_volume-pvc.png)

從上圖可以看出，在使用者使用 PVC 來取得 volume 的架構下，storage 管理者可以先準備好所謂的 `persistent volume pool`，而這個 volume pool 可以包含各式各樣的 volume(可來自 local or clouod)，而這些 volume 都會再被抽象一層變成 `persistent volume`，並賦予相關的屬性(例如：**accessModes**, **volume size**, **tag**)。

有了多個 PV 後，使用者就可以定義 PVC，並指定所需要的 PV 屬性為何，k8s 就會自動將配對成功的 PV 與 PVC 繫結在一起，並掛載到使用者所需要的 mount point 上，整個過程使用者完全不需要知道到底 volume 是從哪裡來，底層 storage 的細節也都完全不需要知道。

而這些抽象的設計，最終的目的就是在於要**讓使用者可以專注在開發上，而非 infra 的設定上**。



References
==========

- [Persistent Volumes - Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

- [DefaultStorageClass Admission Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)
 
