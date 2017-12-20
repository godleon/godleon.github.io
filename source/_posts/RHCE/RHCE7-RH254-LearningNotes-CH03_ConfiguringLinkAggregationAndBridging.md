---
layout: post
title:  "[RHCE7] RH254 Chapter 3 Configuring Link Aggregation And Bridging Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 3 Configuring Link Aggregation And Bridging 留下的內容"
date: 2016-05-25 16:45:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH254]
---

老師補充
=======

### runner

- **activebackup**：同時間只有一張網卡提供服務

- **roundrobin**：outbound 頻寬可以合併，但 inbound 無法

- **broadcast**：封包往外送時，會往每張網卡都各送一份

- **loadbalance**：以 client ip 作 hash 後決定用哪個網卡提供網路服務

- **lacp**：進出都有 load balance

### teaming

透過 nmcli 修改 runner：`nmcli connection modify team0 team.config '{"runner": {"name": "activebackup"}}'`

-------------------------------------------------------

3.1 Configuring Network Teaming
===============================

## 3.1.1 Network teaming

Network Teaming 的用途是將兩個實體的網路卡結合成一張邏輯上的虛擬網卡，用來提供 failover & 更高的 throughput 目的之用，目前 RHEL7 中提供的 network teaming 比起以前的 bonding 有較好的效能以及擴充性(模組式設計)。

RHEL 7 中有一個 kernel driver(負責有效率的處理網路封包) 以及稱為 `teamd`(負責管理 interface) 的 user-space daemon 來負責實現 network teaming 的功能；其中還有稱為 `runner` 的軟體用來處理網路封包在多條實體網路間配送的問題。

RHEL 7 中以下支援五種 runner，分別是 `broadcast`、`roundrobin`、`activebackup`、`loadbalance`、`lacp`，其中以 **lacp** 的效率最好，但網路卡所連接的 switch 也要做相對應的設定才可以。

> bonding 消耗的資源比 teaming 還多，因此建議使用 teaming

## 3.1.2 Configuring network teams

設定 network teaming 共分為四個步驟：(以下直接用範例說明)

```bash
# 建立 team interface (runner 會使用 json 格式字串來指定)
$ sudo nmcli connection add con-name team0 type team ifname team0 config '{"runner": {"name": "loadbalance"}}'
Connection 'team0' (5d8173d9-fc07-4b11-a72c-5be10d1bea68) successfully added.

# 馬上就會出現 teaming interface(team0) 了
$ ip addr show team0
5: team0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 6e:68:9a:d9:75:e7 brd ff:ff:ff:ff:ff:ff

# 設定 IPv4 & IPv6 相關資訊，以及此 teaming interface 的其他屬性
$ sudo nmcli connection modify team0 ipv4.method manual ipv4.addresses 192.168.1.10/24

# 設定要做 network teaming 的網路卡
[student@server0 ~]$ sudo nmcli connection add con-name team0-eth1 type team-slave ifname eth1 master team0
Connection 'team0-eth1' (07fc837c-81d8-4f4e-b4c1-a66151128e17) successfully added.
[student@server0 ~]$ sudo nmcli connection add con-name team0-eth2 type team-slave ifname eth2 master team0
Connection 'team0-eth2' (70589bef-cd0d-4e6a-8ccc-fdba85733d1d) successfully added.

# 啟動 teaming 介面
$ sudo nmcli connection up team0
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)

# 檢視 teaming 介面狀態
$ sudo teamdctl team0 state
setup:
  runner: loadbalance
ports:
  eth1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
  eth2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up

# 若因特殊需求需要斷掉某個實體網路卡，可使用以下指令
$ sudo nmcli device connect eth1
```

-------------------------------------------------------

Practice: Configuring Network Teaming
=====================================

## 目標

1. 建立一個名稱為 `team0` 的 network team interface，使用 `eno1` & `eno2` 兩張網卡

2. runner 為 `activebackup`

3. IP 為 `192.168.0.100/24`

## 實作過程

```bash

$ sudo nmcli connection add con-name team0 type team ifname team0 config '{"runner": {"name": "activebackup"}}'
Connection 'team0' (fa5de783-c31d-4239-ab4c-613a0ed0e9c2) successfully added.

$ sudo nmcli connection modify team0 ipv4.method manual ipv4.addresses 192.168.0.100/24


$ sudo nmcli connection add con-name team0-eno1 type team-slave ifname eno1 master team0
Connection 'team0-eno1' (62f7a5d8-171d-457a-b113-af714042463e) successfully added.
$ sudo nmcli connection add con-name team0-eno2 type team-slave ifname eno2 master team0
Connection 'team0-eno2' (565abdf0-0131-40e5-b492-281a27baf540) successfully added.

$ sudo nmcli connection up team0
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)
[student@server0 ~]$ sudo teamdctl team0 state
setup:
  runner: activebackup
ports:
  eno1
    link watches:
      link summary: up
  instance[link_watch_0]:        name: ethtool
        link: up
  eno2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
runner:
  active port: eno2
```

-------------------------------------------------------

3.2 Managing Network Teaming
============================

## 3.2.2 Setting and adjusting team configuration

當 network team interface 設定完成後，後續還是可以進行修改調整，大部分與原有 nmcli connection modify 沒甚麼差別，主要是要更改 runner 的設定，需要加上 `team.config` 關鍵字。

一開始設定時使用一個簡單的 json 字串指定 runner 的設定，也可以透過一個較為複雜的 json 檔案來進行更細緻的 network teaming 設定，以下是個範例：

```json
{
    "device": "team0",
    "mcast_rejoin": {
        "count": 1
    },
    "notify_peers": {
        "count": 1
    },
    "ports": {
        "eno1": {
        "prio": -10,    /* -32729~32727，數字愈小優先權愈高 */
        "sticky": true, /* eno1 掛了會切到 eno2, eno1 回復後會再度主動切回 eno1(以 eno1 為優先使用的 NIC) */
            "link_watch": {
                "name": "arp_ping", /* 比 ethtool 準確 */
                "interval": 100, /* 100ms ping 一次 */
                "missed_max": 30, /* 掉封包超過 30 個以上 */
                "source_host": "192.168.23.2",  /* 需要給 ip 做檢測 */
                "target_host": "192.168.23.1"
            }
        },
        "eno2": {
            "link_watch": {
                "name": "ethtool"
            }
        }
    },
    "runner": {
        "name": "loadbalance"
    }
}
```

假設檔案名稱為 `/tmp/team.conf`，接著依序執行以下指令：

```bash
$ sudo nmcli connection modify team0 team.config /tmp/team.conf
$ sudo nmcli connection down team0
$ sudo nmcli connection up team0
$ sudo nmcli connection up team0-eno1
$ sudo nmcli connection up team0-eno2
```

> 重新啟動 network teaming interface，它所包含的 port 也要一起重新啟動

## 3.2.3 Troubleshooting network teams

```bash
# 查詢指定 network teaming interface 所相關連的 port$ sudo teamnl team0 ports
 4: eno1: up 10000Mbit FD
 6: eno2: up 10000Mbit FD

# 取得 active port
$ sudo teamnl team0 getoption activeport
6

# 取得 network teaming interface 所有相關資訊
$ sudo teamnl team0 options
 queue_id (port:eno2) 0
 priority (port:eno2) 0
 user_linkup_enabled (port:eno2) false
 user_linkup (port:eno2) true
 enabled (port:eno2) true
 queue_id (port:eno1) 0
 priority (port:eno1) 0
 user_linkup_enabled (port:eno1) false
 user_linkup (port:eno1) true
 enabled (port:eno1) true
 mcast_rejoin_interval 0
 mcast_rejoin_count 1
 notify_peers_interval 0
 notify_peers_count 1
 mode roundrobin

# 查詢 network team interface 的狀態
$ sudo teamdctl team0 state
setup:
  runner: activebackup
ports:
  eno1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
  eno2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
runner:
  active port: eno2

# 以 json 格式匯出 network teaming interface 的設定
$ sudo teamdctl team0 config dump > /tmp/team0.json
```

-------------------------------------------------------

Practice: Managing Network Teaming
==================================

## 目標

1. 現有一個 network team interface 為 `team0`，結合了 `eno1` & `eno2` 兩個 interface，並設定 IP 為 `192.168.0.100/24`，runner 為 `activebackup`

2. 將 runner 從 `activebackup` 改為 `roundrobin`

## 實作過程

```bash
# 匯出 network teaming interface 設定檔
$ sudo teamdctl team0 config dump >> /tmp/team.conf

# 將 runner 從 activebackup 改為 roundrobin，並套用設定
$ sudo sed -i 's/activebackup/roundrobin/g' /tmp/team.conf
$ sudo nmcli connection modify team0 team.config /tmp/team.conf

# 停用 connection
$ sudo nmcli connection down team0

# 啟用 connection
$ sudo nmcli connection up team0
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/7)

# 查雲 network teaming interface status (僅有 runner 資訊可看，要另外啟動兩個 port)
$ sudo teamdctl team0 state
setup:
  runner: roundrobin

# 重新啟動 participant port
$ sudo nmcli connection up team0-eno1
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/8)
$ sudo nmcli connection up team0-eno2
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/9)

# 重新檢視 network teaming interface 資訊
$ sudo teamdctl team0 state
setup:
  runner: roundrobin
ports:
  eno1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
  eno2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up

# 驗證連線
$ ping -c 1 -I team0 192.168.0.254
PING 192.168.0.254 (192.168.0.254) from 192.168.0.100 team0: 56(84) bytes of data.
64 bytes from 192.168.0.254: icmp_seq=1 ttl=64 time=0.142 ms

--- 192.168.0.254 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.142/0.142/0.142/0.000 ms
```

-------------------------------------------------------

3.3 Configuring Software Bridges
================================

## 3.2.2 Configure software bridges

新增一個 bridge `br0`，並與實體 NIC `eno1` & `eno2` 連接：

```bash
$ sudo nmcli connection add con-name br0 type bridge ifname br0
Connection 'br0' (8280bb56-62fb-4743-8ca3-86b3ed405c5d) successfully added.

$ sudo nmcli connection add con-name br0-eno1 type bridge-slave ifname eno1 master br0
Connection 'br0-eno1' (2cc68634-e2ca-4dcf-8485-69f619a77e37) successfully added.

$ sudo nmcli connection add con-name br0-eno2 type bridge-slave ifname eno2 master br0
Connection 'br0-eno2' (02df2464-0daf-4bc6-ad0f-c979323a5ef8) successfully added.

[student@server0 ~]$ sudo brctl show
bridge name	    bridge id		     STP enabled     interfaces
br0		        8000.9e32e837e9d6	 yes		     eno1
							                         eno2
```

> 詳細的使用方式可參考 `nmcli-examples(5)` & `brctl(8)`

建立完 bridge 後，會產生以下鄉對應的設定檔：

```bash
$ cat /etc/sysconfig/network-scripts/ifcfg-br0
DEVICE=br0
STP=yes
BRIDGING_OPTS=priority=32768
TYPE=Bridge
BOOTPROTO=dhcp
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=br0
UUID=8280bb56-62fb-4743-8ca3-86b3ed405c5d
ONBOOT=yes

$ cat /etc/sysconfig/network-scripts/ifcfg-br0-eno1
TYPE=Ethernet
NAME=br0-eno1
UUID=2cc68634-e2ca-4dcf-8485-69f619a77e37
DEVICE=eno1
ONBOOT=yes
BRIDGE=br0

$ cat /etc/sysconfig/network-scripts/ifcfg-br0-eno2
TYPE=Ethernet
NAME=br0-eno2
UUID=02df2464-0daf-4bc6-ad0f-c979323a5ef8
DEVICE=eno2
ONBOOT=yes
BRIDGE=br0
```

> Network Manager 僅支援將實體網卡連接到 bridge，不支援將 aggregate interface 加入到 bridge

-------------------------------------------------------

Practice: Configuring Software Bridges
======================================

##  目標

1. 建立一個名稱為 `br1` 的 bridge，並設定實體 NIC `eno1` 與其連接

2. 設定 static ip 為 `192.168.0.100/24`

## 實作過程

```bash
$ sudo nmcli connection add con-name br1 type bridge ifname br1 ip4 192.168.0.100/24
Connection 'br1' (cf675bd7-607e-445e-a952-3dcdab359636) successfully added.

$ sudo nmcli connection add con-name br1-eno1 type bridge-slave ifname eno1 master br1
Connection 'br1-eno1' (7f204182-dddf-4031-90dc-d8573059199f) successfully added.

$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 52:54:00:00:00:0b brd ff:ff:ff:ff:ff:ff
4: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master br1 state UP mode DEFAULT qlen 1000
    link/ether 4e:ea:9e:33:ac:ca brd ff:ff:ff:ff:ff:ff
6: eno2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether f6:a5:73:ed:7c:61 brd ff:ff:ff:ff:ff:ff
8: br1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT
    link/ether 4e:ea:9e:33:ac:ca brd ff:ff:ff:ff:ff:ff

$ brctl show
bridge name	    bridge id		     STP enabled	interfaces
br1		        8000.4eea9e33acca	 yes		    eno1

$ ping -I br1 -c 1 192.168.0.254
PING 192.168.0.254 (192.168.0.254) from 192.168.0.100 br1: 56(84) bytes of data.
64 bytes from 192.168.0.254: icmp_seq=1 ttl=64 time=0.054 ms

--- 192.168.0.254 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.054/0.054/0.054/0.000 ms
```

-------------------------------------------------------

Lab: Configuration Link Aggregation and Bridging
================================================

## 目標

1. 建立一個 network teaming interface `team0`，runner 為 `activebackup`，使用 `eno1` & `eno2` 兩個實體 NIC

2. 建立一個 bridge `brteam0`，連接步驟一設定好的 team0，並設定 IP 為 `192.168.0.100/24`

### 實作過程

設定 network teaming interface：

```bash
$ sudo nmcli connection add con-name team0 type team ifname team0 config '{"runner": {"name": "activebackup"}}'
Connection 'team0' (064b65f0-a794-41a1-819f-ead0ba4f03c5) successfully added.

$ sudo nmcli connection add con-name team0-eno1 type team-slave ifname eno1 master team0
Connection 'team0-eno1' (93004100-b241-4aa4-81d4-393995cb2147) successfully added.
$ sudo nmcli connection add con-name team0-eno2 type team-slave ifname eno2 master team0
Connection 'team0-eno2' (1b171dbd-2de4-4cdb-b7dc-5c597ba59797) successfully added.

$ sudo teamdctl team0 state
setup:
  runner: activebackup
ports:
  eno1
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
  eno2
    link watches:
      link summary: up
      instance[link_watch_0]:
        name: ethtool
        link: up
runner:
  active port: eno1
```

建立 bridge `brteam0`：

```bash
$ sudo nmcli connection add con-name brteam0 type bridge ifname brteam0 ip4 192.168.0.100/24
```

接著停止 network teaming interface & NetworkManager 服務：

```bash
$ sudo nmcli connection down team0

$ sudo systemctl stop NetworkManager.service
$ sudo systemctl disable NetworkManager.service
rm '/etc/systemd/system/multi-user.target.wants/NetworkManager.service'
rm '/etc/systemd/system/dbus-org.freedesktop.NetworkManager.service'
rm '/etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service'
```

修改 `/etc/sysconfig/network-scripts/ifcfg-*` 相關設定檔：

```bash
$ echo "BRIDGE=brteam0" | sudo tee --append /etc/sysconfig/network-scripts/ifcfg-team0

$ sudo sed -e '/^BOOTPROTO.*/d' -e '/^DEFROUTE.*/d' -e '/^PEER.*/d' -e '/^IPV.*/d' -i /etc/sysconfig/network-scripts/ifcfg-team0-eno1

$ sudo sed -e '/^BOOTPROTO.*/d' -e '/^DEFROUTE.*/d' -e '/^PEER.*/d' -e '/^IPV.*/d' -i /etc/sysconfig/network-scripts/ifcfg-team0-eno2
```

最後重啟網路並重開機：

```bash
$ sudo systemctl restart network

$ reboot

# 驗證是否成功
$ ping -I brteam0 -c 3 192.168.0.254
PING 192.168.0.254 (192.168.0.254) from 192.168.0.100 brteam0: 56(84) bytes of data.
64 bytes from 192.168.0.254: icmp_seq=1 ttl=64 time=0.146 ms
64 bytes from 192.168.0.254: icmp_seq=2 ttl=64 time=3.44 ms
64 bytes from 192.168.0.254: icmp_seq=3 ttl=64 time=0.148 ms

--- 192.168.0.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 0.146/1.245/3.442/1.553 ms
```
