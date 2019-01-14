---
layout: post
title:  "[Kubernetes] Volume Overview"
description: "本篇文章的主題在介紹 Kubernetes 中的 Volume 的設計概念 & 使用方式"
date: 2019-01-15 04:15:00
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

使用 container 的開發者都知道，在 container 中的檔案不是永久存在的，隨著 container 的重啟，檔案就會隨之消失，當然這情況在 k8s pod 中也是相同；此外，在 k8s pod 中的多個 container 之間時常也會有檔案共享的需求，而為了解決這些問題，k8s 中提出了 `Volume` 這個抽象概念。

因此回到系統本質來看，`Volume` 的出現就是為了解決以下的問題：

- 檔案在 container 中無法永續存活

- pod 內多個 container 之間有檔案共享的需求

回到 container 來看，其實 Docker 也有 [Volume](https://docs.docker.com/storage/volumes/) 的概念，相較於 k8s Volume 比較沒那麼著重在管理層面。在一開始 Docker Volume 僅支援 local directory，到最近才慢慢開始有 [Volume plugin](https://docs.docker.com/engine/extend/legacy_plugins/#types-of-plugins) 來支援多種不同的 storage。

> 其實 volume plugin 的功能也不算是很完整 (例如：在 Docker v1.17 時，使用 volume plugin 時還無法傳入任何參數)

反之 k8s 對於 volume 的管理就相對嚴謹：

- volume lifetime 與 pod lifetime 相同 (會一直存活到 pod 中的 container 全部消失為止)

- 只要 pod 存在，資料就不會因為 container restart 而消失

- 基本上 volume 會以 directory 的型式存在於 pod 中，pod 中的每一個 container 都可以存取 (至於 directory 如何產生、資料實際存在哪裡則是由 volume type 決定，k8s 不會負責管理這個)

- 在 pod 中要使用 volume 就必須使用 `.spec.volumes` & `.spec.containers.volumeMounts` 兩個欄位進行設定

- 一個 volume 僅能給一個 pod 使用



Kubernetes 支援那些 Volume Type ?
===============================

k8s 現有支援的 volume 相當的多，有些來自 public cloud(例如：awsElasticBlockStore, azureDisk, gcePersistentDisk ... 等等)，也有些是屬於 on-premise 的(例如：glusterfs, nfs, iscsi ... 等等)，詳細列表可以參考[官網連結](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)。

以下就比較常會在 on-premise 環境碰到的 volume type 作個簡單說明：

## emptyDir

emptyDir 在使用上有以下幾點特性：

- emptyDir 會在 pod 被分配到特定的 node 後產生，一開始永遠是一個空的目錄

- emptyDir 的 life cycle 是跟著 pod + node 一起走，只要該 pod 持續的在原本的 node 上運行，emptyDir 就會保留著；相反的，只要兩個條件有任一消失，emptyDir 的資料也會跟著消失

- 若是有資料存進去 emptyDir，node 會把資料放在 RAM(設定 `emptyDir.medium` 為 `Memory`，emptyDir 會以 tmpfs 的方式提供) or local SSD/HD 上

- 存放在 SSD or HD，取決於 `/var/lib/kubelet` 目錄位於哪種硬碟上 (也就是說 emptyDir 會存放在 `/var/lib/kubelet` 目錄中)

- 每個在 pod 中的 container，可以將 emptyDir 掛載在不同的路徑

- 目前無法限制使用量

在官網中有提到一些 emptyDir 的 use case：

- scratch space, such as for a disk-based merge sort

- checkpointing a long computation for recovery from crashes

- holding files that a content-manager Container fetches while a webserver Container serves the data

由於這幾項 use case 我並沒有接觸過，因此就不太多著墨了。

## hostPath

若是希望 volume 新增後裏面就存放一些 node 上已經存在的資料，可以使用 hostPath。

通常這不是一般 pod 會用的 volume type，只有在一些情況下例外，例如：

- container 需要監控 host Docker 的內部運作細節，會需要掛載 host `/var/lib/docker` 資料夾

- container 中執行 cAdvisor，需要掛在 host `/sys` 目錄

- init container，用來產生另一個 container 需要的 host path

此外，hostPath 還細分成好幾種不同的類型。分別介紹如下：

| 類型 | 運作行為 |
|-----|---------|
| `(空字串)` | 掛載 host path 到 pod 之前，不會做任何檢查 |
| `DirectoryOrCreate` | 若 host path 指定的目錄不存在，則會新建一個目錄，並設定為權限 0755，目錄的 ownership & group 則是 kubelet |
| `Directory` | host path 指定的目錄必須存在 |
| `FileOrCreate` | 若 host path 指定的檔案不存在，則會新建一個檔案，並設定為權限 0755，目錄的 ownership & group 則是 kubelet |
| `File` | host path 指定的檔案必須存在 |
| `Socket` | host path 指定的 UNIX socket 必須存在 |
| `CharDevice` | host path 指定的 character device 必須存在 |
| `BlockDevice` | host path 指定的 block device 必須存在 |

> hostPath volume 的內容僅能被 root 修改，因此若有修改需求的話，就必須把 container 以 privileged mode 執行，或是修改 hostPath volume 的權限

以下是個 hostPath 使用範例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```

> 目前 hostPath 無法限制使用量，因此在使用時可能必須注意磁碟空間的問題


## local

`local` volume type 在 v1.10 時變成 beta 版本，主要用來表示在 host 上掛載的 local storage device，例如：disk, partition, directory ... 等等。

相較於 hostPath，`local` volume type 有以下幾點的不同：

- 只能以 PersistentVolume 的方式被使用，尚不支援 dynamic provisioning

- 由於 `local` 可以額外設定 **nodeAffinity**，因此 k8s scheduler 會協助挑選合適的 node 來使用

`local` volume type 看似比 `hostPath` 更有彈性些，但使用上還是有些地方必須注意，例如：

- local volume 的可用性依然是跟著 node 走，因為不一定每個 node 都有相同的 disk or partition 配置

- 如果 node 的狀態轉變成 unhealthy，同時也會造成 local volume 無法使用(因為資料就是存在於該 unhealthy node 上)，進而造成掛載 local volume 的 pod 也無法存取資料

- 承上，若 application 只能使用 local volume 的話，那就要考量到資料無法正常存取時的容錯設計

以下是個設定 local volume 的範例：

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 100Gi
  # 若要額外設定 volumeMode 這個欄位
  # 就必須要開啟 feature gate "BlockVolume"
  # 除了 Filesystem 之外，也可以設定為 "Block"(預設為 "Filesystem")
  # 來指定使用 raw block device
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  # nodeAffinity 為必要欄位
  # 目的是為了讓 k8s scheduler 可以協助找到合適的 node
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - example-node

---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

從上面的範例可以看出，local volume 是由 Storageclass 所產生，透過 `volumeBindingMode: WaitForFirstConsumer` 的設定，讓 PV 會在 PVC 需求實際發生(可能透過 node selector, pod affinity, pod anti-affinity ... 等設定)時才會產生。


## nfs

NFS volume type 就是利用外部的 NFS share(這部份要自己準備好，k8s 不負責這個) 來存放資料，有以下幾種特性：

- 跟 emptyDir 不同，NFS volume 中的資料不會因為 pod 被刪除而一併被刪除

- 可以預先把資料存放在 NFS volume 後再掛載

- NFS volume 可以同時被多個 pod 掛載 & 寫入，因此可以利用此特性來同時分享資料給多個 pod


## secret

secret 顧名思義就是存放機密性的資料，而 secret volume 有以下幾個特點：

- 要使用 secret volume 必須先定義 secret resource object 後，再掛載為 secret volume

- secret volume 會存在於 `/tmpfs` 中，也就是 RAM 中，不會存在於硬碟上

- 若透過 [subPath](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath) 掛載 secret volume，若是後續 secret volume 有更新，就不會反應到該 subPath 上

而 secret 的詳細使用方式可以參考[此篇文章](https://kubernetes.io/docs/concepts/configuration/secret/)。

## configMap

k8s 透過 configMap，提供使用者一個可以將任何設定相關資料放到 pod 中的方式，透過 `configMap` 關鍵字就可以將 configMap resource object 以 volume 的型式掛載到 pod 中使用。

以下是一個 **configMap -> File** 簡單的範例：

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
data:
  special.level: very
  special.type: charm

---
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      # 在 /etc/config 目錄中就會看到兩個檔案
      # /etc/config/special.level => very
      # /etc/config/special.type => charm
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # 指定要掛載的 configMap 名稱
        name: special-config
  restartPolicy: Never
```

然而 configMap 掛載到 pod 之後不一定只能以 file 的型式存在，也可以是以環境變數(Environment Variable)的型式存在，詳細的操作可以參考[官網文件](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)。



使用 subPath
============

有時候我們要掛載的目錄中可能包含了多個子目錄，而這些子目錄恰巧又分別被多個不同的 container 使用，此時就可以透過 `subPath` 的方式來簡化 volume 的設定，以下有個簡單的 LAMP 範例，在這個範例的 volume 中有兩個目錄，分別是 `mysql` & `html`，然後透過 subPath 的方式分別掛載到 pod 中的不同 container 上：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-lamp-site
spec:
    containers:
    - name: mysql
      image: mysql
      env:
      - name: MYSQL_ROOT_PASSWORD
        value: "rootpasswd"
      # 掛載 volume 中的 "mysql" subPath
      volumeMounts:
      - mountPath: /var/lib/mysql
        name: site-data
        subPath: mysql
    - name: php
      image: php:7.0-apache
      # 掛載 volume 中的 "html" subPath
      volumeMounts:
      - mountPath: /var/www/html
        name: site-data
        subPath: html
    # 包含多個目錄(mysql & html)的 volume 設定
    volumes:
    - name: site-data
      persistentVolumeClaim:
        claimName: my-lamp-site-data
```

## 搭配 Expanded Environment Variables

透過 Expanded Environment Variables　的搭配，可以在 YAML 定義檔中，使用與 pod 相關的識別資訊，可以讓 subPath 在使用上更加的彈性，
以下是一個簡單的範例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: container1
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.name
    image: busybox
    command: [ "sh", "-c", "while [ true ]; do echo 'Hello'; sleep 10; done | tee -a /logs/hello.txt" ]
    volumeMounts:
    - name: workdir1
      mountPath: /logs
      subPath: $(POD_NAME)
  restartPolicy: Never
  volumes:
  - name: workdir1
    hostPath:
      path: /var/log/pods
```

透過 `$(POD_NAME)` 讓不同的 pod 的 log 可以存放在不同的目錄中，並以 pod name 作為區別，藉此可以提升 debug 的困難度。



Out-of-Tree Volume Plugins
==========================

[k8s 支援了相當多的 volume type](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)，市面上常看到的 storage 基本上都包含了，而使用這些 storage 所需要的程式碼都被整合到 k8s 的 code repository 中，這表示都是經過驗證，使用者不需要安裝任何額外的 plguin 就可以使用的，但程式碼要進入到 k8s code repository 也代表的要進行嚴格的審查。

但若是其他的 storage 廠商也希望可以跟 k8s 整合時該怎麼做呢? 答案就是透過 `Container Storage Interface (CSI)` & `Flexvolume` 這兩種 volume plugin。

透過 CSI & Flexvolume，storage 廠商可以自行開發 volume plugin 而不需要經過 k8s 社群的審查(不代表不嚴謹)，可以獨立的開發而不受影響，並佈署在 k8s 上成為一個擴充套件來使用，而關於 out-of-tree volume plugin 如何使用，可以參考[官網文件](https://github.com/kubernetes/community/blob/master/contributors/devel/flexvolume.md)。

## FlexVolume

從 v1.2 版開始，k8s 提供了 FlexVolume 的方式讓 storage 廠商可以自行開發可以與 k8s 整合的 volume plugin(driver)。k8s 會以 exec-based 的方式與 volume driver 溝通，因此相對應的 FlexVolume driver binary 就要預先放到每個 node 預定定義好的目錄下。

而 FlexVolume 在使用上也會有些限制與問題存在，例如：

- FlexVolume 為了要安裝 driver 相關檔案到每個 node 上，需要 root 存取權限

- FlexVolume driver 的相依性套件也要預先準備在每個 node 上，%這代表著也要 root 權限來安裝

因此現在 k8s 社群建議使用 CSI，而非 FlexVolume。

## Container Storage Interface (CSI)

顧名思義，[CSI](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md) 就是為了 container orchestration systems(例如：k8s，但不僅限於 k8s) 定義了一系列的標準 interface，讓 container 可以透過這些 interface 與 storage 互動。

而 CSI 在 k8s 的支援上，從 v1.9 為 alpha，v1.10 為 beta，最後到 v1.13 正式 GA。

關於 CSI 的使用細節，由於目前還沒使用過，所以這個部份就先跳過，但還是有些額外的資訊需要注意：

- k8s v1.13 後開始要廢除 CSI spec 0.2 & 0.3 版本的支援

- CSI driver 不一定會與所有的 k8s 版本相容，使用前要注意一下

- 從 k8s v1.11 開始，CSI 開始支援 **raw block volume**，目前為 `alpha`，需要額外開啟 `BlockVolume` & `CSIBlockVolume` 兩個 feature gate

- 無法在 pod 定義中直接使用 CSI volume，必須透過 `PersistentVolumeClaim` 才可以使用

- 若要自行開發 CSI plugin，可以參考 [Kubernetes CSI Developer Reference](https://kubernetes-csi.github.io/docs/)



Mount Propagation
=================

Mount Propagation 允許可以將掛載在某個 container 中的 volume 分享給在同一個 pod 中的 container，甚至是該 pod 所在的 node 的一種特殊功能。

要使用 Mount Propagation 功能，必須在 `Container.volume.mounts` 中加上 `propagation` 參數，而可用的選項有以下幾種：

- `None`：已經掛載到 container 的 volume，即使從 host 端在其子目錄上加上其他 mount，container 也不會收到；在 container 中建立的 mount 也不會讓 host 看到(**default mode**，等同於 Linux 中的 `private` mount propagation)

- `HostToContainer`：與上面較為不同，container 可以收到 host 後續在子目錄加上的 mount；但在 container 中建立的 mount 不會讓 host 看到(等同於 Linux 中的 `rslave` mount propagation)

- `Bidirectional`：除了有 `HostToContainer` 的行為之外，在 container 中掛載的其他內容也會在 host 上出現 (等同於 Linux 中的 `rshared` mount propagation)

`Bidirectional` 模式在使用上其實蠻危險的，因此僅限於 Priviledged Container 中使用，此外在 container 中建立的任何 volume 都必須在 container 結束時一併移除。

最後，若真的要使用到 Mount Propagation 的功能，必須在 Docker 中加上 `MountFlags=shared` 設定，並重新啟動 Docker service。



References
==========

- [Volumes - Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/)

- [(39) Kubernetes Volumes 1: emptydir, NFS, YAML, volumes, and intro to Persistent Volume Claims - YouTube](https://www.youtube.com/watch?v=VB7vI9OT-WQ)

- [(39) Kubernetes Volumes 2: Understanding Persistent Volume (PV) and Persistent Volume Claim (PVC) - YouTube](https://www.youtube.com/watch?v=OulmwTYTauI)

- [(39) Kubernetes Volumes 3: How things connect - YouTube](https://www.youtube.com/watch?v=X6Vkz-ny574)

- [Storage Classes - Kubernetes](https://kubernetes.io/docs/concepts/storage/storage-classes/#local)

- [kubernetes-sigs/sig-storage-local-static-provisioner: Static provisioner of local volumes](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner)

- [community/volume-plugin-faq.md at master · kubernetes/community](https://github.com/kubernetes/community/blob/master/sig-storage/volume-plugin-faq.md)

- [談談k8s1.12新特性--Mount propagation(掛載命名空間的傳播) - k8s小店 - SegmentFault 思否](https://segmentfault.com/a/1190000016567617)