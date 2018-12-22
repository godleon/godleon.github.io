---
layout: post
title:  "[Kubernetes] Package Manager - Helm 簡介"
description: "本篇文章的主題在介紹在 Kubernetes Package Manager(Helm) 如何協助 Kubernetes 使用者更方便 & 更有效率的將複雜的 service 佈署到 Kubernetes cluster 上"
date: 2018-12-22 16:40:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Helm
---


什麼是 Kubernetes Helm?
======================

首先來看一下[官網][Helm]的解釋：

> Helm is a tool for managing Kubernetes charts. Charts are packages of pre-configured Kubernetes resources.

Helm 就有點像是 Ubuntu 中的 APT 系統，也類似 Ansible 中的 Galaxy；簡單來說就是可以將一個 or 多個應用包裝成一個服務，並透過 chart 的形式發佈，讓大家可以方便在 k8s 上安裝特定的服務。

詳細的說明 & 架構可以參考此篇文章 ----> [The Kubernetes Package Manager : Helm - inwinSTACK](http://www.inwinstack.com/zh/2017/06/15/the-kubernetes-package-manager-helm/)

以下也有兩張圖可以說明一下 Helm 系統架構：

![Kubernetes Persistent Volume Provisioning](/blog/images/kubernetes/helm-infra-1.jpg)

![Kubernetes Persistent Volume Provisioning](/blog/images/kubernetes/helm-infra-2.png)

從上面的文章中，要弄清楚 `Helm client` & `Tiller server` 的關係：

- Tiller server 是用來與 API server 溝通，使用 chart 在 k8s cluster 上建立服務

- Helm client 則是用來操作 Tiller server 用

此外，佈署 Helm 時可以將其佈署在 cluster level(整個 k8s cluster 皆可用)，也可以佈署在 namespace level(只會在該 namespace 可用)，這部份可以根據自己的需求去決定。


Helm 要解決什麼問題?
=================

一個工具被開發出來，總是被賦予要解決問題的任務，而 Helm 到底可以協助我們解決什麼樣的問題呢?

先來討論目前的趨勢：

- 線上的服務開始往 micro service 的方向走

- k8s 似乎也變成整個 container 大一統的平台

因此使用 k8s 作為承載 micro service 的平台的確也是很正常的事情，但 k8s 為了解決各式各樣的問題，提供了大量的 resource object(pod, deployment, service ... etc)，因此如何利用這些 resource object 來組成一個完整的系統，整個過程是很複雜的，其中包含了很多不同面向的問題需要處理，例如：

- 如何管理、編輯 & 更新大量的 k8s resource object 設定檔?

- 如何快速佈署一個含有很多設定檔的 k8s application?

- 如何分享 & 重複利用 k8s resource object 設定檔?

- 如何透過 parameter & template 的方式，使用相同的設定檔來支援不同環境上的佈署?

- 如何管理 application 的 deployment, rollback, diff & history?

- Application 發佈後如何進行驗證?

在 micro service 的過程中，林林總總的問題也跑了出來，而 Helm 就是為了解決上述問題而被開發出來的工具，它使用了以下幾個方式來解決上述的問題：

1. 將 k8s resource object 打包到一個 chart 中 (chart 是多個 k8s resource object configuration 的集合)

2. 將 chart 保存在 chart repository 中，透過 repository 的方式來儲存 & 分享 chart

3. Helm 則利用 chart 來進行 deployment, upgrade, rollback, version control .... 等工作

透過以上方式，大幅簡化在 k8s 上佈署 application 的複雜度。



安裝 Helm client
===============

在 k8s cluster 安裝 tiller server 之前，我們要先來準備 Helm client，使用以下的指令安裝最新版的 Helm：

```bash
$ wget https://storage.googleapis.com/kubernetes-helm/helm-$(curl -s https://api.github.com/repos/helm/helm/releases/latest | jq --raw-output '.tag_name')-linux-amd64.tar.gz -O helm-linux-amd64.tar.gz

$ tar -zxvf helm-linux-amd64.tar.gz

$ sudo mv linux-amd64/helm /usr/local/bin/helm
```


將 Helm 佈署至 Cluster Level
===========================

## 設定 current context

這裡要先確認 Helm 要安裝在哪個 k8s cluster 中(如果有多個的話)

```bash
# 設定 default context
$ kubectl config use-context admin-cluster.local
Switched to context "admin-cluster.local".
```


## 為 Tiller server 開啟權限

從上面的說明可看出，其實 tiller 就是一個用來與 k8s API server 溝通的 service，因此我們要賦予 tiller 所需要的權限。

權限設定的流程如下：

1. 在 `kube-system` namespace 中建立 Service Account `tiller`

2. 建立 ClusterRoleBinding，將 Service Account `tiller` 與 Cluster Role `cluster-admin` 作繫結

首先準備以下檔案來進行設定，名稱為 `rbac-config.yaml`：

```yaml
# 建立 Service Account "tiller" 給 Helm service 與 API server 認證之用
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system

# 建立 ClusterRoleBinding，將 Role "cluser-admin" 權限賦予 Service Account "tiller" 
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
```

接著將上面的設定套用到 k8s 中:

> kubectl apply -f rbac-config.yaml




最後進行 Helm 的初始化：

> helm init --service-account tiller

如此一來 Helm client 就會開始透過 kubectl 與 k8s cluster 溝通並建立所需要的相關資源：

```bash
$ kubectl get deployment --all-namespaces | grep tiller
kube-system   tiller-deploy             1         1         1            0           11s
```


將 Helm 佈署至 Namespace Level
=============================

這個部份的步驟其實跟 cluster level 差不多，大概就是：

1. 在特定的 Namespace(假設是 `helm-lab`) 中建立一個 Service Account(假設為 `tiller`)

2. 建立一個只有 `helm-lab` namespace 中擁有所有權限的 Role(假設為 `tiller-manager`)

3. 建立一個 RoleBinding(假設為 `tiller-binding`)，將 `tiller-manager` 與 `tiller` 進行繫結

如此一來相關的權限設定大致上就完成了。

最後執行以下命令完成初始化：

> helm init --service-account tiller

詳細的設定可以參考[官網的說明](https://docs.helm.sh/using_helm/#role-based-access-control)。



開始使用 Kubernetes Helm
=======================

## 新增 namespace

這邊先建立一個全新的 namespace(`helm-lab`) 來進行 Helm 的測試，並設定 default namespace：

```bash
# 建立 namespace
$ kubectl create namespace helm-lab
namespace "helm-lab" created

# 設定 context 所使用的 namespace
$ kubectl config set-context "admin-cluster.local" --namespace=helm-lab
Context "admin-cluster.local" modified.
```

## 更新 Helm Repo

接著更新 Helm repository，Helm client 會取得所有 stable 的 charts：

```bash
# 取得最新的 chart 資訊
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 

# 搜尋 helm charts (會出現很多 charts info)
$ helm search -l
.... (略)
```

## 安裝第一個 Helm Chart

```bash
# 搜尋 mysql 相關的 Helm chart
$ helm search mysql
NAME                            	CHART VERSION	APP VERSION	DESCRIPTION                                                 
stable/mysql                    	0.11.0       	5.7.14     	Fast, reliable, scalable, and easy to use open-source rel...
.... (略)

# 安裝 mysql helm chart
$ helm install stable/mysql
NAME:   wayfaring-butterfly
LAST DEPLOYED: Sat Dec 22 07:34:54 2018
NAMESPACE: helm-lab
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME                       TYPE    DATA  AGE
wayfaring-butterfly-mysql  Opaque  2     0s

==> v1/ConfigMap
NAME                            DATA  AGE
wayfaring-butterfly-mysql-test  1     0s

==> v1/PersistentVolumeClaim
NAME                       STATUS   VOLUME  CAPACITY  ACCESS MODES  STORAGECLASS  AGE
wayfaring-butterfly-mysql  Pending  0s

==> v1/Service
NAME                       TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)   AGE
wayfaring-butterfly-mysql  ClusterIP  10.233.22.63  <none>       3306/TCP  0s

==> v1beta1/Deployment
NAME                       DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
wayfaring-butterfly-mysql  1        1        1           0          0s

==> v1/Pod(related)
NAME                                       READY  STATUS   RESTARTS  AGE
wayfaring-butterfly-mysql-6bbdd45dc-jdsn6  0/1    Pending  0         0s
.... (略)

# 列出已經安裝的 Helm chart
$ helm ls
NAME               	REVISION	UPDATED                 	STATUS  	CHART       	APP VERSION	NAMESPACE
wayfaring-butterfly	1       	Sat Dec 22 07:34:54 2018	DEPLOYED	mysql-0.11.0	5.7.14     	helm-lab

# 檢視目前的 resource object
$ kubectl get all
NAME                                            READY   STATUS    RESTARTS   AGE
pod/wayfaring-butterfly-mysql-6bbdd45dc-jdsn6   0/1     Pending   0          5m51s

NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/wayfaring-butterfly-mysql   ClusterIP   10.233.22.63   <none>        3306/TCP   5m51s

NAME                                        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/wayfaring-butterfly-mysql   1         1         1            0           5m51s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/wayfaring-butterfly-mysql-6bbdd45dc   1         1         0       5m51s
```

helm chart 的每一個佈署，都被稱為一個 `release`，因此可以用同樣的 helm chart 安裝多個 release，從上面可以看到，在沒有指定 release name 的情況下，helm client 會自動幫你產生一個，上面的範例則是 `wayfaring-butterfly`

若要移除 release，可以使用 helm delete 命令：

```bash
$ helm delete wayfaring-butterfly --purge
release "wayfaring-butterfly" deleted
```

雖然 release 被刪除，但是還是可以回溯的，因為 Helm 會幫你保留歷史紀錄：

```bash
# 已經被刪除的 release 還是可以查到歷史紀錄
$ helm status wayfaring-butterfly
LAST DEPLOYED: Sat Dec 22 07:34:54 2018
NAMESPACE: helm-lab
STATUS: DELETED
..... (略)
```

若想要回溯已經刪除的 release，可以透過 `helm rollback` 來完成。



結論
===

Helm 提供了 k8s 使用者一個方便佈署服務的方式，雖然 micro service 將 service 管理變得複雜了，但透過 Helm 的協助，可以讓管理者有條理地管理各個不同的服務 & 版本，再也不用跟一堆 YAML configuration 奮鬥，大幅減少人為的錯誤，並提升工作效率。



References
==========

- [五分鐘 Kubernetes 有感 – Michael Hsu – Medium](https://medium.com/@evenchange4/%E4%BA%94%E5%88%86%E9%90%98-kubernetes-%E6%9C%89%E6%84%9F-e51f093cb10b)

- [Helm Docs | Quickstart](https://docs.helm.sh/using_helm/#quickstart)

- [Helm User Guide](https://whmzsu.github.io/helm-doc-zh-cn/)

- [簡化Kubernetes應用部署工具-Helm - 掃文資訊](https://hk.saowen.com/a/2be587a04c4ec2ae2d5542397ad68a94bd102e3aa94e2e58f313f70a3d87f8c9)



[Helm]: https://github.com/kubernetes/helm "Kubernetes Helm"

[charts]: https://github.com/kubernetes/charts "Helm Charts"
