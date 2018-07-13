---
layout: post
title:  "[Docker] MACVLAN Network 簡介"
description: "此篇文章介紹 Docker MACVLAN Network 及其運作方式"
date: 2018-07-13 15:45:00
published: true
comments: true
categories:
  - Docker
tags:
  - Docker
---

此篇文章介紹 Docker MACVLAN Network 及其運作方式


環境介紹
======

以下的測試將會在以下環境進行：

- OS：`Ubuntu 18.04`

- Docker: `18.03.1-ce`

網卡配置：

- eth0：`10.103.19.0/24`

- eth1：`trunk port (10.103.[17-18].0/24)`



Prerequisites & Limitations
===========================

使用 MACVLAN 功能，在環境上是有所限制的：

1. 大多數的 cloud provider 是不支援這個功能的，因為這會需要使用到實體的網路設備

2. 需要 Linux kernel 3.9 以上 or 4.0 以上

3. 承上，所以僅支援 Linux

4. 網卡必須開啟 `promiscuous mode`，網卡才可以設定多個 MAC address 上去


## 簡易檢測

MACVLAN 這個功能需要 Linux Kernel(`v3.9–3.19` and `4.0+`) 的支援，為了確保 MACVLAN 可用，必須做以下檢查：

```bash
# 掛載模組
$ modprobe macvlan
# 列出目前已經掛載的模組
$ lsmod | grep macvlan
macvlan                24576  0
```

上面若是有指令發生錯誤，或是第二個指令沒有回傳任何結果，就表示該 host 上的 Linux kernel 不支援 MACVLAN 的功能。



MACVLAN 簡介
===========

MACVLAN 允許你在主機的一個 NIC 上配置多個虛擬的 NIC，這些 NIC 有自己獨立的 MAC 地址，也可以配置上 IP address 進行通訊。在 MACVLAN 下的 VM 或者 container 的網路和Host 都在同一個網段中，共享同一個 broadcast domain。



Bridge v.s. MACVLAN
===================

## Bridge

Bridge 有以下特點：

- Bridge 是 layer 2 設備，僅用來處理 layer 2 的通訊

- Bridge 使用 MAC address table 來決定網路封包要怎麼 forward

- Bridge 會從 host 之間的通訊中的封包中學習 MAC address

- 可以是硬體設備，也可以是純軟體(例如：Linux Bridge)

以下是一個在 Linux Host 上，多個 VM 使用 bridge 相互通訊的狀況：

![Bridge Example](http://hicu.be/wp-content/uploads/2016/03/linux-bridge.png)


## MACVLAN

MACVLAN 有以下特點：

- 可讓使用者在同一張實體網卡上設定多個 Layer 2 address (一般就是 MAC address)

- 承上，帶有上述設定的 MAC address 的網卡稱為 sub interface；=而實體網卡則稱為 parent interface

- 可在 parent/sub interface 上設定的不只是 MAC address，IP address 同樣也是可以被設定

- sub interface 無法直接與 parent interface 通訊 (帶有 sub interface 的 VM or container 無法與 host 直接通訊)

- 承上，若 VM or container 需要與 host 通訊，那就必須額外建立一個 sub interface 給 host 用

- sub interface 通常以 `mac0@eth0` 的形式來命名以方便區別

以下用張圖來解釋一下設定 MACVLAN 後的樣子：

![MACVLAN](http://hicu.be/wp-content/uploads/2016/03/linux-macvlan.png)



MACVLAN Modes
=============

MACVLAN 共支援四種模式，分別是：

## Private

![MACVLAN Private Mode](http://hicu.be/wp-content/uploads/2016/03/linux-macvlan-private-mode.png)

在 private mode 下，sub interface 之間無法相互通訊

## VEPA

![MACVLAN VEPA Mode](http://hicu.be/wp-content/uploads/2016/03/linux-macvlan-802.1qbg-vepa-mode.png)

在此 VEPA mode 下， sub interface 的通訊必須透過外部的 switch 來完成，而且此 switch 必須支援 `IEEE 802.1Qbg` 協定。

VM or container 之間的通訊透過外部的 switch，因此廠商可以在外部的 switch 上針對此類的流量進行優化設定，以達到更好的效能。

## Bridge

![MACVLAN Bridge Mode](http://hicu.be/wp-content/uploads/2016/03/linux-macvlan-bridge-mode.png)

sub interface 之間的通訊在 host 之間完成(類似上面使用 Linux bridge 時 VM or container 之間的通訊方式)，且不用 Linux bridge，因此也沒有 MAC learning，也不需要 STP，因此效能比起使用 Linux bridge 好上很多。

## Passthru

![MACVLAN Passthru Mode](http://hicu.be/wp-content/uploads/2016/03/linux-macvlan-passthru-mode.png)

直接把實體網卡分配給單一 sub interface，因此使用此 sub interface 的 VM or container 可以自行修改網卡的 MAC address or 相關參數。 



在 Docker 上使用 MACVLAN
======================

使用 MACVLAN 的 container 會有以下幾個特點：

1. 由於 container 的 interface 與 host NIC 連接，因此即使沒有 iptables or port mapping 的相關設定，就可以連外(只要 gateway 設定正確即可)

2. 效能比起 bridge 方式相對好

3. 若 container 很多，可能造成 IP 耗盡的狀況

4. 需要自行管理眾多不同的 MAC address


## Bridge Example

建立 container 並檢視結果：

```bash
# 根據 parent NIC 的網路配置，建立一個 MACVLAN network
$ docker network create -d macvlan \
  --subnet=10.103.19.0/24 \
  --gateway=10.103.19.1 \
  -o parent=eth0 \
  my-macvlan-net
8e9ae1c6629de975356184f8d6959a490dbaf351a8ed5334ba050049a9a5c3d2

# 確認 docker MACVLAN network 已經被正確建立
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
a1c7b6f80389        bridge              bridge              local
dc2f51e1056f        host                host                local
8e9ae1c6629d        my-macvlan-net      macvlan             local
f28460d3a620        none

# 啟用一個 container，並使用上面建立的網路
$ docker run --rm -itd \
  --network my-macvlan-net \
  --name my-macvlan-alpine \
  alpine:latest \
  ash

# 檢視 container 詳細資訊
$ docker container inspect my-macvlan-alpine
[
... (略)
"Networks": {
    "my-macvlan-net": {
        "IPAMConfig": null,
        "Links": null,
        "Aliases": [
            "bb4d75902a4e"
        ],
        "NetworkID": "8e9ae1c6629de975356184f8d6959a490dbaf351a8ed5334ba050049a9a5c3d2",
        "EndpointID": "5e88fd818f4c294332e31caaf084f053b7de950281154f4f8e3587aaa3fc5ba4",
        "Gateway": "10.103.19.1",
        "IPAddress": "10.103.19.2",
        "IPPrefixLen": 24,
        "IPv6Gateway": "",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        # 這個 MAC address 與 host eth0 的 MAC address 並不相同 (新產生的)
        "MacAddress": "02:42:0a:67:13:02",
        "DriverOpts": null
    }
... (略)
]
```

最後從 container 內部來檢視一下網路情況：

```bash
# 檢視 IP，跟上面看到的是相同的
$ docker exec -it my-macvlan-alpine ip addr show eth0
25: eth0@if2: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:0a:67:13:02 brd ff:ff:ff:ff:ff:ff
    inet 10.103.19.2/24 brd 10.103.19.255 scope global eth0
       valid_lft forever preferred_lft forever

# 檢視 routing information
$ docker exec -it my-macvlan-alpine ip route
default via 10.103.19.1 dev eth0 
10.103.19.0/24 dev eth0 scope link  src 10.103.19.2 

# 測試對 gateway 的通訊
$ docker exec -it my-macvlan-alpine ping -c 2 10.103.19.1
PING 10.103.19.1 (10.103.19.1): 56 data bytes
64 bytes from 10.103.19.1: seq=0 ttl=64 time=0.309 ms
64 bytes from 10.103.19.1: seq=1 ttl=64 time=0.317 ms

--- 10.103.19.1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.309/0.313/0.317 ms

# 測試連外通訊 & DNS resolution
$ docker exec -it my-macvlan-alpine ping -c 2 www.google.com
PING www.google.com (216.58.200.36): 56 data bytes
64 bytes from 216.58.200.36: seq=0 ttl=54 time=7.995 ms
64 bytes from 216.58.200.36: seq=1 ttl=54 time=7.876 ms

--- www.google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 7.876/7.935/7.995 ms
```

## 802.1Q Trunked Bridge Example

建立 container 並檢視結果：

```bash
# interface 預設是 down，因此要把它啟用
$ ifconfig ens19 up

# 建立 802.1Q MACVLAN network
$ docker network create -d macvlan \
  --subnet=10.103.18.0/24 \
  --gateway=10.103.18.1 \
  -o parent=ens19.1318 \
  my-8021q-1318-macvlan-net

# 檢視 docker network
$ docker network ls
NETWORK ID          NAME                        DRIVER              SCOPE
e9a149acc74d        bridge                      bridge              local
dc2f51e1056f        host                        host                local
3cf96bd7e0d1        my-8021q-1318-macvlan-net   macvlan             local
ef98674855e0        my-macvlan-net              macvlan             local
f28460d3a620        none                        null                local

# 使用上面的 MACVLAN 建立 container
$ docker run --rm -itd \
  --network my-8021q-1318-macvlan-net \
  --name my-8021q-1318-macvlan-alpine \
  alpine:latest \
  ash

# 檢視 container 詳細內容
$ docker container inspect my-8021q-1318-macvlan-alpine
[
.... (略)
        "NetworkSettings": {
            "Bridge": "",
            "SandboxID": "a09cfbaa619c387b2647fd8236a28b4a40ca7473048faf5c1ca723897f82e001",
            "HairpinMode": false,
            "LinkLocalIPv6Address": "",
            "LinkLocalIPv6PrefixLen": 0,
            "Ports": {},
            "SandboxKey": "/var/run/docker/netns/a09cfbaa619c",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "",
            "Gateway": "",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "",
            "IPPrefixLen": 0,
            "IPv6Gateway": "",
            "MacAddress": "",
            "Networks": {
                "my-8021q-1318-macvlan-net": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": [
                        "0192eb5284c1"
                    ],
                    "NetworkID": "3cf96bd7e0d18dac332dc160aecdc212d3fe46a03622ca208a6ac3cb90c2debe",
                    "EndpointID": "8f44064144c68f31b273973b61375b5c41c6bbd9e9d9b5d1e6a3034101fdeba2",
                    "Gateway": "10.103.18.1",
                    "IPAddress": "10.103.18.2",
                    "IPPrefixLen": 24,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:0a:67:12:02",
                    "DriverOpts": null
                }
            }
        }
.... (略)
]
```

最後從 container 內部來檢視一下網路情況：

```bash
# 檢視 container IP 資訊，跟上面的詳細資訊相同
$ docker exec my-8021q-1318-macvlan-alpine ip addr show eth0
6: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:0a:67:12:02 brd ff:ff:ff:ff:ff:ff
    inet 10.103.18.2/24 brd 10.103.18.255 scope global eth0
       valid_lft forever preferred_lft forever

# 查詢 routing information
$ docker exec my-8021q-1318-macvlan-alpine ip route
default via 10.103.18.1 dev eth0 
10.103.18.0/24 dev eth0 scope link  src 10.103.18.2 

# 測試與 gateway 的通訊
$ docker exec my-8021q-1318-macvlan-alpine ping -c 2 10.103.18.1
PING 10.103.18.1 (10.103.18.1): 56 data bytes
64 bytes from 10.103.18.1: seq=0 ttl=64 time=0.379 ms
64 bytes from 10.103.18.1: seq=1 ttl=64 time=0.290 ms

--- 10.103.18.1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.290/0.334/0.379 ms

# 測試對外通訊 & DNS resolution
$ docker exec my-8021q-1318-macvlan-alpine ping -c 2 www.google.com
PING www.google.com (172.217.160.68): 56 data bytes
64 bytes from 172.217.160.68: seq=0 ttl=54 time=7.284 ms
64 bytes from 172.217.160.68: seq=1 ttl=54 time=7.223 ms

--- www.google.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 7.223/7.253/7.284 ms
```


結論
===

Bridge 是 docker 中提供的預設方式，但使用者如果希望 container 的網路可以跟 host 的環境放在一起，選擇 MACVLAN 也是一種不錯的方式。

Docker 提供了非常多的網路設定選項，可以讓使用者根據需求選擇合適的方案；弄清楚每一種不同的 network driver 合適的場景、優缺點以及使用方式，container technology 一定可以在提供開發者以及維運人員更多的彈性。



References
==========

- [圖解幾個與Linux網絡虛擬化相關的虛擬網卡-VETH/MACVLAN/MACVTAP/IPVLAN - CSDN博客](https://blog.csdn.net/jincm13/article/details/50351268)

- [Bridge vs Macvlan – HiCube](http://hicu.be/bridge-vs-macvlan)

- [Configure Network | Docker Documentation](https://docs.docker.com/network)

- [Use Macvlan networks | Docker Documentation](https://docs.docker.com/network/macvlan/)

- [linux 網絡虛擬化： macvlan @ 小廷的部落格 :: 痞客邦 ::](http://lyt0112.pixnet.net/blog/post/306813547)