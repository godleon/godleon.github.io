---
layout: post
title: "[Proxmox] 使用 cloud-init 快速產生 VM"
description: "This article introduces how to use cloud-init to generate multiple VMs in a short time"
date: 2018-06-07 09:50:00
comments: true
published: true
tags: 
  - Proxmox
  - cloud-init
categories: 
  - Proxmox
---

本文將會介紹如何在 Proxmox 上使用 cloud-init 產生 VM


前言
===

使用 [Proxmox] 也有一段時間了，這套免費的 KVM virtualization platform 真是佛心來的，好用穩定又有 Web GUI，但唯一的缺憾就是要作 Infrastructure as code 真的有點困難，因為他本身並不具備 cloud-init 的功能。

還好這個功能終於在 5.2 的時候被支援了，不過看起來還是很陽春，但基本使用上應該還算足夠。

> 希望未來會有 scheduler 也被開發出來，不然我都要自己指定 host 來擺 VM....


準備 Template VM
===============

以下將以 [ubuntu 16.04 cloud image](https://cloud-images.ubuntu.com/xenial/current) 做示範來準備 template VM

在準備 template VM 前必須有以下資訊：

1. VM ID：這個不要跟其他 VM 重複到即可，下面使用 `9999`

2. CPU / Memory / Network 相關設定：這個部份就根據自己的環境 & 需求調整

3. Storage：要選擇一個 Template VM disk 存放的 storage，以下以 `rbd_vm` 做示範 (官網的文章是直接放到 `local-lvm`)

4. SSH login key：cloud image 預設無法使用密碼登入，因此必須準備好登入用的 SSH Key(public)

接著執行以下 script 即可：

```bash
#!/bin/bash

cd /tmp

wget https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img

# 設定 CPU type 為 host, 共 8(2x4) vcpu, 8GB memory, 網路使用 vmbr1(tag 1310, 此為我自己設定的 trunk bridge)
qm create 9999 --cpu cputype=host --sockets 2 --cores 4 --memory 8192 --net0 virtio,bridge=vmbr1,tag=1310

# 將 cloud image 匯入到指定的 storage 作為 template VM 的第一個 disk
qm importdisk 9999 xenial-server-cloudimg-amd64-disk1.img rbd_vm

# 設定 VM 細節
# 設定 storage type
# 設定 cloud-init 的功能以 cd-rom 的形式掛載
# serial 一定要加，否則 cloud image 會無法正常開機
qm set 9999 --virtio0 rbd_vm:vm-9999-disk-1 --ide2 rbd_vm:cloudinit --boot c --bootdisk virtio0 --serial0 socket

# 設定 SSH key (cloud image 預設是無法使用密碼登入，必須設定 SSH key)
qm set 9999 --sshkey <your public ssk key path>

# 轉換成 template VM
qm template 9999
```

執行完成後就會有一個 VM ID=9999 的 template VM 產生，從系統上可以看到類似以下資訊：

！[Template VM Hardware Information](/images/proxmox/vm_hw_info.png)

！[Template VM cloud-init Information](../../../../../images/proxmox/vm_cloud-init_info.png)


接著就可以使用這個 template VM 來快速產生 VM 了，以下用個簡單的 script 來完成：

```bash
#!/bin/bash

# 複製 VM
qm clone 9999 1001 --name ubuntu1604-1

# 將 storage size 擴大到 64GB
qm resize 1001 virtio0 64G; 

# 網路設定 (以下是對應到上面的 tag 1310，請自行修改以對應自己的環境)
qm set 1001 --ipconfig0 ip=10.103.10.51/24,gw=10.103.10.1 --nameserver '8.8.8.8 1.1.1.1'

qm start 1001
```

以上的 script 修改一下，放到 loop 中，一下子要產生多個 VM 其實就是一件很簡單的事情了!



References
==========

- [Cloud-Init Support - Proxmox VE](https://pve.proxmox.com/wiki/Cloud-Init_Support)

- [qm(1)](https://pve.proxmox.com/pve-docs/qm.1.html)