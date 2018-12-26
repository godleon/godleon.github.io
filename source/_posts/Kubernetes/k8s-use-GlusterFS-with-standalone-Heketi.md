---
layout: post
title:  "[Kubernetes] 快速安裝 GlusterFS + Heketi 並與 StorageClass 搭配使用"
description: "此篇文章介紹如何快速的安裝 GlusterFS + Heketi 並與 Kubernetes 中的 StorageClass 搭配使用"
date: 2018-09-26 14:50:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Storage
  - CKA
  - CKA Storage
  - GlusterFS
---


緣由
===

由於最近在花些時間研究 k8s，但一直研究到 [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) 時，發現對於 StatefulSet 來說，**persistet volume 是必備的**，而且若是有設定 replica 的部份，就演變成 **動態產生 persistent volume 是必備** 的。

動態產生 PV 這個需求的確是存在的，試想如果有很多不同的 workload or job 都在 k8s 上運行，但同時都需要保存資料時，每次都請 storage manager 開好 PV 是很麻煩的，因此把這個部份的自動化的需求就產生了；而動態產生 **PV**(persistent volume) 這件工作，在 k8s 是由 [Storage Class](https://kubernetes.io/docs/concepts/storage/storage-classes/) 這個機制來完成；於是仔細看了一下 provisioner list，決定使用 GlusterFS 來作為 persistent storage 的測試對象。

> 選擇 GlusterFS 的原因是因為同時支援 ReadWriteOnce/ReadOnlyMany/ReadWriteMany 三種 [Access Mode](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)，同時也支援 Quota & Snapshot 等功能


What is GlusterFS?
==================

這個部份就不多著墨，詳細的內容可以參考以下網站：

- [Gluster 架構 | Gluster Storage System](http://www.l-penguin.idv.tw/book/Gluster-Storage_GitBook/gluster_intra/gluster_arch.html)

上面有提到關於一些 GlsuterFS 重要的概念，例如：**Peer**, **Brick**, **Volume** .... 等等。

若對 GlusterFS 的運作細節跟優化很有興趣，可以參考[官方的文件說明](https://docs.gluster.org/en/latest/)。



What is Heketi?
===============

首先看看以下官網的說明：

> Heketi provides a RESTful management interface which can be used to manage the life cycle of GlusterFS volumes. With Heketi, cloud services like OpenStack Manila, Kubernetes, and OpenShift can dynamically provision GlusterFS volumes with any of the supported durability types. Heketi will automatically determine the location for bricks across the cluster, making sure to place bricks and its replicas across different failure domains. Heketi also supports any number of GlusterFS clusters, allowing cloud services to provide network file storage without being limited to a single GlusterFS cluster.

首先關於 GlusterFS，必須知道以下幾件事情：

- GlusterFS 並沒有提供管理用的 REST API service

- GlusterFS 並沒有提供 Failure Domain 的設計

而以上兩個在 GlusterFS 中沒有的設計，Heketi 則是在更上層的抽象設計中，把這兩個功能補足了! 因此細部檢視 Heketi 的功能，就包含了：

- 提供 REST API service 作為管理 GlusterFS 之用

- 可以讓 OpenStack Manila, Kubernetes, OpenShift 等平台動態產生 volume 並掛載使用

- 可同時管理多個 GlusterFS cluster，可讓使用者把資料放到不同的 GlusterFS cluster 中

- 可根據需求，自動化的在分配 brick 時，分散到不同的 failure domain，提供資料可用性



環境需求
======

## Kubernetes

假設已經有一個正常運行的 Kubernetes


## GlusterFS

GlusterFS 的環境安裝過程中，需要 2 個 node 來進行安裝，規格如下：

- OS: `Ubuntu 18.04 (Bionic)`

- CPU & RAM: 由於只是測試環境....因此這邊只給 4 vcore + 4GB RAM

- Disk: `3 (1 for OS, 2 for Data)` (**/dev/vda**, **/dev/vdb**, **/dev/vdc**)

> 以上的 node 建議需要設定成搭配 SSH pribate key 進行 passwordless 登入


## Heketi

以下安裝過程中，會將 Heketi 安裝在 GlusterFS cluster 的第一個 node 上，單獨以 docker container 的方式運行


## 安裝用機器(Bootstrapper)

這台機器可以是實體機 or VM，使用的環境必須為 `Ubuntu 18.04` or `Ubuntu 16.04`



安裝事前準備
==========

## 取得安裝程式

首先登入到 bootstrapper，輸入以下指令：

```bash
$ sudo apt-get update && sudo apt-get -y install git
$ cd /tmp
$ git clone https://github.com/godleon/k8s-lab.git
$ cd /tmp/k8s-lab/GlusterFS-with-standalone-Heketi/ansible
```

## 修改安裝參數檔

以下的安裝參數檔請按照自己的環境進行修改：

### $(pwd)/inventory

此檔案包含了用來安裝 GlusterFS 的兩個 node 的名稱 & IP 資訊

### $(pwd)/group_vars/all

這裡包含三種資訊：

1. 每個 node 的 **SSH 連線資訊**(因為安裝程式是使用 Ansible 所開發，因此正確的 SSH 連線資訊是必備的)

2. GlusterFS 的**版本** (預設為 `4.1`)

3. 每個 node 上用來儲存資料的 **device name** (也可以是 **partition name**)


```yaml
# Login information of GlusterFS nodes
ansible_python_interpreter: /usr/bin/python3
ansible_user: "ubuntu"
ansible_become_user: root
ansible_ssh_private_key_file: /ansible/ssh-privkey/id_rsa

# GlusterFS version
glusterfs_version: "4.1"

# disk list for storing data (not OS disk)
storage_devs:
- "/dev/vdb"
- "/dev/vdc"
```

## 準備 SSH Private Key

將用來登入 GlusterFS node 用的 ssh private key 放到 `$(pwd)/ssh-privkey` 目錄中，並命名為 **id_rsa**。



安裝程式做了那些事情?
=================

在開始安裝之前，先說明一下整個安裝程式到底完成了那些事情：

1. 在 2 個乾淨的 node 上安裝 GlusterFS，並設定 peer(這是一個 GlusterFS cluster node 之間相互認識的一個過程)

2. 在第一個 node 上安裝 docker

3. 透過 docker 啟動 Heketi service container

4. 匯入 cluster topology 資訊到 Heketi service
> topology 資訊包含 hostname, IP, 用來儲存 data 的 device(or partition name), failure domain ... 等資訊

5. 取得新建立的 cluster ID 並顯示

有了最後一個步驟取得的 cluster ID 後，才可以用來設定 k8s StorageClass for GlusterFS。



開始安裝
======

執行以下命令開始安裝：

```bash
$ cd /tmp/k8s-lab/GlusterFS-with-standalone-Heketi
$ ./start.sh
```

接著安裝程式會開始執行，大概需要 3~5 分鐘完成所有的安裝流程，而安裝完成後會出現類似以下訊息：

```txt
....(略)
TASK [Display topology import result] ******************************************
Wednesday 26 September 2018  05:49:15 +0000 (0:00:10.832)       0:02:01.148 *** 
skipping: [gfs02]
ok: [gfs01] => {
    "import_result.stdout_lines": [
        "Creating cluster ... ID: 6190e1bfdb33d611af35361d7f72625b", 
        "\tAllowing file volumes on cluster.", 
        "\tAllowing block volumes on cluster.", 
        "\tCreating node 10.107.13.101 ... ID: 18db1bc118707ed70e26213b484068c2", 
        "\t\tAdding device /dev/vdb ... OK", 
        "\t\tAdding device /dev/vdc ... OK", 
        "\tCreating node 10.107.13.102 ... ID: 900110afd1dff7c89c8fd7ef427f4337", 
        "\t\tAdding device /dev/vdb ... OK", 
        "\t\tAdding device /dev/vdc ... OK"
    ]
}
....(略)
TASK [Display Cluster ID] ******************************************************
Wednesday 26 September 2018  05:49:16 +0000 (0:00:00.116)       0:02:01.970 *** 
skipping: [gfs02]
ok: [gfs01] => {
    "msg": "Cluster ID = 6190e1bfdb33d611af35361d7f72625b"
}
....(略)
```

從上面可以看到 topology 資訊匯入到 Heketi 所產生的回應訊息 & Cluster ID，其中 **Cluster ID**(上面 Cluster ID = `6190e1bfdb33d611af35361d7f72625b`) 會在下一個步驟中使用到。



Kubernetes Cluster 設定
======================

由於要掛載 GlusterFS volume，因此在**每個 worker node** 上都必須要準備好 GlusterFS client 才行，由於實驗的 k8s 環境是安裝在 Ubuntu 上，因此可以透過以下指令安裝 GlusterFS client：

```bash
$ sudo add-apt-repository ppa:gluster/glusterfs-4.1 -y
$ sudo apt-get -y install glusterfs-client
```

若是 worker node 是 CentOS，可用下面指令安裝 GlusterFS client：

```bash
$ yum -y install centos-release-gluster41
$ yum install -y glusterfs-fuse
```

這個部份若是漏掉了，會出現 **mount: unknown filesystem type 'glusterfs'** 錯誤，在 pod 生成的時候會出現類似下面的訊息：

```txt
the following error information was pulled from the glusterfs log to help diagnose this issue: could not open log file for pod web-0
  Warning  FailedMount  1m  kubelet, leon-k8s-node03  MountVolume.SetUp failed for volume "pvc-6f3abcc5-bfec-11e8-885a-627e9087949f" : mount failed: mount failed: exit status 32
Mounting command: systemd-run
Mounting arguments: --description=Kubernetes transient mount for /var/lib/kubelet/pods/6f3b9642-bfec-11e8-885a-627e9087949f/volumes/kubernetes.io~glusterfs/pvc-6f3abcc5-bfec-11e8-885a-627e9087949f --scope -- mount -t glusterfs -o auto_unmount,log-level=ERROR,log-file=/var/lib/kubelet/plugins/kubernetes.io/glusterfs/pvc-6f3abcc5-bfec-11e8-885a-627e9087949f/web-0-glusterfs.log,backup-volfile-servers=10.107.13.101:10.107.13.102:10.107.13.103 10.107.13.101:vol_e3f5e3f4faf604ab10014c9d1a274624 /var/lib/kubelet/pods/6f3b9642-bfec-11e8-885a-627e9087949f/volumes/kubernetes.io~glusterfs/pvc-6f3abcc5-bfec-11e8-885a-627e9087949f
Output: Running scope as unit run-ra3dac606d12c48fc9cdf4667bf5a9963.scope.
mount: unknown filesystem type 'glusterfs'
```



測試 Kubernetes 與 Heketi 的連結
==============================

在上面有提到 Heketi service 將會裝在第一個 node 上(**gfs01**, IP=**10.107.13.101**)，以 docker container 的方式提供服務；而且為了安裝方便，有些參數已經寫死在參數檔中(例如：admin password)，因此這邊可以彙整出以下資訊：

- GlusterFS cluster ID: `6190e1bfdb33d611af35361d7f72625b`

- Heketi service URL: `http://10.107.13.101:8080`

- admin password: `admin_key`

有了以上的資訊後，就可以往下進行與 k8s 的整合測試。

## 建立 Storage Class

在安裝程式中，預設 admin 的密碼為 `admin_key`，在 k8s 中可以透過 [secret](https://kubernetes.io/docs/concepts/configuration/secret/) 的方式帶入：

- 檔名：`gfs-storageclass.yml`

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
data:
  # echo -n "admin_key" | base64
  key: YWRtaW5fa2V5
type: kubernetes.io/glusterfs

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: "my-gfs-storageclass"
  annotations:
    # 設定為預設的 storage class
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.107.13.101:8080"
  clusterid: "6190e1bfdb33d611af35361d7f72625b"
  restauthenabled: "true"
  restuser: "admin"
  secretNamespace: "default"
  secretName: "heketi-secret"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:2"
```

```bash
$ kubectl apply -f gfs-storageclass.yml
secret/heketi-secret created
storageclass.storage.k8s.io/my-gfs-storageclass created
```

## 建立 StatefulSet

這邊使用 StatefulSet 搭配 VolumeClaimTemplate 對 `my-gfs-storageclass` 進行 dynamic volume provisioning：

- 檔名: `statefulset.yml`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-gfs-storageclass"
      resources:
        requests:
          storage: 1Gi
```

```bash
$ kubectl apply -f nginx.yml 
statefulset.apps/web created
```

完成之後，可以看到 PV & PVC 自動被建立，並 bind 到每個 pod 中：

```bash
# 取得 pv 資訊
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM               STORAGECLASS          REASON    AGE
pvc-0b8f49d6-c156-11e8-885a-627e9087949f   1Gi        RWO            Delete           Bound     default/www-web-1   my-gfs-storageclass             1m
pvc-1d15ce12-c156-11e8-885a-627e9087949f   1Gi        RWO            Delete           Bound     default/www-web-2   my-gfs-storageclass             44s
pvc-fdda23de-c155-11e8-885a-627e9087949f   1Gi        RWO            Delete           Bound     default/www-web-0   my-gfs-storageclass             1m

# 取得 pvc 資訊
$ kubectl get pvc
NAME        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
www-web-0   Bound     pvc-fdda23de-c155-11e8-885a-627e9087949f   1Gi        RWO            my-gfs-storageclass   2m
www-web-1   Bound     pvc-0b8f49d6-c156-11e8-885a-627e9087949f   1Gi        RWO            my-gfs-storageclass   1m
www-web-2   Bound     pvc-1d15ce12-c156-11e8-885a-627e9087949f   1Gi        RWO            my-gfs-storageclass   1m

# 剛剛佈署的 StatefulSet 也都完成佈署了
$ kubectl get all
NAME        READY     STATUS    RESTARTS   AGE
pod/web-0   1/1       Running   0          2m
pod/web-1   1/1       Running   0          2m
pod/web-2   1/1       Running   0          1m

NAME                                  TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/glusterfs-dynamic-www-web-0   ClusterIP   10.233.60.3   <none>        1/TCP     2m
service/glusterfs-dynamic-www-web-1   ClusterIP   10.233.17.8   <none>        1/TCP     1m
service/glusterfs-dynamic-www-web-2   ClusterIP   10.233.16.6   <none>        1/TCP     1m
service/kubernetes                    ClusterIP   10.233.0.1    <none>        443/TCP   21d

NAME                   DESIRED   CURRENT   AGE
statefulset.apps/web   3         3         2m
```

看到以上訊息，就表示 **Kubernetes <--> Heketi <--> GlusterFS** 三個不同服務的串接已經正確的設定完成。



結論
====

若是單獨使用 GlusterFS volume 的情況下，完整的流程應該分成兩大部份：

- 安裝 GlusterFS
  - 建立 peer information
  - check peer status & pool list

- 設定 volume
  - 新增 volume (設定 replica & 指定使用哪個 node 的 brick)
  - 掛載 volume 並使用

而 Heketi 把第二個部份的工作都自動化了，因此在設定 Heketi 時，只要匯入所謂的 topology 資訊(每一次匯入則是新增一個 cluster)，告訴 Heketi 以下資訊：

- 這個 cluster 有哪些 node

- 每個 node 可用來儲存 data 的 device(or partition) name

- failure domain (這個尚不清楚可以達成什麼效果，但應該跟 data redundacy policy 有關係)

接著，產生 brick & volume 這種煩瑣的工作，就由 Heketi 來負責自動完成。

然而 Heketi 除了以上功能外，還可以同時管理多個 GlusterFS cluster，因此可以讓使用者在資料配置上不受限於單一 cluster，在規劃 & 運用上也更有彈性；不但可以減輕 storage manager 的負擔，同時也藉由與 Kubernetes 的搭配，讓實際應用程式的開發者可以更專注在應用上，而非複雜的 infra 設定上。



References
==========

- [Gluster Storage System](http://www.l-penguin.idv.tw/book/Gluster-Storage_GitBook/)

- [heketi/heketi: RESTful based volume management framework for GlusterFS](https://github.com/heketi/heketi)

- [Heketi Documentation - Administration](https://github.com/heketi/heketi/blob/master/docs/admin/readme.md)

- [Prepare Heketi Topology](https://github.com/heketi/heketi/blob/master/docs/admin/topology.md)

- [独立部署GlusterFS+Heketi实现Kubernetes共享存储 - breezey - 博客园](https://www.cnblogs.com/breezey/p/8849466.html)

- [GlusterFS 操作備忘 « Jamyy's Weblog](http://jamyy.us.to/blog/2014/03/6220.html)

