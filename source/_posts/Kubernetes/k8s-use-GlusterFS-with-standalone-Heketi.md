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




References
==========

- [Gluster Storage System](http://www.l-penguin.idv.tw/book/Gluster-Storage_GitBook/)

- [heketi/heketi: RESTful based volume management framework for GlusterFS](https://github.com/heketi/heketi)

- [Heketi Documentation - Administration](https://github.com/heketi/heketi/blob/master/docs/admin/readme.md)

- [Prepare Heketi Topology](https://github.com/heketi/heketi/blob/master/docs/admin/topology.md)

- [独立部署GlusterFS+Heketi实现Kubernetes共享存储 - breezey - 博客园](https://www.cnblogs.com/breezey/p/8849466.html)

- [GlusterFS 操作備忘 « Jamyy's Weblog](http://jamyy.us.to/blog/2014/03/6220.html)

