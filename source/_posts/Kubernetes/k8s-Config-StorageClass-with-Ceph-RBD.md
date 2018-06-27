---
layout: post
title:  "[Kubernetes] 設定 StorageClass (以 Ceph RBD 為例)"
description: "本篇文章介紹如何在 Kubernetes 上設定 StorageClass，以 Ceph RBD 為例"
date: 2018-06-27 10:30:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Ceph
---

前言
===

安裝完 k8s cluster 後，接著緊接下來要解決的問題就是 "資料儲存" 的問題。

由於 container 的生命周期跟 VM 不同，而且在 k8s cluster 上，container 可能會因為某些原因在不同的機器之間遷移，因此"**資料儲存**"這件事情就變得要謹慎考慮。

此篇文章將會介紹如何使用 Ceph RBD 為 k8s cluster 提供 block device 的 persistent volume。此外由於本文是著重在 k8s 與 Ceph 的整合，因此就不會再 k8s & Ceph 的安裝著墨，而是直接切入整合時的設定。



環境說明
======

Ceph: `Luminous`

Kubernetes：`1.10.4`



設定 Ceph cluster
================

首先要在 Ceph cluster 完成以下幾件事情：

1. 建立一個給 k8s cluster 用的 pool (例如： `kube`)

2. 設定 pool 相關參數 (replica 數量，Quota .... etc)

3. 新增使用該 pool 的使用者帳號(例如： `kube`)，並指定該 pool 的存取權限

接著我們透過以下指令完成上面的工作：(必須以 Ceph 管理者的權限進行設定)

```bash
# 建立名稱為 "kube" 的 pool
$ ceph osd pool create kube 128

# 設定停用 replication 功能
$ ceph osd pool set kube size 1

# 設定 pool 最大容量為 10GB
$ ceph osd pool set-quota kube max_bytes $((10 * 1024 * 1024 * 1024))

# 建立使用該 pool 的使用者帳號
$ ceph auth get-or-create client.kube mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=kube'
[client.kube]
	key = [YOUR_KEY]

# 取得 Ceph admin keyring
$ ceph auth get-key client.admin | base64
QVFEcytKaGF6UVlmRmhBQWJOZTNaZjYvaFVFdkhpRVVQejJOWFE9PQ==

# 取得 Ceph user keyring
$ ceph auth get-key client.kube | base64
QVFDTmp6RmJONy9wRkJBQUZxN3QzQnVLaTJpb2YwR0dDZEJ2dEE9PQ==
```

為了測試接下來要示範的 persistent volume，再多建立一個名稱為 `ceph-image` 的 RBD image：

> rbd create kube/ceph-image --size 4096 --image-format 2 --image-feature layering

**Ceph 預設的 Crush Map 設定會讓 k8s 與 Ceph 的整合發生問題，因此要先調整 Crush Map，詳情可以參考下方的[障礙排除](#障礙排除)。**



設定 Kubernetes
==============

## Prerequisite

由於在 mount Ceph RBD image 之前，kubelet 會檢查 image 的狀態，因此在 k8s worker node 都需要額外進行以下調整：

1. 安裝 `ceph-common` 套件

2. 將 ceph admin keyring 放到 `/etc/ceph/ceph.keyring`


## 獨立的 namespace

為了讓系統環境維持乾淨，接著建立一個獨立的 namespace(名稱為 `ceph-rbd-pv-lab`) 來進行以下測試：

```bash
# 取得目前的 context 為 "admin-cluster.local"
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://10.103.10.51:6443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: admin-cluster.local
  name: admin-cluster.local
current-context: admin-cluster.local
....(略)

# 建立 namespace
$ kubectl create namespace ceph-rbd-pv-lab

# 指定預設的 namespace
$ kubectl config set-context "admin-cluster.local" --namespace=ceph-rbd-pv-lab
```



使用 Ceph RBD 作為 Persistent Volume
===================================

準備檔案 `ceph-pv.yaml`，內容如下：

```yaml
# 定義 secret，儲存 Ceph admin keyring
---
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-admin
data:
  key: QVFEcytKaGF6UVlmRmhBQWJOZTNaZjYvaFVFdkhpRVVQejJOWFE9PQ==

# 定義 PV，指定到上面建立的 ceph rbd image
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      - 10.103.2.24:6789
    pool: kube
    image: ceph-image
    user: admin
    secretRef:
      name: ceph-secret-admin
    fsType: ext4
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle

# 定義 PVC，與上面的 PV 進行繫結
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-claim
spec:
  accessModes:     
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

# 定義使用 PVC 的 pod
---
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod1           
spec:
  containers:
  - name: ceph-busybox
    image: busybox          
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-vol1       
      mountPath: /usr/share/busybox 
      readOnly: false
  volumes:
  - name: ceph-vol1         
    persistentVolumeClaim:
      claimName: ceph-claim
```

套用以上設定：

> kubectl create -f ceph-pv.yaml

過一陣子檢查 pod 狀態：

```bash
# pod 已經處於正常運作的狀態
$ kubectl get pod
NAME        READY     STATUS    RESTARTS   AGE
ceph-pod1   1/1       Running   0          2m

# 成功掛載在 /usr/share/busybox 目錄下
$ kubectl exec -it ceph-pod1 -- df -h | grep '/usr/share/busybox'
/dev/rbd0                 3.8G      8.0M      3.6G   0% /usr/share/busybox
```



使用 StorageClass 動態生成 Persistent Volume
==========================================

## What's StorageClass?

不曉得有沒有人想過以下問題：

- 如果我常常需要產生 PV，又要刪除它呢 ?

- 可以不要一直去煩 storage manager 嗎? 每次產生新的 PV 都需要請他建立對應的 RBD image

- 產生 PV 的動作可以自動完成嗎 ?

- 每次都要手動建立 RBD image 哪有 cloud native ?

而 [Kubernetes StorageClass][K8S StorageClass] 就是以上問題的解答，其流程如下圖：

![Kubernetes Persistent Volume Provisioning](/blog/images/kubernetes/dynamic-volume-provision.jpg)

1. 設定 persistent volume provisioner (Ceph RBD 已經被 k8s 支援)

2. k8s ckuster 管理者建立 Storage Class，並指定要使用的 PV provisioner(這裡使用 `kubernetes.io/rbd`)

3. 使用者建立 PVC，指定要使用的 [StorageClass][K8S StorageClass]

4. [StorageClass][K8S StorageClass] 使用 provisioner 在實際的 storage 上產生 volume，並建立 PV 與其繫結

5. 將 PV 與 使用者 PVC 進行繫結

6. 使用者建立 pod，並使用 PVC 取得外部的儲存空間

> **以上的重點在於，管理者再也不用辛苦的手動建立儲存空間，並設定 PV 與其繫結了，這個部份都交給 StorageClass 自動處理**


## 在 Kubernetes 上設定 StorageClass

準備以下設定檔，名稱為 `ceph-storageclass.yml`：

```yaml
---
# admin secret 要定義在 cluster level (對應 PV 的生成)
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-admin
  namespace: kube-system
data:
  key: QVFEcytKaGF6UVlmRmhBQWJOZTNaZjYvaFVFdkhpRVVQejJOWFE9PQ==
type: kubernetes.io/rbd

---
# user secret 要定義在 namespace level (對應 PVC 的生成 & 繫結)
apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret-user
data:
  key: QVFDTmp6RmJONy9wRkJBQUZxN3QzQnVLaTJpb2YwR0dDZEJ2dEE9PQ==
type: kubernetes.io/rbd

---
# user secret 要定義在 namespace level (對應 PVC 的生成 & 繫結)
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ceph-dynamic
  annotations:
     storageclass.beta.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/rbd
parameters:
  monitors: 10.103.2.24:6789
  adminId: admin
  adminSecretName: ceph-secret-admin
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-user
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

套用上述設定檔即可完成：

> kubectl create -f ceph-storageclass.yml

從以上可以看出，其實 [StorageClass][K8S StorageClass] 的內容就是在實際的 storage 進行操作所必要的資訊(包含 IP，帳號、密碼.....等等)，敏感資訊的部份就透過 **secret** 放進去。

此外，在建立 [StorageClass][K8S StorageClass] 的時候，就可以額外帶上建立 RBD 時的部份參數，以上面的設定為例，包含了：

- fsType: ext4

- imageFormat: "2"

- imageFeatures: "layering"

如此一來當 RBD image 被建立時，就會是以在 [StorageClass][K8S StorageClass] 中所定義的規格去產生。


## 驗證動態 Persistent Volume 的生成

接著來驗證 persistent volume 的自動生成，透過以下的設定(名稱為 `storageclass-test.yml`)來完成：


```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ceph-dynamic
  resources:
    requests:
      storage: 2Gi

---
apiVersion: v1
kind: Pod
metadata:
  name: ceph-pod1           
spec:
  containers:
  - name: ceph-busybox
    image: busybox          
    command: ["sleep", "60000"]
    volumeMounts:
    - name: ceph-vol1       
      mountPath: /usr/share/busybox 
      readOnly: false
  volumes:
  - name: ceph-vol1         
    persistentVolumeClaim:
      claimName: ceph-pvc
```


```bash
# 套用上面的設定
$ kubectl create -f storageclass-test.yml

# 檢視 PVC 設定
$ kubectl get pvc
NAME       STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ceph-pvc   Bound     pvc-426e7cc8-79b1-11e8-81d0-56277529c641   2Gi        RWO            ceph-dynamic   6s

# 確認 pod 已經正常的執行中
$ kubectl get pods
NAME        READY     STATUS    RESTARTS   AGE
ceph-pod1   1/1       Running   0          20m

# 檢視與上面的 PVC 繫結的 PV 內容
$ kubectl describe pv/pvc-426e7cc8-79b1-11e8-81d0-56277529c641
Name:            pvc-426e7cc8-79b1-11e8-81d0-56277529c641
....(略)
StorageClass:    ceph-dynamic
Status:          Bound
Claim:           ceph-rbd-pv-lab/ceph-pvc
....(略)
Source:
    Type:          RBD (a Rados Block Device mount on the host that shares a pod's lifetime)
    CephMonitors:  [10.103.2.24:6789]
    RBDImage:      kubernetes-dynamic-pvc-4273f5b9-79b1-11e8-9115-3a01c1a41f09
    FSType:        ext4
    RBDPool:       kube
    RadosUser:     kube
    Keyring:       /etc/ceph/keyring
    SecretRef:     &{ceph-secret-user }
    ReadOnly:      false
Events:            <none>
```

可以看到有一個 RBD image `kubernetes-dynamic-pvc-4273f5b9-79b1-11e8-9115-3a01c1a41f09` 被產生出來。

接著回到 Ceph cluster 上查詢一下是不是有相對應的 object 被產生出來：

```bash
# 的確有個 "kubernetes-dynamic-pvc" 開頭的 object 被產生出來了!
$ rbd -p kube ls
ceph-image
kubernetes-dynamic-pvc-4273f5b9-79b1-11e8-9115-3a01c1a41f09

# 檢視 RBD image 的細節
$ rbd info kube/kubernetes-dynamic-pvc-4273f5b9-79b1-11e8-9115-3a01c1a41f09
rbd image 'kubernetes-dynamic-pvc-4273f5b9-79b1-11e8-9115-3a01c1a41f09':
	size 2048 MB in 512 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.bdce80238e1f29
	format: 2
	features: layering
	flags: 
	create_timestamp: Wed Jun 27 10:24:46 2018
```

從上面 RBD image 的內容可以看出，[StorageClass][K8S StorageClass] 的確是有幫我們根據上面的規格產生出一個 RBD image 與 PV，並相互繫結後，最後再與 PVC 繫結。



障礙排除
======

## feature set mismatch (1)

在 pod event 中看到以下的訊息：

```txt
MountVolume.WaitForAttach failed for volume "ceph-pv" : rbd: map failed exit status 110, rbd output: 2018-06-26 05:44:18.264219 7f50a90e4100 -1 did not load config file, using default settings.

rbd: sysfs write failed

In some cases useful info is found in syslog - try "dmesg | tail" or so.
rbd: map failed: (110) Connection timed out
```

於是到 worker node 上執行 `dmesg`，看到了很多類似以下的訊息：

```txt
........ (略)
libceph: mon0 10.103.2.24:6789 feature set mismatch, my 106b84a842a42 < server's 40106b84a842a42, missing 400000000000000
libceph: mon0 10.103.2.24:6789 missing required protocol features
........ (略)
```

### 解決方法

這個問題只要修改 Ceph 的 crush map 即可，修改的方式到 Ceph 的 admin node 上執行以下指令：

```bash
$ ceph osd crush tunables legacy

$ ceph osd crush reweight-all
```

> 其實也可以將 kernel 升級到 4.5 以上解決此問題


## feature set mismatch (2)

這個問題同樣是 **feature set mismatch**，會出現以下訊息：

```txt
rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable".
```

這個問題需要在建立 RBD image 時就要設定好特定的 feature，因此回到 Ceph admin node 上，先以正常的方式建立 RBD image

```bash
# 以正常的方式建立 rbd image
$ rbd create kube/ceph-image --size 1024

# 顯示 image feature
$ rbd info kube/ceph-image
rbd image 'ceph-image':
	size 1024 MB in 256 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.ba3aa02ae8944a
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags: 
	create_timestamp: Tue Jun 26 14:08:35 2018
```

其中 `exclusive-lock`, `object-map`, `fast-diff`, `deep-flatten` 這些 feature 都要拿掉才行....

因此我們改成以下指令建立 RBD image：

> rbd create kube/ceph-image --size 1024 --image-format 2 --image-feature layering

上面的問題就解決了!



References
==========

- [https://hk.saowen.com/a/edc90fdb04b4cc267bcbcbae056e2a8b0e9d73ac05c81d05c1f72bf295112a13](kubernetes持久化存儲Ceph RBD - 掃文資訊)

- [Kubernetes儲存之Persistent Volumes簡介 - 掃文資訊](https://tw.saowen.com/a/aa7346a600499cbde6551ef13c685b37b4148c6d5b3d03dcff6b48d431c7a10a)

- [Storage Classes - Kubernetes](https://kubernetes.io/docs/concepts/storage/storage-classes/#ceph-rbd)

- [Ceph Pool操作总结 - int32bit的博客 | int32bit Blog](http://int32bit.me/2016/05/19/Ceph-Pool%E6%93%8D%E4%BD%9C%E6%80%BB%E7%BB%93/)

- [User Management — Ceph Documentation](http://docs.ceph.com/docs/mimic/rados/operations/user-management/)

- [rados – rados object storage utility — Ceph documentation](http://docs.ceph.com/docs/argonaut/man/8/rados/)

- [Bug #1716735 “ceph with luminous can't be used with KRBD with Xe...” : Bugs : OpenStack ceph-mon charm](https://bugs.launchpad.net/charm-ceph-mon/+bug/1716735)

- [Feature Set Mismatch Error on Ceph Kernel Client - CephNotes](http://cephnotes.ksperis.com/blog/2014/01/21/feature-set-mismatch-error-on-ceph-kernel-client/)

- [Ceph Luminous CRUSH map 400000000000000 問題 | KaiRen's Blog](https://kairen.github.io/2018/02/11/ceph/luminous-crush-issue/)


[K8S StorageClass]: https://kubernetes.io/docs/concepts/storage/storage-classes "Kubernetes Storage Class"
