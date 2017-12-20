---
layout: post
title:  "[RHCE] RH124 Chapter 11 Managing Red Hat Enterprise Linux Networking 學習筆記"
description: "此文章記錄學習 RHCE RH124 Chapter 11 Managing Red Hat Enterprise Linux Networking 留下的內容"
date: 2016-04-26 10:10:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---

<a name="ch11.1" />
11.1 Network Concepts
=====================

## 11.1.2 Network interface names

網卡命名原則：

- Ethernet 介面卡，開頭為 `en`
> onboard 的網卡名稱為 eno1, eno2 ... etc
> 可插拔(PCI 介面)的網卡名稱為 enp2s0

- 無線網路卡，開頭為 `wl`

- 3G/4G 網路卡，開頭為 `ww`

> 虛擬機則一律為 `eth0`, `eth1`, `eth2` ... etc

-----------------------------------------

<a name="ch11.2" />
11.2 Validating Network Configuration
=====================================

``` bash
# 顯示 enp0s8 資訊
[student@server0 ~]$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:00:00:0b brd ff:ff:ff:ff:ff:ff
    inet 172.25.0.11/24 brd 172.25.0.255 scope global dynamic eth0
       valid_lft 16494sec preferred_lft 16494sec
    inet6 fe80::5054:ff:fe00:b/64 scope link
       valid_lft forever preferred_lft forever

# 顯示網路介面的統計紀錄
[student@server0 ~]$ ip -s link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 52:54:00:00:00:0b brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast   
    5641965    4752     0       0       0       0      
    TX: bytes  packets  errors  dropped carrier collsns
    611742     3112     0       0       0       0

# 檢視路由
[student@server0 ~]$ ip route
default via 172.25.0.254 dev eth0  proto static  metric 1024
172.25.0.0/24 dev eth0  proto kernel  scope link  src 172.25.0.11
172.25.253.254 via 172.25.0.254 dev eth0  proto static  metric 1
```

```bash
# 列出所有資訊
$ sudo ss

# 列出已經建立的 connection
$ sudo ss -t

# 列出包含 listen 的 port & connection
$ sudo ss -ta

# 列出 listening 的 tcp socket
$ sudo ss -lt
```

-----------------------------------------

<a name="ch11.3" />
11.3 Configuring Network with nmcli
===================================

在 RHEL 7 中提供了 **<font color='red'>nmcli</font>**(NetworkManager) 作為網路設定管理之用。

## 11.3.1 Network Manager

`nmcli` 命令是修改 `/etc/sysconfig/network-scripts/ifcfg-*` 中的內容，有兩個觀念必須弄清楚，分別是 **<font color='red'>device</font>** & **<font color='red'>connection</font>**：

- `device`：每一個網路卡(介面)都屬於一個 device

- `connection`：每一個 device 可以同時有多個 connection 設定(每次只有一種可以生效)，可快速因應在不同場景所需要的網路設定變更

## 11.3.2 Viewing network information with nmcli

- `sudo systemctl status NetworkManager.service`：檢查 Network Manager 目前服務狀態

- `nmcli connection show`：列出目前所有的 connection

- `nmcli connection show --active`：顯示出目前狀態為 active 的 connection

- `nmcli connection show "System eth0"`：顯示指定 connection 的詳細內容 (小寫的部分可以變更、大寫的部分無法變更)

```bash
[student@server0 ~]$ nmcli connection show --active
NAME         UUID                                  TYPE            DEVICE
System eth0  5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03  802-3-ethernet  eth0
[student@server0 ~]$ nmcli connection show "System eth0"
connection.id:                          System eth0
connection.uuid:                        5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03
connection.interface-name:              eth0
connection.type:                        802-3-ethernet
.....
GENERAL.NAME:                           System eth0
GENERAL.UUID:                           5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03
GENERAL.DEVICES:                        eth0
GENERAL.STATE:                          activated
.....
```

- `nmcli device status`：顯示目前 device 的狀態

- `nmcli device show eth0`：顯示指定 device 的詳細狀態

## 11.3.3 Creating network connections with nmcli

- `sudo nmcli connection add con-name "my-connect-name" type ethernet ifname eth0`
> device: eth0<br />
> connection: my-connect-name<br />
> 設定內容：DHCP

- `sudo nmcli connection add con-name "static" ifname eth0 type ethernet autoconnect no ip4 172.25.40.11/24 gw4 172.25.40.254`
> device: eth0 <br />
> connection: static <br />
> 設定內容: 開機時不套用 | IPv4 | IP: 172.25.40.11/24 | Gateway: 172.25.40.254 <br />
> **<font color='red'>但 connection add 無法增加 DNS 設定</font>**

- `sudo nmcli connection up static`：套用 "staic" connection 設定

- `sudo nmcli connection reload`：reload 所有的 connection(設定檔)，不會套用到網路介面上(設定完建議 reload 以確保設定有被 Network Manager 抓到)

## 11.3.4 Modifying network interfaces with nmcli

- `sudo nmcli connection modify "static" ipv4.dns 8.8.8.8`：在指定的 connection 中設定 DNS(作完要重新 up connection 才會生效)

- `sudo nmcli connection modify "static" +ipv4.dns 8.8.4.4`：在指定的 connection 中增加 DNS 設定

- `sudo nmcli connection modify "static" connection.autoconnect on`：設定開機自動套用指定 connection

- `sudo nmcli connection delete "static"`：刪除指定的 connection

- `sudo nmcli connection down`：網路斷掉後，Network Manager 會嘗試找到另外一個 autoconnect=on 的 connection 並套用其設定

- `sudo nmcli device disconnect eth0`：強制停用指定 device 的網路設定(不會自動套用設定)

- `sudo nmcli net off`：停止所有的網路介面

-----------------------------------------

<a name="ch11.4" />
11.4 Editing Network Configuration Files
========================================

直接修改 `/etc/sysconfig/network-scripts/ifcfg-*` 檔案中的內容，再使用 `sudo nmcli connection reload`，就可以讓 Network Manager 取得新的設定。

-----------------------------------------

<a name="ch11.5" />
11.5 Configuring Host Names and Name Resolution
===============================================

## 11.5.1 Changing the System host name

若 `/etc/hostname` 不存在，則系統在網卡被分配到 ip 後，就會進行一個 DNS 的反向查詢

hostname 可透過 `hostnamectl` 命令來設定

```bash
$ hostname
server5.example.com

$ sudo hostnamectl set-hostname sercerX.example.com

$ hostnamectl status
   Static hostname: sercerx.example.com
   Pretty hostname: sercerX.example.com
         Icon name: computer-vm
           Chassis: vm
        Machine ID: adf65a29af58497b8bb516fc6d366b8d
           Boot ID: 963e5bc0e26a42e8acf285616ed9c9b6
    Virtualization: kvm
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-327.3.1.el7.x86_64
      Architecture: x86-64
```

## 11.5.2 Configuring name resolution

> 若查詢簡短名稱，系統會自動戴上 `/etc/resolv.conf` 中的 `domain` or `search` 的值再查詢

使用 `getent` & `host` 測試 DNS 設定：

```bash
# getenv 主要以 IPv6 為主
$ getent hosts tw.yahoo.com
2406:2000:ec:601::1009 fd-fp3.wg1.b.yahoo.com tw.yahoo.com

# 沒有 IPv6 的設定，則回傳 IPv4
$ getent hosts ptt.cc
140.112.172.3   ptt.cc
140.112.172.4   ptt.cc
140.112.172.2   ptt.cc
140.112.172.11  ptt.cc
140.112.172.5   ptt.cc
140.112.172.1   ptt.cc

# 使用 host 查詢 google
$ host www.google.com
www.google.com has address 210.242.127.104
www.google.com has address 210.242.127.88
........
www.google.com has address 210.242.127.109
www.google.com has IPv6 address 2404:6800:4008:c01::6a

# 使用 getent 查詢 google
$ getent hosts www.google.com
2404:6800:4008:c04::6a www.google.com
```

另外，一般網路設定若使用 DHCP，會把原有的 DNS 設定覆蓋，若要避免此情況，可用 `sudo nmcli connection "System eth0" ipv4.ignore-auto-dns yes` 來避免這樣的狀況發生。

> 也可以用 nslookup 來測試 DNS，但需要額外加裝 `bind-utils` 套件
