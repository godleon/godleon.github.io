---
layout: post
title:  "使用 Kubespray 安裝 Kubernetes 1.10.4"
description: "此篇文章介紹如何使用 Kubespray 安裝 Kubernetes 1.10.4"
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

環境介紹
======

在這個安裝過程中，共有 8 個 VM，皆為透過 Ubuntu 16.04 cloud image 所產生，分別有以下幾個 node：

- Master Node x 3 (**同時兼任 etcd node**)

- Worker Node x 3

- Ingress Node x 2 (**同時兼任 worker node**)


準備安裝環境
==========

## 準備安裝環境

確認了環境之後，另外還要準備一台用來執行 Ansible 程式的機器(or VM)，以下以 docker container 的方式來執行：

```bash
$ docker run --name kubespray -td ubuntu:xenial /bin/bash

$ docker exec -it kubespray bash
```

接著在 ubuntu container 中執行以下 script：

```bash
#!/bib/bash

apt-get update
apt-get -y install git python3-pip python
pip3 install --upgrade pip

cd /tmp
git clone https://github.com/kubernetes-incubator/kubespray.git
cd -
pip install -r /tmp/kubespray/requirements.txt
```

如此一來 Kubespray 的安裝環境就準備好了


## 準備環境設定檔

接著要針對我們自己的環境來進行 Ansible inventory & variables 的相關設定：(以下內容皆在 container 中)

### /tmp/kubespray/inventory/mycluster/hosts.ini

```ini
kube-ingress0   ansible_host=[YOUR_SERVER_IP]   ip=[YOUR_SERVER_IP]
kube-ingress1   ansible_host=[YOUR_SERVER_IP]   ip=[YOUR_SERVER_IP]

kube-master0    ansible_host=[YOUR_SERVER_IP]   ip=[YOUR_SERVER_IP]
kube-master1    ansible_host=[YOUR_SERVER_IP]   ip=[YOUR_SERVER_IP]
kube-master2    ansible_host=[YOUR_SERVER_IP]   ip=[YOUR_SERVER_IP]

kube-worker0    ansible_host=[YOUR_SERVER_IP]   ip=[YOUR_SERVER_IP]
kube-worker1    ansible_host=[YOUR_SERVER_IP]   ip=[YOUR_SERVER_IP]
kube-worker2    ansible_host=[YOUR_SERVER_IP]   ip=[YOUR_SERVER_IP]


[kube-master]
kube-master[0:2]

[etcd]
kube-master[0:2]

[kube-node]
kube-worker[0:2]
kube-ingress[0:1]

[kube-ingress]
kube-ingress[0:1]

[k8s-cluster:children]
kube-master
kube-node
kube-ingress
```


### /tmp/kubespray/inventory/mycluster/group_vars/all.yml

```yaml
#### ....預設值忽略.... ####

bootstrap_os: ubuntu
ansible_user: ubuntu
ansible_become_user: root
ansible_ssh_private_key_file: /ansible/ssh-privkey/id_rsa
```


### /tmp/kubespray/inventory/mycluster/group_vars/k8s-cluster.yml

```yaml
#### ....預設值忽略.... ####

# 在 ansible host 上存放 kubeconfig
kubeconfig_localhost: true
```


安裝 Kubernetes
==============

最後執行以下指令即可安裝：

```bash
$ cd /tmp/kubespray
$ ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml
```

安裝完之後可以在執行 ansible 指令的機器上找到 kubeconfig，位置在 `/tmp/kubespray/inventory/mycluster/artifacts/admin.conf`


操作 Kubernetes
==============

要操作 k8s 可以直接連到 master node 上直接使用 kubectl 進行操作，也可以使用其他機器進行操作。

但若要使用非 master node 的機器對 k8s 進行操作，必須先準備好以下環境：

1. 安裝 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

2. 準備好 kubeconfig (以上例來說，位於 `/tmp/kubespray/inventory/mycluster/artifacts/admin.conf`)，檔案路徑為 `~/.kube/config`

完成後執行簡單的指令：

```bash
$ kubectl get nodes
NAME            STATUS    ROLES          AGE       VERSION
kube-ingress0   Ready     ingress,node   1h        v1.10.4
kube-ingress1   Ready     ingress,node   1h        v1.10.4
kube-master0    Ready     master         1h        v1.10.4
kube-master1    Ready     master         1h        v1.10.4
kube-master2    Ready     master         1h        v1.10.4
kube-worker0    Ready     node           1h        v1.10.4
kube-worker1    Ready     node           1h        v1.10.4
kube-worker2    Ready     node           1h        v1.10.4
```

這樣大致上基本的 Kubernetes 就安裝完成了!

> 若是要取得安裝時所產生的相關憑證檔案，可以到 master node 上的 `/etc/kubernetes/ssl` 目錄中尋找


References
==========

- [kubernetes-incubator/kubespray: Setup a kubernetes cluster](https://github.com/kubernetes-incubator/kubespray)