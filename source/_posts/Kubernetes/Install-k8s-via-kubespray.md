---
layout: post
title:  "使用 Kubespray 安裝 Kubernetes v1.10 以上版本"
description: "此篇文章介紹如何使用 Kubespray 安裝 Kubernetes v1.10 以上版本"
date: 2018-06-14 15:50:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubespray
---

此篇文章介紹如何使用 [Kubespray](https://github.com/kubernetes-incubator/kubespray) 安裝 Kubernetes

Current Status
==============

目前支援到 `v1.11.3`，如果有需要調整，可以按照下面的說明進行修改。


準備安裝環境
==========

## 安裝 k8s cluster 用機器

在這個安裝過程中，共有 6 個 VM，皆為透過 `Ubuntu 16.04 cloud image` 所產生，分別有以下幾個 node：

- Master Node x 3 (**同時兼任 etcd node**)

- Worker Node x 3

> 需要確保以上的 VM 可以透過 ssh key 或是帳號密碼登入


## 準備用來執行安裝指令的機器 (bootstrapper)

這台機器是用來作為執行安裝程序之用，需求如下：

- OS: **Ubuntu 16.04(Trusty)** or **Ubuntu 18.04(Bionic)**



取得安裝程式碼
===========

首先登入到 bootstrapper，輸入以下指令：

```bash
$ sudo apt-get update && sudo apt-get -y install git
$ cd /tmp
$ git clone https://github.com/godleon/kubernetes-installation-template.git
```

上面的程式其實只是個 wrapper，目的是協助使用者快速產生安裝 Kubespray 用的環境。



修改安裝環境設定
=============

接著進入 **/tmp/kubernetes-installation-template** 目錄中

> cd /tmp/kubernetes-installation-template

接著根據自己環境的情況，進行後續的修改 & 調整。


## 準備存取機器的 SSH Key

將存取機器的 SSH Key(檔名為 **id_rsa**) 放到 **$(pwd)/ssh-privkey** 目錄中

> 若用帳號 + 密碼存取可忽略此步驟


## $(pwd)/kubespray/hosts.ini

這個檔案定義了要用來安裝 k8s cluster 的機器有哪些，修改時有以下幾點需要注意：

- 務必輸入機器正確的 hostname

- 下方每個機器的 role(`kube-master`/`etcd`/`kube-node`) 要定義清楚

- 一般來說，master 跟 etcd 會安裝在一起 (大規模佈署才需要分開)

- Ingress 可以暫時先忽略

以下是根據上面的安裝環境所撰寫的設定範例：

```ini
kube-master0    ansible_ssh_host=[YOUR_SERVER_IP]
kube-master1    ansible_ssh_host=[YOUR_SERVER_IP]
kube-master2    ansible_ssh_host=[YOUR_SERVER_IP]

kube-worker0    ansible_ssh_host=[YOUR_SERVER_IP]
kube-worker1    ansible_ssh_host=[YOUR_SERVER_IP]
kube-worker2    ansible_ssh_host=[YOUR_SERVER_IP]


[kube-master]
kube-master[0:2]

[etcd]
kube-master[0:2]

[kube-node]
kube-worker[0:2]

[kube-ingress]

[k8s-cluster:children]
kube-master
kube-node
kube-ingress
```


## $(pwd)/kubespray/group_vars/all.yml

- 確認遠端安裝機器的 python 路徑，並修改 **ansible_python_interpreter** 參數(預設是 `/usr/bin/python3.5`)

- 確認遠端機器的 OS，並修改 **bootstrap_os** 參數(預設是 `ubuntu`)

最後要設定安裝用 VM 的存取方式，分為以下兩種狀況：

### 使用 SSH Key 存取

- 確認 SSH Key 為 **$(pwd)/ssh-privkey/id_rsa**

## 使用帳號 + 密碼存取

- 將 **ansible_ssh_pass** 參數的註解取消，並輸入存取密碼

- **ansible_user** 參數請設定為登入帳號

- 移除(or 註解) **ansible_ssh_private_key_file** 參數


## $(pwd)/kubespray/group_vars/k8s-cluster.yml

若有需要調整安裝的版本，目前預設是 `v1.11.3`，可以調整 **kube_version** 來變更。



執行安裝程式
==========

當以上設定都完成後，就可以開始執行安裝程式：

```bash
# 若按照上面的操作，沒有離開原本的目錄的話，應該就是位於下面的目錄中
$ cd /tmp/kubernetes-installation-template

# 開始安裝
$ bash start.sh
```

整個安裝過程大約需要耗費十來分鐘(端看網路 & VM 運行速度)。

安裝完之後可以在執行安裝指令的機器(**bootstrapper**)上找到 kubeconfig，位置在 `/tmp/kubernetes-installation-template/kubeconfig/admin.conf`。


操作 Kubernetes
==============

要操作 k8s 可以直接連到 master node 上直接使用 kubectl 進行操作，也可以使用其他機器進行操作。

但若要使用非 master node 的機器對 k8s 進行操作，必須先準備好以下環境：

1. 安裝 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

2. 準備好 kubeconfig (以上例來說，位於 `/tmp/kubernetes-installation-template/kubeconfig/admin.conf`)，檔案路徑為 `~/.kube/config`

完成後執行簡單的指令：

```bash
$ kubectl get nodes
NAME            STATUS    ROLES          AGE       VERSION
kube-master0    Ready     master         1h        v1.11.2
kube-master1    Ready     master         1h        v1.11.2
kube-master2    Ready     master         1h        v1.11.2
kube-worker0    Ready     node           1h        v1.11.2
kube-worker1    Ready     node           1h        v1.11.2
kube-worker2    Ready     node           1h        v1.11.2
```

這樣大致上基本的 Kubernetes 就安裝完成了!

> 若是要取得安裝時所產生的相關憑證檔案，可以到 master node 上的 `/etc/kubernetes/ssl` 目錄中尋找


References
==========

- [kubernetes-incubator/kubespray: Setup a kubernetes cluster](https://github.com/kubernetes-incubator/kubespray)