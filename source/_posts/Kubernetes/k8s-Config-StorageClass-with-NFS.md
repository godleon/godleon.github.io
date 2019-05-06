---
layout: post
title:  "[Kubernetes] 設定 StorageClass (以 NFS 為例)"
description: "本篇文章介紹如何在 Kubernetes 上設定 StorageClass，並以 NFS 作為後端的 storage"
date: 2018-10-26 16:10:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Storage
  - CKA
  - CKA Storage
  - NFS
---

2019-05-06 更新
==============

目前在 NFS storage 的部份，已經變成 `NFS Provisioner` & `NFS-Client Provisioner` 兩種了：

- [NFS Provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs)：會在 k8s 中啟動一個 NFS server 來使用

- [NFS-Client Provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)：這部份跟下面原本介紹的相同，使用者必須先設定好一個外部的 NFS server，然後將這個 plugin 指定過去，它就會在需要時在上面建立目錄並存放資料。

> 若有 NFS external storage 的需求，可以參考上面新的連結來設定，新的方法已經改用 Helm 來管理，佈署安裝的流程變得相當簡單


前言
===

之前有介紹過 StorageClass 搭配 GlusterFS + Heketi 作為後端的 storage，但 GlusterFS 可能對很多人來說還是有點複雜，因此想說介紹一個比較容易入門的 storage，學習 k8s 的過程才不會覺得很艱難。

只要用過 Linux，大概 NFS 幾乎就會是個必學的服務，因此這邊要介紹以 NFS 作為 StorageClass 後端 storage 的設定方式，讓 k8s 可以動態的在 NFS share 上產生所需要 volume 來使用。


運作原理說明
=========

基本上要正確設定 StorageClass + NFS，大概要準備以下幾個東西：

1. 一個可用的 NFS share

2. `NFS provisioner`：主要有兩個工作，一個是實際在 NFS share 中建立 volume(其實就是一般的 directory)，另一個則是建立 PV，並與 NFS volume 作繫結

3. `Service Account`：這是用來管控 NFS provisioner 在 k8s 中可以運行的權限

4. `StorageClass`：負責建立 PVC 並呼叫 NFS provisioner 進行設定工作，並讓 PVC 與 PV 繫結

詳細的運作流程可以參考下圖：

![Kubernetes Persistent Volume Provisioning](/blog/images/kubernetes/dynamic-volume-provision.jpg)



設定過程
======

## 設定 NFS share

這個部份就留給大家自己去作了，網路上有非常多的教學可以查。

假設這裡使用以下的 NFS share：

- IP：`10.1.2.3`

- Export Path: `/var/k8s-nfs-share`


## 設定 Service Account & 對應權限

如果要精準的管理權限，那就必須要自訂一個 service account 並搭配 k8s 中的 RBAC 機制來進行設定。

在以下的範例中會完成兩件事情：

1. 新增 service account `nfs-client-provisioner`，作為 NFS provisioner 的權限來源

2. 在 k8s 中有賦予 service account 足夠的權限處理跟 StorageClass & PersistentVolumeClaim 相關的工作 (透過 **Role** + **RoleBinding** + **ClusterRole** + **ClusterRoleBinding**)

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

## 安裝 NFS provisioner

NFS provisioner 負責以下工作：

1. 在 NFS share 中產生 volume(directory)

2. 新增 PV

3. 告知 PVC 已經完成 PV 的設定，讓 PVC 與 PV 繫結

使用以下的設定檔佈署 NFS provisioner


```yaml
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: my-nfs-provisioner
            - name: NFS_SERVER
              value: 10.1.2.3
            - name: NFS_PATH
              value: /var/k8s-nfs-share
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.1.2.3
            path: /var/k8s-nfs-share
```

## 建立 StorageClass

透過以下設定檔，建立 StorageClass 來使用上面所佈署好的 NFS provisioner：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-nfs-storage
provisioner: my-nfs-provisioner
parameters:
  archiveOnDelete: "false"
```

以上的設定都佈署完之後，就可以在系統中看到 StorageClass 出現：

```bash
$ kubectl get storageclass
NAME                         PROVISIONER               AGE
... (略)
my-nfs-storage               my-nfs-provisioner        5h10m
```


驗證佈署是否成功
=============

驗證的步驟如下：

1. 建立 PVC，指定上面所設定好的 StorageClass

2. 建立 Pod，使用上一個步驟設定好的 PVC

## 建立 PVC

使用下面的設定來建立一個測試用的 PVC：

```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "my-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

如果可以從系統中看到 PVC Status 為 `Bound` 就表示建立成功了：

```bash
# 檢視 PVC
$ kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-claim   Bound    pvc-052b8e95-d8f4-11e8-9dda-54ab3a09ec1d   1Mi        RWX            my-nfs-storage        5s

# 檢視 PV
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                 STORAGECLASS          REASON   AGE
pvc-052b8e95-d8f4-11e8-9dda-54ab3a09ec1d   1Mi        RWX            Delete           Bound    default/test-claim                                                    managed-nfs-storage            2m46s
```

上面的訊息表示 StorageClass 成功完成了以下幾件工作：

1. 在 NFS share 上建立 volume

2. 建立 PV，並與上述的 NFS volume 作繫結

3. 將 PVC 與 PV 繫結


## 建立 Pod

最後就是建立 Pod 並指定掛載 PVC，而以下的設定如果成功在 NFS 上建立檔案，這個 Pod 的狀態就會顯示完成，反之則會顯示失敗：

```yaml
---
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: gcr.io/google_containers/busybox:1.24
    command:
      - "/bin/sh"
    args:
      - "-c"
      - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
      - name: nfs-pvc
        mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```

檢視一下 Pod 產生的結果：

```bash
$ kubectl get pod
NAME                                      READY   STATUS      RESTARTS   AGE
test-pod                                  0/1     Completed   0          9s
```

看到 Pod 的 Status 為 Completed 表示 StorageClass + NFS 成功設定完成囉!



References
==========

- [Kubernetes NFS-Client Provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)