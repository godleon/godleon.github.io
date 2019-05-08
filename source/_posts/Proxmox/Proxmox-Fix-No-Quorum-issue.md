---
layout: post
title: "[Proxmox VE] 修復 cluster 發生的 no quorum 錯誤"
description: "本篇文章介紹在 Proxmox VE cluster 中發生 cluster not ready – no quorum 錯誤時，要如何修復"
date: 2019-05-09 03:15:00
comments: true
published: true
tags: 
  - Proxmox
categories: 
  - Proxmox
---


今天同事突然跟我說內部 Redmine 服務掛點了，想說它是執行在 OpenShift 上，且這個 OpenShift 是由散佈在 Proxmox cluster 中的多個 VM 所組成，要掛掉實在不容易，結果登入 Proxmox web console，發現整個 cluster 完全都異常，每個 node 都出現錯誤，且對 VM 的操作都會出現以下錯誤訊息：

> TASK ERROR: cluster not ready – no quorum?

接著到每一台 node 上執行 `pvecm status`，會出現以下兩種情況：

1. `Cannot initialize CMAP service`

2. `Quorum: X Activity blocked` (其中 X 為某個數字)

很明顯 cluster status 在同步上有問題，上網找了一下資料，找到了解決方式，出現第一種錯誤的 node，執行以下指令後可以修復：

```bash
$ systemctl restart pve-cluster.service

$ systemctl restart pvedaemon.service

$ systemctl restart pveproxy.service

$ systemctl restart corosync.service 
```

而第二種錯誤的 node，則需要執行以下指令來修復：

```bash
$ pvecm expected 1

$ systemctl restart pve-cluster.service

$ systemctl restart pvedaemon.service

$ systemctl restart pveproxy.service

$ systemctl restart corosync.service 
```

執行以上指令讓 corosync 服務重新啟動，並抓取原本的 cluster 設定，就會恢復正常了。


References
==========

- [修復Proxmox VE：集叢未啟動 / Fix Proxmox VE: Cluster Not Ready - 布丁布丁吃什麼？](http://blog.pulipuli.info/2014/08/proxmox-ve-fix-proxmox-ve-cluster-not.html)

- [大佬们 问个proxmox集群的问题-美国VPS综合讨论-全球主机交流论坛 - Powered by Discuz!](https://www.hostloc.com/thread-394364-1-1.html)