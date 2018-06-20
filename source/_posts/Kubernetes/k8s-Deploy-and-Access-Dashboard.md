---
layout: post
title:  "佈署 & 存取 Kubernetes Dashboard"
description: "本篇文章介紹在 Kubernetes 中佈署 & 存取 Dashboard"
date: 2018-06-20 14:50:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
---


剛安裝完 Kubernetes，第一個會想到的大概就是 "**入口網站在哪裡**"?

k8s 無疑是個非常強大的 container orchestration platform，但它預設並沒有附上一個精美的 UI(因為不是每個人都需要 UI，畢竟多少還是會消耗掉一些資源)，設計者提供給使用者 & 開發者完全自由的使用方式。

**有需要就自己設計 & 安裝吧!**



安裝 Dashboard
=============

自己設計 dashboard 當然是無法，但找個現成的就挺簡單了。

若是使用 [Kubespray][Kubespray] 安裝的 k8s cluster，dashboard 是預設會安裝好的；若要確認 dashboard 是否會被安裝，可以在 `group_vars/k8s-cluster.yml` 這個檔案中，加入以下設定：

> dashboard_enabled: true

當 k8s cluster 安裝好後，dashboard 也會被一併安裝完成。

若是沒有呢? 那其實也很簡單，進入 master node 中，透過 [kubectl][kubectl] 執行以下指令安裝：

```bash
#
# 安裝最新版的 k8s dasdboard (namespace = kube-system)
#
root@kube-master0:~# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml


#
# 檢查 dashboard 是否安裝完成
#
root@kube-master0:~# kubectl get deployment -n kube-system | grep dashboard
kubernetes-dashboard   1         1         1            1           5d
```

當系統出現 `kubernetes-dashboard` 這個 deployment resource 時，就表示安裝完成了。



存取 Dashboard
=============

由於 [kubectl][kubectl] 並不在本地端，因此使用 `kubectl proxy` 也沒太大意義，因此這邊直接就直接使用 master node 上的入口進入，使用以下的連結:

> https://first_master:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

進入後會出現以下畫面：

![Kubernetes Dashboard Login Page](/blog/images//kubernetes/dashboard-login-page.png)

登入方式有兩種，分別是：

- Kubeconfig

- Token

以下介紹如何使用 Token 來登入 Dashboard。



建立可登入 Dashboard 的使用者
=========================

剛安裝好的 k8s cluster，上面已經存在有很多 Service Account，但這些都是 k8s cluster 中的各種服務，**並非真實的使用者**。

但由於目前 k8s cluster 中並沒有與任何的帳號系統(OpenLDAP / Windows AD / OpenID .... etc) 進行連結，因此以下還是會以建立 Service Account 的方式來模擬建立一個新的使用者進行示範:

```bash
#
# 建立一個名稱為 "admin-user" 的 Service Account，用來模擬管理者
#
root@kube-master0:~# cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
EOF

#
# 為了取得最大權限，預計會將 "admin-user" 與 role "cluster-admin" 進行繫結
#
root@kube-master0:~# kubectl describe clusterrole/cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate=true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]


#
# 將 "admin-user" 與 role "cluster-admin" 進行繫結，建立名稱為 "admin-user" 的 ClusterRoleBinding
#
root@kube-master0:~# cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF

#
# 檢視 ClusterRoleBinding "admin-user" 的內容
# 
root@kube-master0:~# kubectl describe ClusterRoleBinding/admin-user
Name:         admin-user
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind            Name        Namespace
  ----            ----        ---------
  ServiceAccount  admin-user  kube-system
```

當 Service Account 被建立完成時，Kubernetes Token Controller 就會自動的為其產生一個 **secret** resource(名稱為 `[SERVICE_ACCOUNT_NAME]-token-[RANDOM_STRING]`)，並使用裡面的 token 作為與 API server 認證用，可以用以下指令察看：

```bash
#
# 檢視建立 service account 時產生的 secret
#
root@kube-master0:~# kubectl get secret -n kube-system | grep admin-user
admin-user-token-572j8                           kubernetes.io/service-account-token   3         11m

#
# 檢視 secret 細節，並取得 token 內容
#
root@kube-master0:~# kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
Name:         admin-user-token-572j8
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=ad79cce8-7453-11e8-b6b3-065296fdbf18

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1090 bytes
namespace:  11 bytes
token:      [TOKEN_CONTENT]
```



登入 Dashboard
=============

透過上面的 secret 取得 token 後，就可以選擇 login page 上的 `Token` 選項登入，進入後可以看到以下畫面：

![Kubernetes Dashboard Overview Page](/blog/images//kubernetes/dashboard-overview-page.png)

接著就可以以 Cluster Admin 的身份進行操作了。



後記
===

由於 k8s 內部是不儲存使用者帳號的，它設計可以外接不同的帳號認證系統。因此若是要真的執行嚴格的權限管理，並不是像上面的建立一個擁有 cluster admin 權限的 Service Account(因為 Service Account 顧名思義是給 service 用的)，而是要與外部的認證系統串接(例如：OpenID, LDAP ... etc)。

因此之後將會找時間補上與外部認證系統串接這個部份，將使用者帳號獨立在 k8s cluster 外部。



References
==========

- [kubernetes/dashboard: General-purpose web UI for Kubernetes clusters](https://github.com/kubernetes/dashboard)

- [Creating sample user · kubernetes/dashboard Wiki](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user)