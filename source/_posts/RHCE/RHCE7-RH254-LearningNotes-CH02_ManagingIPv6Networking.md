---
layout: post
title:  "[RHCE7] RH254 Chapter 2 Managing IPv6 Networking Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 2 Managing IPv6 Networking 留下的內容"
date: 2016-05-24 04:55:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH254]
---

老師補充
=======

### 設定網路卡以舊的方式呈現(eth0, eth1...)

1. 編輯 `/boot/grub2/grub.cfg`

2. 尋找 kernel 參數列：尋找 `linux16` 開頭的設定

3. 加上參數 `biosdevname=0 net.ifnames=0`，重開機後即完成

### ip 指令可達成的功能

- 不考慮 network namespace 的前提下，Linux kernel 有 256 個 routing table

- 設定 tc(traffic control) 的功能

- `ip route show`：可查詢 default gateway

- 查詢 network namespace：`ip netns show`

- 切換到指定的 network namespace(**hidden**)：`ip exec netns hidden bash`

### nmcli

- 沒有加上 ip 資訊則表示設定為 DHCP：`nmcli connection add con-name office type ethernet ifname eth1`

- 把跟 eth1 相關的 connection 全部停掉：`nmcli device disconnect eth1`

- 把 DHCP 改為 static 的示範：`nmcli connection modify office ipv4.method manual ipv4.addresses "192.168.0.1/24 192.168.0.254"`

### IPV6

- `::1/128`：等同於 ipv4 的 **127.0.0.1/8**

- `::`：等同於 ipv4 的 **0.0.0.0**(for listen port check)

- `::/0`：default gateway ipv4 的 **0.0.0.0/0**

- `2000::/3`：`2000::/16` ~ `3ffff::/16`：為 ipv6 的 public ip

- `fd00::/8`：ipv6 所使用的 private ip

- `fe80::/64`：link local address，沒有 DHCP service 的時候會用這一段的 ip(避免 ip 衝突，Linux 會把 MAC address 嵌入到 ip 內)

- `ff00::/8`：作為 multicast 之用，等同 ipv4 的 **224.0.0.0/4**

- 顯示 ipv6 address：`ip -6 addr show`

- 顯示 ipv6 routing table：`ip -6 route show`

- 指定網卡 ping 其他 ipv6 ip(**%** 之後帶網卡名稱)：`ping6 ff02:;1%eth0`

- 持續監控傳遞延遲時間：`mtr 8.8.8.8`

### 其他

- 參數補齊功能要安裝 `bash-completion` 套件才會有

- 查詢 hostname 對應的 ip：`dig server0.example.com`

- 查詢 ip 對應的 hostname：`dig -x 172.25.0.11`
  - client 向 DHCP 要求 ip
  - client 取得 ip
  - client 拿 ip 向 DNS server 作反向名稱解析
  - DNS server 給 client 對應的 hostname
  - client 使用上一個步驟給的 hostname 作為自己的 hostname

> 透過 DHCP 取得 ip 的狀況下不會有 `/etc/hostname` 這個檔案

----------------------------------------------------

2.1 Reviewing of IPv4 Networking Configuration
==============================================

## 關於 autoconnect

若透過 nmcli 設定的 connection 有加上 `autoconnect yes` 的屬性，當執行 `sudo nmcli connection down [CONNECTION_NAME]` 時，nmcli 會自動帶起其他有 autoconnect yes 屬性的 connection。

## 關於 nmcli 的設定參數

- 設定好的 connection 會有相對應的檔案存放在 `/etc/sysconfig/network-scripts/ifcfg-[CONNECTION_NAME]` 中

- 詳細設定可以參考 man page `nm-settings(5)`

- 若要將原本為 DHCP 設定改成 static IP 指定，要特別加上 `ipv4.method manual`，並附上 IP 相關資訊，以下為範例：

> sudo nmcli connection modify "System eth0" ipv4.addresses "192.168.0.10/24 192.168.0.1" ipv4.method manual

- `nm-settings` 與 `ifcfg-*` 的對應：

| nmcli con mod | ifcfg-* file | 效果 |
|---------------|--------------|------|
| ipv4.method manual | BOOTPROTO=none  | IPv4 IP 以固定的方式指定 |
| ipv4.method auto | BOOTPROTO=dhcp | 以 DHCP 方式取得 ip |
| ipv4.addresses "192.168.0.10/24 192.168.0.1" | IPADDR0=192.168.0.10<br />PREFIX0=24<br />GATEWAY0=192.168.0.1 | 指定 IP, Netmask, and Gateway |
| ipv4.dns 8.8.8.8 | DNS0=8.8.8.8 | 修改 `/etc/resolv.conf` 中的設定，加入 `nameserver 8.8.8.8` |
| ipv4.dns-search example.com | DOMAIN=example.com  | 修改 `/etc/resolv.conf` 中的 `search` 設定 |
| ipv4.ignore-auto-dns true | PEERDNS=no | 忽略來自 DHCP 的 DNS server 資訊 |
| connection.autoconnect yes | ONBOOT=yes | 開機自動啟動 |
| connection.id eth0 | NAME=eth0 | 指定 connection 名稱 |
| connection.interface-name eth0 | DEVICE=eth0 | 指定 connection 所要綁定的裝置名稱 |
| 802-3-ethernet.mac-address xxxxx | HWADDR=xxxxx | 指定 connection 所要綁定裝置的 MAC address |

> 因為 nmcli 都會直接去改 `/etc/resolv.conf` 的設定，因此若是每個 connection 有不同的DNS 設定，建議直接去 `/etc/sysconfig/network-scripts/ifcfg-*` 檔案中修改 `DNSn` & `DOMAIN` 的設定值

## 2.1.6 Deleting a network connection

移除 connection 的指令為 `sudo nmcli connection delete [CONNECTION_NAME]`

同時間也會將此 connection 斷線，並移除 /etc/sysconfig/network-scripts 目錄中相對應的 ifcfg 檔案

## 2.1.7 Modifying the system host name

hostname 相關指令：

- `hostnamectl status`：查詢 host 狀態

- `sudo hostnamectl set-hostname [HOSTNAME]`：變更 hostname

```bash
# 查詢 host 狀態
[student@server0 ~]$ hostnamectl status
   Static hostname: server5.example.com
         Icon name: computer
           Chassis: n/a
        Machine ID: 946cb0e817ea4adb916183df8c4fc817
           Boot ID: 87802c89e9a54f7087857bf5e6de16de
  Operating System: Red Hat Enterprise Linux Server 7.0 (Maipo)
       CPE OS Name: cpe:/o:redhat:enterprise_linux:7.0:GA:server
            Kernel: Linux 3.10.0-123.el7.x86_64
      Architecture: x86_64

# 修改 hostname
[student@server0 ~]$ sudo hostnamectl set-hostname server1.example.com
[student@server0 ~]$ hostnamectl status
   Static hostname: server1.example.com
         Icon name: computer
           Chassis: n/a
        Machine ID: 946cb0e817ea4adb916183df8c4fc817
           Boot ID: 87802c89e9a54f7087857bf5e6de16de
  Operating System: Red Hat Enterprise Linux Server 7.0 (Maipo)
       CPE OS Name: cpe:/o:redhat:enterprise_linux:7.0:GA:server
            Kernel: Linux 3.10.0-123.el7.x86_64
      Architecture: x86_64

$ cat /etc/hostname
server1.example.com
```

> 透過 hostnamectl 修改 hostname 之後，會反映到 /etc/hostname 的內容上

----------------------------------------------------

Practice: Configuring IPv4 Networking
=====================================

## 目標

1. 透過 nmcli 建立一個 connection，名稱為 `eno1`

2. 設定 connection 使用的網路介面為 `eno1`，IP `192.168.0.1/24`，沒有 Gateway

3. 進行 DNS 查詢時，會把 `otherhost` 解析為 `192.168.0.254`

## 實作過程

```bash
$ sudo nmcli connection add con-name eno1 ifname eno1 type ethernet autoconnect yes ip4 192.168.0.1/24
Connection 'eno1' (4b182e51-8798-406c-a07c-c63289733543) successfully added.
$ sudo nmcli connection up eno1
 successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/2)

 [student@server0 ~]$ ip addr show eno1
 6: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
     link/ether f6:0f:ec:b7:98:e1 brd ff:ff:ff:ff:ff:ff
     inet 192.168.0.1/24 brd 192.168.0.255 scope global eno1
        valid_lft forever preferred_lft forever
     inet6 fe80::f40f:ecff:feb7:98e1/64 scope link
        valid_lft forever preferred_lft forever

$ echo "192.168.0.254 otherhost" | sudo tee --append /etc/hosts
192.168.0.254 otherhost

$ ping otherhost
PING otherhost (192.168.0.254) 56(84) bytes of data.
64 bytes from otherhost (192.168.0.254): icmp_seq=1 ttl=64 time=0.055 ms
♥
--- otherhost ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.055/0.055/0.055/0.000 ms

# 顯示目前有那些額外的 network namespace
$ ip netns show
hidden
# 切換到指定的 network namespace
$ ip exec netns hidden bash
```

----------------------------------------------------

2.2 IPv6 Networking Concepts
============================

## 2.2.1 IPv6 overview

IPv6 目的是要解決 IPv4 IP 不足的問題，但因為沒有一個簡單的方式可以直接讓 IPv6 獨立運作，因此目前屬於過渡時期的方式，稱為 `dual-stack`，即是讓 IPv4 與 IPv6 同時存在

## 2.2.2 Interpreting IPv6 addresses

IPv6 address 長度為 128 bits，以 16 進位顯示，每 16 個 bits 會用冒號隔開，格式大概如下：

> 2001:0DB8:02de:0000:0000:0000:0000:0e13

但這樣太難記了，所以可以用較簡單的表示方式：(**都以小寫表示**)

- 2001:db8:2de:0000:0000:0000:0000:e13
- 2001:db8:2de:000:000:000:000:e13
- 2001:db8:2de:00:00:00:00:e13
- 2001:db8:2de:0:0:0:0:e13
- 2001:db8:2de::e13

 以上這幾種所表示的 IPv6 address 都是同一個(**每個由冒號區隔的 group，開頭的 0 都可以省略不寫**)

> 雙冒號不能同時出現兩個，這樣會無法推斷出正確的 IPv6 address

> 若要指定特定的 port，則會以類似這樣的形式表示：`[2001:DB8:2de::e13]:80`

### IPv6 subnets

- IPv6 同樣也有 subnet 的概念，標準的 IPv6 `network prefix` 長度為 64 bits(`/64`)，其餘 64 bits 則是屬於 `interface ID`

- 從上面可知，每個 subnet 可包含的網路裝置數量是很驚人的.....(`2^64`)

- 實際應用上，64 bits 真是太大了，而且一個 subnet 當然也不夠用，因此使用上還是會切成 `/48` 居多，這樣就可以從只有一個 subnet 變成有 `65536(2^16)` 個 subnet 可用

![IPv6](https://lh3.googleusercontent.com/M6--ZzzQCdXEj7HPZ2HNgI304o4xkBiUwCq0WAKL8pNQSHyvWs9fRiOgjCV32b2AkoMwNmYXU5umlO9TVWL-CeQ6Kh4tZEXfhYQ8_f60JDc7bHwgt6wFGSoYRyRex7_UmYLL9HAoofc_5wi5P7p3aEhQVJds1ZVEjgdmF-2g6ZTQ2DMloS0R0s3vJelc7QG8j7unlYjRf8QEuzF-mRSwU5xcYtfy3yn66ytEoiE0eIxJDlPrzDKHiLmk1M6-RPlCJJHMmEO9Bd7mAweT4TKMkO6o4lB8QLjwkDSODKf_hOpBls_AvmVWJMrTrFD-9JV-KuYp5o5LSr18Hp5cwm4N4NkUrJgwGpf2bLBdshT0AHgt3k6863PVTSAkbZJyKqKP4_6Ot1rg5RZehlJnRqEEZFnXUaJbTZJZc4G3fd7w5zOjvmYIuYW6wUKz3Pu2-6ttyWFCzGbgqMytn1XOelwbUEkgP4WxV3IpZ-Ou3eQmfZROn1FHh9jZ84UgoYspcHRptpPgoxUP710J3OhVEDFOz825EUL3U4Huhcc_gPqSchZf27Hr1AoCsw48zMmwa66oCyDgBA2Ey2sY4DeDLPQF82l0-vR666c=w650-h232-no)

## 2.2.3 IPv6 address allocation

IPv6 跟 IPv4 相同也有一些特定的位址，作為特定目的使用，例如：`localhost`、`multicast`、`unitcast`....等等。

詳細資料可參考[IPv6 - 維基百科，自由的百科全書](https://zh.wikipedia.org/wiki/IPv6#IPv6.E4.BD.8D.E5.9D.80.E7.9A.84.E5.88.86.E9.A1.9E)

### Link-local addresses

IPv6 中，Link-local address 是用來讓 host 指定的網路中互相交流之用(**僅在內部**)，以 `fe80::` 為開頭，完整的位址還包含了網路介面卡的名稱，例如：`fe80::211:22ff:fwaa:bbcc%eth0`

### Multicast

在 IPv6 中已經沒有 broadcast，因此 multicast 扮演了一個更重要的角色，而 multicast 的位址為 `ff02::1`，後面還會帶上網路介面卡名稱，因此完整位址寫法是：`ff02::1%eth0`

## 2.2.4 IPv6 address configuration

IPv6 位址同樣也可以透過靜態指定 or 動態指定(使用 `DHCPv6` 服務)

### Static addressing

跟 IPv4 相同，每個 IPv6 subnet 會有保留位址供特定目的使用，因此在設定時要避開：

1. `0000:0000:0000:0000`：這位址是作為 routing 之用，以 **2001:db8::/64** 為例，此位址就是 `2001:db8::`

2. 從 `fdff:ffff:ffff:ff80` ~ `fdff:ffff:ffff:ffff` 這一段位址

### DHCPv6 configuration

因為 IPv6 沒有 broadcast，因此 DHCPv6 的運作原理跟 DHCPv4 就不太一樣。

DHCP client 會從自己的 link-local address 送出 DHCP request 到 all-dhcp-servers link-local multicast group 中(`ff02::1:2` port `547/UDP`)，而 DHCPv6 server 則會回到 DHCP client 的 link-local address port `546/UDP`

### SLAAC configuration

這是另一種透過 router 協助來完成位址配發的技術，詳細的資訊可以參考下列網址：

- [2012台網中心電子報─IPv6位址配發技術介紹](http://www.myhome.net.tw/2012_09/p03.htm)

- [傲笑紅塵路: IPv6自動組態配置(IPv6 Auto configuration)](http://www.lijyyh.com/2012/04/ipv6ipv6-auto-configuration.html)

----------------------------------------------------

2.3 IPv6 Networking Configuration
=================================

## 2.3.2 Adding an IPv6 network connection

```bash
$ sudo nmcli connection add con-name "eth2-ipv6" type ethernet ifname eth2 ip6 2001:db8:0:1::c000:207/64 gw6 2001:db8:0:1::1 ip4 192.168.2.7/24 gw4 192.168.2.1
Connection 'eth2-ipv6' (e7c97cdd-efea-43c2-a4d7-52b3bb4be4e2) successfully added.

$ sudo nmcli connection modify eth2-ipv6 +ipv6.dns 2001:4860:4860::8888
```

## 2.3.3 Modifying network connection settings for IPv6

修改的方式跟 IPv4 幾乎一模一樣：

```bash
# 修改 IP 資訊
$ sudo nmcli connection modify eth2-ipv6 ipv6.addresses "2001:db8:0:1::a00:1/64 2001:db8:0:1::1"

# 增加 DNS 資訊
$[student@server0 ~]$ sudo nmcli connection modify eth2-ipv6 +ipv6.dns 2001:4860:4860::8888
```

| nmcli con mod | ifcfg-* file | 效果 |
|---------------|--------------|------|
| ipv6.method manual | IPV6_AUTOCONF=none  | IPv6 IP 以固定的方式指定 |
| ipv6.method auto | IPV6_AUTOCONF=yes | 透過 SLAAC，從 router advertisements 取得 ip |
| ipv6.method dhcp | IPV6_AUTOCONF=no<br />DHCPV6C=yes | 使用 DHCPv6 取得 ip |
| ipv6.addresses "2001:db8::a/64 2001:db8::1" | IPV6ADDR=2001:db8::a/64<br />IPV6_DEFAULTGW=2001:db8::1 | 指定 IP, Netmask, and Gateway |
| ipv6.dns 8.8.8.8 | DNS0=8.8.8.8 | 修改 `/etc/resolv.conf` 中的設定，加入 `nameserver 8.8.8.8`(與 IPv4 相同) |
| ipv6.dns-search example.com | DOMAIN=example.com  | 修改 `/etc/resolv.conf` 中的 `search` 設定(與 IPv4 相同) |
| ipv4.ignore-auto-dns true | IPV6_PEERDNS=no | 忽略來自 DHCP 的 DNS server 資訊 |
| connection.autoconnect yes | ONBOOT=yes | 開機自動啟動 |
| connection.id eth0 | NAME=eth0 | 指定 connection 名稱 |
| connection.interface-name eth0 | DEVICE=eth0 | 指定 connection 所要綁定的裝置名稱 |
| 802-3-ethernet.mac-address xxxxx | HWADDR=xxxxx | 指定 connection 所要綁定裝置的 MAC address |

## 其他：IPv6 相關指令

檢視 IPv6 網路資訊：

- `ip addr show eth0`：可檢視 eth0 上 IPv6 相關資訊，尋找關鍵字 `inet6` 即可

- `ip -6 route show`：顯示 IPv6 部分的 routing table

trouble shooting 相關指令：

- `ping6 ff02::1%eth1`：ping **link-local address** & **link-local all-nodes multicast group**，需要帶上 network interface name

- `tracepath6 2001:db8:0:2::451`：查詢連線到指定 host 所走的路徑

- `ss -A inet -n` & `netstat -46n`：查詢目前 network socket 的狀態

----------------------------------------------------

Practice: Configuration IPv6 Networking
=======================================

## 目標

- 建立一個名稱為 `eno1` 的連線

- network interface 名稱為 `eno1`，設定 IP 為 `fddb:fe2a:ab1e::c0a8:1/64`，Gateway 為 `fddb:fe2a:ab1e::c0a8:fe`

## 實作過程

```bash
# 新增連線
$ sudo nmcli connection add con-name eno1 type ethernet ifname eno1 ip6 fddb:fe2a:ab1e::c0a8:1/64 gw6 fddb:fe2a:ab1e::c0a8:fe
Connection 'eno1' (13505591-aeb6-452d-acaa-8f32810eebb3) successfully added.

# 修改連線設定
$ sudo nmcli connection modify eno1 ipv6.method manual

# 啟用連線
$ sudo nmcli connection up eno1
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
```

----------------------------------------------------

Lab: Managing IPv6 Networking
=============================

## 目標

- 建立一個名稱為 `eno1` 的連線，使用 network interface 為 `eno1`

- IPv4 的位址為 `192.168.0.100/24`，IPv6 的位址為 `fddb:fe2a:ab1e::c0a8:64/64`

- 啟動連線是否都有取得正確 IP

- ping IPv4 gateway(`192.168.0.254`) & IPv6 gateway(`fddb:fe2a:ab1e::c0a8:fe`) 確認網路設定正確

## 實作過程

```bash
# 新增連線
$ sudo nmcli connection add con-name eno1 type ethernet ifname eno1 ip4 192.168.0.100/24 ip6 fddb:fe2a:ab1e::c0a8:64/64
Connection 'eno1' (82399d91-7ae1-4544-975c-ae43424de35a) successfully added.

# 指定 ip 設定方式為 manual
$ sudo nmcli connection modify eno1 ipv4.method manual ipv6.method manual\

# 啟動連線
$ sudo nmcli connection up eno1
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/2)

# 查詢 IP 資訊
$ ip addr show eno1
6: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether da:f4:04:59:60:2c brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.100/24 brd 192.168.0.255 scope global eno1
       valid_lft forever preferred_lft forever
    inet6 fddb:fe2a:ab1e::c0a8:64/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::d8f4:4ff:fe59:602c/64 scope link
       valid_lft forever preferred_lft forever

# ping IPv4 address
[student@server0 ~]$ ping -c 1 192.168.0.254
PING 192.168.0.254 (192.168.0.254) 56(84) bytes of data.
64 bytes from 192.168.0.254: icmp_seq=1 ttl=64 time=0.085 ms

--- 192.168.0.254 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.085/0.085/0.085/0.000 ms

# ping IPv6 address
$ ping6 -c 1 fddb:fe2a:ab1e::c0a8:fe
PING fddb:fe2a:ab1e::c0a8:fe(fddb:fe2a:ab1e::c0a8:fe) 56 data bytes
64 bytes from fddb:fe2a:ab1e::c0a8:fe: icmp_seq=1 ttl=64 time=0.187 ms

--- fddb:fe2a:ab1e::c0a8:fe ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.187/0.187/0.187/0.000 ms
```
