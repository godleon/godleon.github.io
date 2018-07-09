---
layout: post
title:  "Docker 網路簡介"
description: "此篇文章簡單介紹 docker network"
date: 2018-07-09 15:10:00
published: true
comments: true
categories:
  - Docker
tags:
  - Docker
---


剛開始使用 docker 的人可能不會對於 docker network 的細節有過多的研究，但若是要將 docker 的功能發揮的淋漓盡致，docker network 相關的知識可是必須要必備的。

以下將會介紹目前 docker 預設支援的網路類型，包含：

- bridge

- host

- overlay

- macvlan

- none

讓使用者可以知道在何種情況下，啟用 container 時應該要選擇哪一種網路類型。



Network Drivers
===============

為了讓 docker 支援多種網路類型，docker networking subsystem 被設計成是以可抽換 driver 的形式來運作，可根據使用者需求置換不同的 driver 來達到不同的網路設定目的；其中有以下5個 driver 已經是預設存在的，用來提供核心的網路功能：

- bridge

- host

- overlay

- macvlan

- none 



Bridge Network
==============

此為預設的 network driver(沒特別指定就是這個)，通常會用在當 container 都是以 standalone 的方式運作並與其他的 container 相互溝通時，架構圖如下：

![docker network - bridge](https://cdn-images-1.medium.com/max/1060/0*cMUND9w1bO1o5sPe.png)

- docker 會新增一個 software bridge 作為 container 網路對外的出口，預設名稱為 `docker0`

- docker0 會與 host 中的對外網卡(上圖為 `eth0`)相連，藉此取得對外連線的能力

- 每個 container 會使用一個 veth device 與 docker0 相連，因此具備連外能力



Host Network
============

此模式的網路有以下特點：

- 不使用獨立的網路(並非獨立的 network stack)，而是與 Docker Host 使用相同的網路

- 目前這個模式僅在 Linux 上有效，無法在 Mac or Windows 上使用

- 在 Docker 17.06 版以後可以在 swarm service 上使用此模式，但同一個 port 只能被用一次

- container 如果沒有對外開 port 提供服務，其實設定為 host mode 並沒有意義

網路架構如下圖：

![docker network - host](https://success.docker.com/api/images/.%2Frefarch%2Fnetworking%2Fimages%2Fhost-driver.png)



Overlay Network
===============

overlay network 是藉由將多個 docker daemon 連結起來，並啟用 swarm service 來讓多個 container 可以相互溝通，有以下特點：

- 需要 Docker Swarm

- 可讓在不同 host 上的 standalone container 互相溝通

- 可讓 standalone container 與 swarm service 互相溝通

由於 overlay mode 需要 Docker Swarm，因此會先忽略不深入討論，因為未來的學習方向會以 Kubernetes 為主。

Overlay network 的網路架構如下圖：

![docker network - overlay](http://img.scoop.it/1nNoIXGkJiDax7l5g5GxH7nTzqrqzN7Y9aBZTaXoQ8Q=)



MACVLAN Network
===============

macvlan network 可以讓使用者直接分配實體網卡的 MAC address 給特定的 container，讓 container 可以透過實體的網卡使用網路；有些需要獨立使用實體網卡的老舊應用程式，可能會用到 macvlan 來設定網路。

換個角度思考，若是要將上述老舊應用程式的網路與 Host 網路隔開，多加一張實體網卡並設定 macvlan 是個不錯的方式。

此外，若是實體網路接的是 switch 上的 trunk port，也可以透過 MACVLAN driver 將不同的 VLAN 分配給不同的 container 使用，而 MACVLAN 的網路架構如下：

![docker network - macvlan](http://img.scoop.it/zD6OR5JZu3qF9dxWL79Gc7nTzqrqzN7Y9aBZTaXoQ8Q=)



None
====

不設定 container 的網路，因此 container 無法對外通訊



References
==========

- [Configure Network | Docker Documentation](https://docs.docker.com/network)

- [Swarm mode overview | Docker Documentation](https://docs.docker.com/engine/swarm/)

- [Understanding Docker Networking Drivers and their use cases - Docker Blog](https://blog.docker.com/2016/12/understanding-docker-networking-drivers-use-cases/)

- [Docker源码分析（七）：Docker Container网络 （上） - DaoCloud](http://blog.daocloud.io/docker-source-code-analysis-part7-first/)
