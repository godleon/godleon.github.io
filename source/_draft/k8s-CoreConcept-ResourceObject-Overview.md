---
layout: post
title:  "[Kubernetes] Resource Object 概觀"
description: "此篇文章從比較 high level 的角度來了解什麼是 Kubernetes Resource Object，以及 Kubernetes 中到底包含了那些 Resource Object & 其用途"
date: 2018-08-31 22:55:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - CKA
---

此篇文章從比較 high level 的角度來了解什麼是 Kubernetes Resource Object，以及 Kubernetes 中到底包含了那些 Resource Object & 其用途


What is Resource Object?
========================

在 k8s 中，定義了各式各樣的 resource object(or `API object`)，每個 resource object 都是一個 persistent entity，用來表示目前 cluster 的特定狀態 & 運作行為，例如：

- 有哪些 containerized application 正在運行?

- 承上，在哪些 worker nodes 上運行?

- 這些 application 可用的 resource 為何?

- application 運作時的規範，例如：restart policy, fault-tolerance

此外，每個 resource object 都包含了 spec & status 兩個部份，其中 spec 是用來描述 object 要以什麼樣的形式(規格 or 內容)呈現，而 status 則是用來描述這個 object 希望由 K8s 所維持的狀態(desired state)。

> 關於 spec & status 的細節，可以參考官網的 [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/api-conventions.md)



Kubernetes 提供了那些 Resource Object Type?
=========================================

k8s 提供的 resource object 種類相當多，以下根據運用的類型作成以下四大分類：

| Category | Resource Object Type Name |
|----------|---------------------------|
| Workload | Pod, HorizontalPodAutoscaler |
| Controller | ReplicaSet, ReplicationController, Deployment, StatefulSet, DaemonSet, Job, CronJob |
| Service Discovery | Service, Ingress |
| Authentication & Authorization | ServiceAccount, RBAC(Role, ClusterRole, RoleBinding, ClusterRoleBinding) |
| Storage | Volume, PersistentVolume, StorageClass, Secret, ConfigMap  |
| Policy | NetworkPolicy, SecurityContext, ResourceQuota, LimitRange |
| Extension | CustomResourceDefinitions |


| Resource | `Pod`, `ReplicaSet`, `ReplicationController`, `Deployment`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob`, `HorizontalPodAutoscaling` |
| Configuration | Node, Namespace, `Service`, `Secret`, `ConfigMap`, `Ingress`, Label, ThirdPartyResource,  `ServiceAccount` |
| Storage | Volume, PersistentVolume, StorageClass |
| Strategy | SecurityContext, ResourceQuota, LimitRange |

從上面的分類可以看出，作為一個 multiple node 的 container orchestration platform，k8s 為了將 container 進行有效的管理，將很多管理概念抽象化，儘量隱藏底層複雜的實作細節，讓使用者可以根據自己的需求，專注在特定的 resource object 上，進而有效率的完成工作。



Workload 類型的 Resource Object
==============================

以下針對各種不同的 Resource Object Type 作個簡單介紹：

## Pod

Pod 是 k8s 中的最小運作單位，要了解 Pod 是什麼，則必須看看官網的定義：

> A pod (as in a pod of whales or pea pod) is a group of one or more containers (such as Docker containers), with shared storage/network, and a specification for how to run the containers.

從上述的定義可以知道幾件事情：

1. 一個 pod 中可以有一個或多個 container

2. 在同一個 pod 的 container 共享相同的 storage & network(因此在同一個 Pod 裡面的 containers，可以用 local port numbers 來互相溝通)

3. pod 的 spec 宣告中還必須包含 container 要如何運作



Controller 類型的 Resource Object
================================

## ReplicationController

用途是在確保同一時間內，有指定數量的 pod 在運行，可隨時進行 scale out/in。

在 k8s v1.2 後，就開始由 **Deployment** + **ReplicaSet** 取代，為了就是要準備支援在 v1.3 後出現的 Auto Scaling 的功能。


## ReplicaSet

可以將 ReplicaSet 視為進化版的 ReplicationController，其中兩者最大的差別在於 ReplicaSet 可以被更多種不同的 label selector 支援。

> ReplicationController 僅能支援 **=** 以及 **!=** 兩種，而 ReplicaSet 還支援了 chain & subset 等更複雜的 selector 功能

雖然 ReplicaSet 提供了更強大的 selector 功能，但不建議單獨使用，而是透過與 Deployment 來搭配使用。


## Deployment

Deployment 其實就是 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) + [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)，同樣的，使用者對 k8s 宣告 Deployment 的 desired status，接著 Deployment controller 就會在資源允許的範圍內幫你達成。

> 請不要直接控制包在 Deployment 內部的 ReplicaSet，應該是要透過調整 Deployment 來改變實際 workload 運作的行為

然而 Deployment 可以幫我們達成以下幾件事情：

- 部署一個應用服務(application)，並確保維持在 desired status

- 將 applications 升級到某個特定版本，同時也可以降級到某個特定版本

- zero downtime upgrade

- 當問題發生時，可以快速的 rollback

當然 Deployment 的功能不只上面這些，之後有機會會用獨立篇幅來介紹。


## StatefulSets - Kubernetes

StatefulSets 是在 v1.9 後正式釋出的功能，顧名思義，主要功能就是作為 stateful application 的管理，例如：Database。

與 Deployment Pod 相比較之下，有些相同也有些不同的地方，首先說明相同的地方：

- 與 Deployment 相同，都是透過 container spec 來描述 desired status

而與 Deployment Pod 不同的地方則是：

- StatefulSets 會為每一個 pod 維護一個固定的識別資訊，而 Deployment 則不會

- 承上，StatefulSets pod 雖然都是從同樣的 spec 產生出來，但卻不是等價關係，因此無法互相替代對方的任務

- 每個 StatefulSets Pod 都有一個永久性的識別資訊，不會因為被 reschedule 而改變

StatefulSet 的管理是由 k8s 中名稱為 StatefulSet Controller 的元件來負責。


## DaemonSet

由 daemon controller 管理，確認每個 node 上都會有運行一個指定的 pod，而當有新的 node 加入到 cluster 中時，k8s 也會自動幫新的 node 加上一個 daemonset pod；若是 node 被移除時，當然這些 pod 也都會被刪除掉。

典型的應用如下：

- 運行 storage daemon，例如: glusterd, ceph ... 等等

- 運行蒐集 log 用的 daemon，例如: fluentd, logstash ... 等等

- 運行監控用的 daemon，例如: [Prometheus Node Exporter](https://github.com/prometheus/node_exporter), collectd ... 等等


## Job 

Job 很直覺，跟 Linux 中的 `at` 是類似的概念。

而在 k8s 中，job 會建立一個 or 多個 pod，並確保有特定數量的 pod 執行成功，條件達成後，job 就會進入完成狀態，此時 job 以及相關的 pod 都會被刪除。


## CronJob

跟 Linux 相同，CronJob 也是在特定的時間 or 週期性的特定時間運行 job；在使用上，時間格式的定義也是與 Linux 相同。

典型的運用像是資料庫的備份、發送郵件 ... 等等



Service Discovery 類型的 Resource Object
=======================================

## Service

Service 是為了讓**應該要送到 pod 處理的網路流量可以正確的送達**而抽象出來的一個概念，為了達到此目的，需要具備幾個條件：

DNS 服務 (沒人會想用 IP 存取服務)

網路服務 (kube-proxy 在 pod 上層提供了一個 Load Balance 的服務)

以下是一個典型的 service 應用：

![Kubernetes Service Example](/blog/images/kubernetes/k8s-service.svg)

- 每個 service 會帶有一個 VIP

- 在 cluster 內部可以透過 **\<service_name>.\<namespace>.cluster.local** 這個 domain name 存取到

- 若要在 cluster 外部存取 service，則必須將 service 的 type 設定為 **NodePort** or **LoadBalancer**(只有在公有雲有效)

- service 的流量要導到那些 pod，是用 Label 來定義

- kube-proxy 會主動產生相對應的 iptables rule，讓 traffic 可以平均的分流到指定的 pod


## Ingress

k8s 在沒有 Ingress 之前，要從外部存取 k8s 的 service，僅能透過 **NodePort** 的方式來達成(在公有雲可以使用 **LoadBalancer**)，架構大概如下：

```
    internet
        |
  ------------
  [ Services ]
```

Ingress 在 v1.1 後出現，目的就是要解決外部存取 k8s service 的問題，因此架構變成如下圖：

```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

Ingress 負責的事情主要被定義為下面幾項：

- give services externally-reachable urls

- load balance traffic

- SSL Termination

- offer name based virtual hosting



權限管理類型的 Resource Object
===========================

## Service Account

在 k8s 中一個 pod 可以正常開始運作與否，取決於這個 pod 有沒有足夠的權限，在 k8s 中每個 pod 運作時的權限，則是以 service account 這個 resource object 來表示。

但在 k8s 中，真正決定 pod 可不可以被 schedule 到 worker node 的是 API server，而要跟 API server，必須要有合法的 token，從哪裡來呢?

首先要知道幾件事情：

- 每個 pod 都會屬於特定的 namespace

- 每個 namespace 在建立之初，會有一個名稱為 **default** 的 service account 會同時被建立

```bash
$ kubectl get serviceaccount
NAME      SECRETS   AGE
default   1         18h
```

- 每個 service account 會帶有一個 `secret` resource object

```bash
$ kubectl describe serviceaccount/default
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-zmx94
Tokens:              default-token-zmx94
Events:              <none>
```

- 上述的 secret 帶有與 API server 認証用的 token 資訊

```bash
$ kubectl describe secret/default-token-zmx94
Name:         default-token-zmx94
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=default
              kubernetes.io/service-account.uid=b9881486-abf6-11e8-9824-66712a0dd587

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1090 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tem14OTQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImI5ODgxNDg2LWFiZjYtMTFlOC05ODI0LTY2NzEyYTBkZDU4NyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.l8xziA1DiH_u2UUSPmbo3Gry5DqjRn9qHG6SAiu83BkHmWNXaY_K1-55Ta_Uq3m1I2LCkMxCmjL_Aoau_efHLMqbT7nCJHuG78r9mVPOuy2OyhFLPniKE3-QteY_53smOFV5p-DathRP1zzbdzsiasigjKOlLn_2TDFMcy9ZvAyOzm20MqiQE31CVrdiMWPOD3pl0AerWnRkEsoxdIJ70RK8pc0Z65FQ3uILCrGSMlwr5FPl_cxw6nj2yTm7UeONPvHskIG2uzedUi9dX7LsnXj__SorWizuFc1G8aN3RyzXUgstXo0h6x_tOwINAk7f9LHPBhQJ7zBpWGNC3hxR8w
```

因此藉由上面的說明，可以了解 namespace, pod, service account, secret, token, 與 API server 之間的關係


## RBAC 機制

RBAC 是目前 k8s 預設會啟動的 authorization module。

在 RBAC API 中定義了 resource target，用來描述使用者以及 resource 之間的權限關係：

- **Role**：定義在特定 namespace 下的 resource 的存取權限

- **RoleBinding**： 設定哪些使用者(or service account)與 role 綁定而擁有存取權限

- **ClusterRole**：定義在整個 k8s cluster 下的 resource 的存取權限

- **ClusterRoleBinding**：設定哪些使用者(or service account)與 role 綁定而擁有存取權限

了解 RBAC 的運作機制，在使用一些 k8s extension(例如：Helm) 的時候會有幫助，可以了解這些 extension 是如何在 k8s 中取得合法的運作權限。



Storage 管理類型的 Resource Object
================================

## Volume

由於 container 的生命周期是短暫的，並且可能會隨時被 reschedule 到其他的 node 上面運行，這看似資源有效的被管理的前提下，同時也衍生出了 application 資料儲存的問題，而 **volume** 就是在 k8s 中被提出來的一個抽象概念，要來解決資料儲存的問題，並隱藏底層複雜的 storage 相關細節。

類似於 Docker 中的 volume 概念，k8s 支援的 volume 有相當多種，例如：hostPath, iSCSI, NFS, GlusterFS, Ceph ... 等等，詳細清單可以參考[官網的列表](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes)。


## PersistentVolume & PersistentVolumeClaim

有了 volume 被設計出來的目的後，接著進入到在 k8s 實作的細節；由於 storage 的管理一直是一個獨立且複雜的課題這個部份，因此k8s 提出了 **PersistentVolume(PV)** & **PersistentVolumeClaim(PVC)** 兩個抽象概念來實現永久性資料的儲存，藉由提供標準的 API，讓 storage 的管理與應用抽象化。

其中，PersistentVolume(簡稱 `PV`) resource 在概念上可以視為可讓 pod 掛載的 storage，後面的實作(NFS, iCSCI, RBD ... etc)都已經被標準的 API 隱藏，管理者可以透過標準的宣告方式佈署所需要的 storage resource；而 PersistentVolume 的 lifecycle 會跟著使用 PersistentVolume 的 pod 走，當所有 pod 都消失，PersistentVolume resource 也會跟著消失，但儲存在上面的資料依然會存在。

PersistentVolumeClaim(簡稱 `PVC`) resource 則是來自於使用者的 request，概念上類似 pod，只是 pod 使用 worker node 上的運算資源，可限定 CPU & Memory；而 PersistentVolumeClaim 使用的則是 PV，可限定 storage 的 size & access mode(read/write or read-only)。

然而其實 pod, PV, PVC 之間的關係是這樣的：

```
Pod <---> PersistentVolumeClaim(PVC) <---> PersistentVolume(PV)
```

1. Pod 中定義要使用哪個 PVC，其中包含了一些 storage requirement

2. PVC 在 k8s cluster 中尋找可用的 PV 並掛載


## StorageClass

PV + PVC 解決了永久性資料儲存的問題，但同時也帶來了以下問題：

- 如果我常常需要產生 PV，又要刪除它呢 ?

- 可以不要一直去煩 storage manager 嗎? 每次產生新的 PV 都需要請他建立對應的 RBD image

- 產生 PV 的動作可以自動完成嗎 ?

- 每次都要手動建立 RBD image 哪有 cloud native ?

而 Storage Class 就是以上問題的解答，其流程如下圖：

![Kubernetes Persistent Volume Provisioning](/blog/images/kubernetes/dynamic-volume-provision.jpg)

1. 設定 persistent volume provisioner

2. k8s ckuster 管理者建立 Storage Class，並指定要使用的 PV provisioner

3. 使用者建立 PVC，指定要使用的 Storage Class

4. Storage Class 使用 provisioner 在實際的 storage 上產生 volume，並建立 PV 與其繫結

5. 將 PV 與 使用者 PVC 進行繫結

6. 使用者建立 pod，並使用 PVC 取得外部的儲存空間

> **以上的重點在於，管理者再也不用辛苦的手動建立儲存空間，並設定 PV 與其繫結了，這個部份都交給 StorageClass 自動處理**


## Secret

Secret 是用來解決在 k8s 中密碼、token，key 或是任何敏感資訊的管理問題，使用者不用把這些敏感資訊定義在 image or pod 的 spec 設定中，而是透過 volume 掛載的方式或是環境變數的方式在 pod 中使用。

上面提到在 service account 中，就是利用了 secret 來儲存與 API server 溝通用的 token；而一般使用者也可以定義 **Opaque** 類型(以 base64 的方式編碼)的 secret 來儲存需要隱藏的敏感性資料。


## ConfigMap

所有的 application 在開發上，總是會遇到設定檔(configuration)處理的問題，最典型的就是在不同的環境中(testing, staging, production)需要連結到不同的資料庫，當然也就會有不同的資料庫連線資訊。

若是將 configuration 打包到 container image 中，那有多少個環境就要打包多少個 image，其實這是相當沒有效益的，因此在 k8s 中，提出了 **ConfigMap** 的概念來解決這個問題。

ConfigMap 可以包含的資訊其實沒什麼太多限制，可以保存單一屬性，也可以是完整的 configuration 內容，或是一個 json 格式的資料也都可以，儲存的方式是以 key/value 的型式；使用上類似 secret，若是處理非機密資料的話是相當合適的。



Policy 類型的 Resource Object
============================

## NetworkPolicy

Network Policy 定義了 pod 之間是否可以互相通訊，以及到各個不同的 endpoint 之間是否可以互相通訊。而 policy 的套用同樣也是透過 label selector 來篩選。


## SecurityContext

當我們要在 k8s 使用可能不受信任的 container image 時，可能會希望限制該 pod or container 可以運作的權限 or 範圍，而在 k8s 中，Security Context 就是用來定義這樣的限制，來確保他容器不受其影響 & 保護系統。

在 k8s 中提供了三種配置 Security Context 的方法：

- Container-level Security Context：僅套用到指定的容器

- Pod-level Security Context：套用到 Pod 內所有容器以及 Volume

- Pod Security Policies（PSP）：套用到 k8s cluster 內部所有 Pod 以及 Volume


## ResourceQuota

當一個 k8s cluster 中同時有多個使用者 or 團隊同時使用 & 硬體資源有限時，ResourceQuota 就是管理者可以用來控制資源利用量的工具。

ResourceQuota 的套用是以 namespace 為單位，預設是沒有任何 quota 的，而每個 namespace 也只能有一個 ResourceQuota 物件來管理資源的使用量，目前可管理的資源類型有以下三種：

- **Compute Resource**: 這個包含 CPU & Memory

- **Storage**: 可用來限制每個 namespace 可用的儲存空間

- **Object Count**: 在 v1.9 正式釋出的功能，主要用來限制特定 resource object type 的數量，例如：`count/persistentvolumeclaims`, `count/deployments.apps`


## LimitRange

LimitRange 跟 ResourceQuota 同樣也是用來限制資源使用量，但 LimitRange 主要是用來設定資源申請的上下限範圍限制的預設值。

兩種資源控制的機制都是以 namespace 為單位，其中 `ResourceQuota` 是用來設定在 namespace 所有 pod 佔用資源的 request & limit，而 `LimitRange` 則是用來設定 namespace 中 pod 的預設使用資源的 request & limit。



Extension 類型的 Resource Object
===============================

## CustomResourceDefinitions

k8s 原生的 resource type 有 pod, deployment, daemonset, service ... 等等，但若這些 resource type 並不完全符合使用者需求時，使用者可以自行定義所謂的 Custom Resource & 相對應的 controller 來進行進階的 container 應用，而在操作使用上同樣也可以透過 kubectl 來進行管理。



References
==========

- [Pod Overview - Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)

- [[Day 5] 在 Minikube 上跑起你的 Docker Containers - Pod & kubectl 常用指令 - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](https://ithelp.ithome.com.tw/articles/10193232)

- [[Day 8] 還在用Replication Controller嗎？不妨考慮Deployment - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](https://ithelp.ithome.com.tw/articles/10194152)

- [ReplicaSet - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

- [Deployments - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

- [StatefulSets - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

- [Jobs - Run to Completion - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)

- [CronJob - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

- [Services - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/service/)

- [Ingress - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress/)

- [Configure Service Accounts for Pods - Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)

- [Managing Service Accounts - Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)

- [Using RBAC Authorization - Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

- [Volumes - Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/)

- [Persistent Volumes - Kubernetes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

- [Storage Classes - Kubernetes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

- [Secrets - Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/)

- [Configure a Pod to Use a ConfigMap - Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)

- [Network Policies - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

- [Configure a Security Context for a Pod or Container - Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

- [Security Context 和 Pod Security Policy· feiskyer/kubernetes-handbook](https://github.com/feiskyer/kubernetes-handbook/blob/master/zh/concepts/security-context.md)

- [Resource Quotas - Kubernetes](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

- [Kubernetes中的ResourceQuota和LimitRange配置资源限额 - 宋净超的博客|Cloud Native Developer Advocate|jimmysong.io](https://jimmysong.io/posts/kubernetes-resourcequota-limitrange-management/)

- [Custom Resources - Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)