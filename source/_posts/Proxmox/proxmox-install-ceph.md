---
layout: post
title: "[Proxmox] 安裝 Ceph Luminous"
description: "本篇文章介紹在 Proxmox 5.2.1 上安裝 Ceph Luminous 的正確流程"
date: 2018-10-11 17:15:00
comments: true
published: true
tags: 
  - Proxmox
  - Ceph
categories: 
  - Proxmox
---

本篇文章介紹在 Proxmox 5.2.1 上安裝 Ceph Luminous 的正確流程


今天新裝了一台 PVE 5.2.1，準備要把這台也加入原本的 Ceph cluster，於是先下了以下指令：

> pveceph install --version luminous

結果出現了下面的訊息：

```
W: (pve-apt-hook) !! WARNING !!
W: (pve-apt-hook) You are attempting to remove the meta-package 'proxmox-ve'!
W: (pve-apt-hook)
W: (pve-apt-hook) If you really you want to permanently remove 'proxmox-ve' from your system, run the following command
W: (pve-apt-hook) touch '/please-remove-proxmox-ve'
W: (pve-apt-hook) and repeat your apt-get/apt invocation.
W: (pve-apt-hook)
W: (pve-apt-hook) If you are unsure why 'proxmox-ve' would be removed, please verify
W: (pve-apt-hook) - your APT repository settings
W: (pve-apt-hook) - that you are using 'apt-get dist-upgrade' or 'apt full-upgrade' to upgrade your system
E: Sub-process /usr/share/proxmox-ve/pve-apt-hook returned an error code (1)
E: Failure running script /usr/share/proxmox-ve/pve-apt-hook
```

Google 了一下發現要先加入 `pve-no-subscription` 這個 repository 才可以(不要傻傻跟著上面的訊息移除了 `proxmox-ve`，整個 pve 都會不見)，因此設定 Ceph cluster 的正確步驟應該是如下：

```bash
# 加入 pve-no-subscription repository
$ add-apt-repository "deb http://download.proxmox.com/debian/pve $(lsb_release -cs) pve-no-subscription"

$ apt-get update

# 指定安裝 Ceph Luminous
$ pveceph install --version luminous

# 設定 Ceph cluster 所使用的網段
$ pveceph init --network 10.103.2.0/24

# 建立 ceph monitor daemon
$ pveceph createmon

# 加入 OSD
$ pveceph createosd /dev/sdb
$ pveceph createosd /dev/sdc
....(根據實際的環境加入所需要的 Disk)
```

步驟很簡單，PVE 已經幫大家完成很多事情了，以上幾個指令就可以把 Ceph Luminous 安裝完成囉!


References
==========

- [Ceph install problem | Proxmox Support Forum](https://forum.proxmox.com/threads/ceph-install-problem.47147/)

- [Proxmox - Host System Administration](https://pve.proxmox.com/pve-docs/chapter-sysadmin.html#sysadmin_package_repositories)