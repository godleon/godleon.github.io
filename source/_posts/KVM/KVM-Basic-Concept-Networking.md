---
layout: post
title:  "[Linux KVM] Linux KVM concept - Networking"
description: "This article introduces what should to learn about networking when learning Linux KVM virtualization"
date: 2016-08-10 22:00:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM]
---

此篇文章介紹學習 Linux KVM 時所需要了解的 Networking 相關知識

前言
====

現今的虛擬化技術中，網路的部分永遠都是比重相當高的一部分，許多大型的 service provider 透過網路提供各種不同的服務給使用者，讓使用者在 IT 科技的幫助下，生活更加便利。

也因為這樣，當網路發生故障時，這些公司的損失常常難以估計，因此網路架構的設計上，保有彈性 & 穩定性是相當重要的。

而現在 server 普遍都擁有多張網路卡，為了提高網路使用效率 & 增加備援功能，建議 **<font color='red'>透過 Linux bonding 的技術將所有的網路卡設定成為 single virtual channel</font>**。

> bonding mode 有分為 Mode 1(active-backup)、Mode 2(balance-xor)、Mode 4(802.3ad/LACP)、Mode 5(balance-tlb)，其中建議使用 `Mode 1(active-backup)` & `Mode 5(balance-tlb)`

-------------------------------------------------------------------------

KVM/QEMU 支援的 Network Type
============================

網路絕對是使用 vitual machine 時最不可或缺的一部分，而 QEMU 提供了 virtual machine 一共四種不同的 network type：

1. bridge

2. NAT

3. user mode networking

4. VT-d & SR-IOV (直接分配網路設備)

要透過 QEMU 配置 virtual machine 的網路，必須使用 `-net` 參數，若是完全沒使用任何的網路相關參數，會系統自動帶上 `-net nic -net user` 做為網路預設值。

而 QEMU 可以模擬那些網路卡呢? 肯定必須是主流且被廣泛支援的，可以用以下指令查詢：

```bash
$ kvm -net nic,model=?
qemu: Supported NIC models: ne2k_pci,i82551,i82557b,i82559er,rtl8139,e1000,pcnet,virtio
```

> 若沒指定任何網路相關參數，則預設會使用 RTL8139 網卡

-------------------------------------------------------------------------

設定第一張虛擬網卡
=================

為了要設定虛擬網卡，必須先了解 `-net nic` 相關參數：

> -net nic[,vlan=n][,macaddr=mac][,model=type] [,name=name][,addr=addr][,vectors=v]

- `-net nic`：表示這是一個網卡的設定 (**<font color='red'>必要!</font>**)

- `vlan=n`：將網卡連結到 ID 為 **<font color='red'>n</font>** 的 VLAN

- `macaddr=mac`：指定網卡 MAC address

- `model=type`：設定網卡的種類，預設為 **<font color='red'>rtl8139</font>**

- `name=name`：為網卡命名(但僅有在 QEMU monitor 看的到)

- `addr=addr`：指定網卡在 virtual machine 中的 PCI device address

- `vectors=v`：設定網卡設備的 MSI-X 向量的數量，用於 virtio 驅動的網卡上

簡單做個測試，我們透過以下指令指定網卡：

> kvm -vnc 0.0.0.0:1 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img -net nic,vlan=0,macaddr=52:53:00:11:12:13,model=e1000,addr=08 -net user

於是在系統中可以得到以下訊息：

![System NIC Information](https://lh3.googleusercontent.com/abxoQOVs8BrZGNtB7rvarf28_a0AomNu_UBJvLgblFG6uoAj2lWJnlMGMuujMu_V3ldlY2p2ZPYTQJa84180vlnBkGp4bkJmidQN29QBEO9zb9cWOOQOK5HTTOwglaXnEJ_WXdSnKO1xGFmKkdHZji1tulTDI_mihaOaghtA3YlT_I5ss4-OQbsdK2lIdLz_deaKnb5AcAQA9cmAt6rPaW0pocvL4THwvZY3d4QXOOMso96DPPTWntwl0qsHEd19BMg4B0uzuAbNlIVglYMX8k80YOTZxTQDsAFi16q5MDlgrTPDa0VY7UOoNFvI-7PWmktPf3RYSpXa1QQ6kmASVax3x3NCYC3pZW_HA27dxjwyA3GRM4IYtsjKA9EBjMkhpVZmHeoxAeCk9_gSXKMJUqEdg8u63JRm9xsJobEcOy_g_1YXmMPCc1v-8Mx1QFpwctMr5C7DXq8DI-C_Voz2Y8DGpz1p7prWgDb2j1v6daMfvJwJvtKAyw2p2HyulbsMuSZE3JNB6bsZLmJgi4WjvXT3rr32l1mJQcE64B4MK-ZLQMZjTA0g18g5w5wk5d83XtQiSahUc-5wGhudJ6l5sqqRwDQ1Nec=w771-h384-no)

在 QEMU monitor 則可以看到以下資訊：

![Network Info in QEMU monitor](https://lh3.googleusercontent.com/RPzHnMisxvsHZx5xpTctONaavpAXkcjUxfqHdaiNfwYL-RyspfM2dGW4rffxz7Z-NyzLHH1ChC0zGwjNes5AclVYiW4-8Bb8YOgg61jHs58jmMchhv_ICNNvwhm-vkyHmkffIukmKxX7BqufthPly1VgVMNI1J2x17l66DD2LicTEbGPQB-414rpc8tTR1l3pBPnoNPZuaxTNCXNYlkfmvQUdx1ltYfWusoEUsxhLckpakEJ_F8qCEGrJ-dmLgh6BbLjDNlDfqBix-bMJaBdGlyngC8XvWR-f45eTKUyPCukFb-oSFVFH1VspTRqsBgccbSH06fKlq12CDu0pNhBNbte6zFPt7oJMAFn_Yf2ENBWXQbFjRs2H0x8eaCcQ0MiXY01wUmsNYoGJsgmZgKJZ2jQdhLDXu4bjtoH-0DuZw1NSfKPuIkDks9FdQR0j-adxK_rOg9oqpTdNOBOkU_QPfqQJUejhwlTr5OwjLjGkD5XIYHKbXU72D66eDCZNTMVUGxWo_EkhVluszNlN1PiCzOdqlRK4StC0LVWUFscHLHwyEjHuwTwg0BEIRs91_-nD0DrJT5mZF7emdCRPStDDdWGdP9ftVY=w541-h153-no)

-------------------------------------------------------------------------

設定 Bridge Mode
================

## 設定參數說明

在 bridge mode 下，virtual machine 的網路可視為與 host machine 是相同的，可以有自己的獨立 ip，外部網路可以直接存取 virtual machine。

以下是建立 bridge mode NIC 的參數：

> -net tap[,vlan=n][,name=name][,fd=h][,ifname=name][,script=file][,downscript=dfile][,helper=helper]

- `tap`：表示使用 tap device(屬於 layer 2 的虛擬網路設備；附帶一提，`tun` 則是屬於 layer 3 的虛擬網路設備)

- `vlan=n`：指定使用的 VLAN ID

- `name=name`：設定網卡名稱(可從 QEMU monitor 中檢視，但大多數情況下可忽略)

- `fd=h`：連接到已經開啟 tap 的 file descriptor (**一般交給 QEMU 自行建立**)

- `ifname=name`：設定 tap device 名稱(未設定的話會由 QEMU 自行產生)

- `script=file`：指定啟動 virtual machine 時自動執行的 script

- `downscript=dfile`：指定關閉 virtual machine 時自動執行的 script

- `helper=helper`：指定啟動 virtual machine 時會在 host machine 上執行的 helper 程式(例如：建立 tap device)，一般可以忽略使用預設值即可

## 建立支援 bridge mode 的環境

### 1、檢查環境支援程度

在建立 bridge 模式的網路配置之前，我們必須先確定 host machine 已經掛載相關模組：

```bash
# 檢查是否有掛載 tun 模組
$ lsmod | grep tun
tun                    27141  1

# 確認目前的 user 有 /dev/net/tun 的讀寫權限
$ ls -al /dev/net/tun
crw-rw-rw-. 1 root root 10, 200 Aug  3 20:14 /dev/net/tun
```

### 2、在 host machine 上可建立 bridge device 的環境

環境說明：

- bridge device：`br0`

- physical device：`ens1f0`

- ip address：`10.20.190.2/24`

- gateway：`10.20.190.1`

以下要在 host machine 上建立一個 bridge device，並綁定到要連外的實體網卡，並讓 bridge device 成為連接 host machine 與外網的設備：

```bash
[root@localhost ~]# nmcli connection add type bridge ifname br0
[root@localhost ~]# nmcli connection modify bridge-br0 bridge.stp yes
# 以下請根據自身的 lab 環境做調整
[root@localhost ~]# nmcli connection modify bridge-br0 ipv4.method manual ipv4.address "10.20.190.2/24" ipv4.gateway "10.20.190.1" ipv4.dns 8.8.8.8
[root@localhost ~]# nmcli connection add type bridge-slave ifname ens1f0 master br0
[root@localhost ~]# nmcli connection delete ens1f0

# bridge device 設定完成後，可以檢查是否有與實體網卡連動
$ brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.2c600cb163d5       yes             ens1f0
virbr0          8000.5254001001dc       yes             virbr0-nic
```

設定完成後，實體網路卡會進入到 **<font color='red'>promiscuous mode</font>**，而 bridge device 則會進入到 **<font color='red'>forwarding state</font>**

```bash
$ dmesg | grep ens1f0
.....
[807464.149861] device ens1f0 entered promiscuous mode
.....
[807494.252510] br0: port 1(ens1f0) entered forwarding state
```

接著建立要進行 bridge 網路配置用的 script(**<font color='red'>/etc/qemu-if{up,down}</font>**) 並設定可執行權限：

```bash
# 建立 script (/etc/qemu-ifup)
$ cat <<\EOF >/etc/qemu-ifup
#!/bin/bash
#This is a qemu-ifup script for bridging.
#You can use it when starting a KVM guest with bridge mode network.

#set your bridge name
switch=br0

if [ -n "$1" ]; then
  #start up the TAP interface
  ip link set $1 up
  sleep 1
  #add TAP interface to the bridge
  brctl addif ${switch} $1
  exit 0
else
  echo “Error: no interface specified”
  exit 1
fi
EOF

$ cat <<\EOF >/etc/qemu-ifdown
#!/bin/bash
#This is a qemu-ifdown script for bridging.
#You can use it when starting a KVM guest with bridge mode network.
#Don’t use this script in most cases; QEMU will handle it automatically.

#set your bridge name
switch=br0

if [ -n "$1" ]; then
  # Delete the specified interfacename
  tunctl -d $1
  #release TAP interface from bridge
  brctl delif ${switch} $1
  #shutdown the TAP interface
  ip link set $1 down
  exit 0
else
  echo “Error: no interface specified”
  exit 1
fi
EOF

# 設定執行權限
$ chmod +x /etc/qemu-if{up,down}
```

> 若是目前使用者沒有 script 的 execute 權限，就無法新增 tap interface for bridge mode 喔!

### 3、啟動 bridge-mode virtual machine

當上述環境都準備好後，可使用以下指令啟動 virtual machine：

```bash
# -net nic -net tap => 指定 virtual machine 網卡為 bridge mode，建立 tap interface
# script=/etc/qemu-ifup => 指定 script 配置網卡
# downscript=/etc/qemu-ifdown => 指定 script 移除網卡 (若設定為 no 則 QEMU 會自動協助處理)
# --daemonize => QEMU 程序背景執行
$ kvm -vnc 0.0.0.0:1 -m 2048 /kvm/storage/vm_disks/ubnutu1604.img -net nic -net tap,script=/etc/qemu-ifup,downscript=/etc/qemu-ifdown --daemonize

# 查詢是否有啟動 tap interface 並與 bridge device 相連
$ brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.26503b3a5e74       yes             ens1f0
                                                        tap0
virbr0          8000.5254001001dc       yes             virbr0-nic
```

當 virtual machine 啟動後(會在 DHCP 的地方卡一段時間)，在 virtual machine 中可以透過以下指令設定 IP：

```bash
# 假設新增的 network interface 為 ens3
$ sudo ip addr add 10.20.190.3/24 dev ens3
$ sudo ip route add 0.0.0.0/0 via 10.20.190.1 dev ens3
```

如此一來在網路的部分就可以等同 host machine 一樣，正常存取 Internet & 提供網路服務了

詳細的設定操作可以參考下圖：

![Configure static IP by using ip command](https://github.com/godleon/godleon.github.io/blob/master/_posts/images/2016/KVM-Basic-Concept-Networking/kvm_network-bridge-mode-configure-ip.PNG?raw=true)


-------------------------------------------------------------------------

設定 NAT mode
=============

## NAT

NAT 是為了解決 public ip address 不足而產生的，其詳細的原理這邊就不贅述了，可參考下列網址：

- [鳥哥的 Linux 私房菜 -- Network Address Transfer ( NAT ) 架設](http://linux.vbird.org/linux_server/0250simple_firewall/0320nat.php)

## 檢查 host machine 環境

### 1、檢查 bridge device

在 QEMU/KVM 中，使用 NAT mode 的 virtual machine 其實是透過 host machine 中的 **<font color='red'>virbr0</font>** 這個 bridge device 連外的，可以來檢視一下相關資訊：

```bash
$ ip addr show
1: lo: .......
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 2c:60:0c:c1:b9:2d brd ff:ff:ff:ff:ff:ff
3: ens1f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master br0 state UP qlen 1000
    link/ether 2c:60:0c:b1:63:d5 brd ff:ff:ff:ff:ff:ff
4: ens1f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 2c:60:0c:b1:63:d6 brd ff:ff:ff:ff:ff:ff
5: virbr0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 52:54:00:10:01:dc brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
       valid_lft forever preferred_lft forever
6: virbr0-nic: <BROADCAST,MULTICAST> mtu 1500 qdisc pfifo_fast master virbr0 state DOWN qlen 500
    link/ether 52:54:00:10:01:dc brd ff:ff:ff:ff:ff:ff
7: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 2c:60:0c:b1:63:d5 brd ff:ff:ff:ff:ff:ff
    inet 10.20.190.2/24 brd 10.20.190.255 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::2e60:cff:feb1:63d5/64 scope link
       valid_lft forever preferred_lft forever
```

從上面可以看出，QEMU/KVM 已經幫我們把 virbr0 設定完成，使用的是 `192.168.122.1/24` 的網段。

### 2、檢查 DHCP 服務

但關於 DHCP 的部分呢? QEMU/KVM 同樣也設定好 **<font color='red'>dnsmasq</font>** 幫我們處理好 ip 的分配 & traffic forwarding 的相關問題，若是要進一步的了解詳細的設定細節，可使用下面方式：

```bash
$ ps -aux | grep dns
nobody    4322  0.0  0.0  15552   888 ?        S    Jul27   0:01 /sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/libexec/libvirt_leaseshelper
root      4323  0.0  0.0  15524   176 ?        S    Jul27   0:00 /sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf --leasefile-ro --dhcp-script=/usr/libexec/libvirt_leaseshelper
root     16414  0.0  0.0 112652   976 pts/2    S+   06:02   0:00 grep --color=auto dns

[root@ocp-kvm-host ~]# cat /var/lib/libvirt/dnsmasq/default.conf
##WARNING:  THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BE
##OVERWRITTEN AND LOST.  Changes to this configuration should be made using:
##    virsh net-edit default
## or other application using the libvirt API.
##
## dnsmasq conf file created by libvirt
strict-order
pid-file=/var/run/libvirt/network/default.pid
except-interface=lo
bind-dynamic
interface=virbr0
dhcp-range=192.168.122.2,192.168.122.254
dhcp-no-override
dhcp-lease-max=253
dhcp-hostsfile=/var/lib/libvirt/dnsmasq/default.hostsfile
addn-hosts=/var/lib/libvirt/dnsmasq/default.addnhosts
```

### 3、檢查 traffic forward 設定

```bash
$ cat /proc/sys/net/ipv4/ip_forward
1
```

> ‵`1` 表示 traffic forward 的功能是開啟的

### 3、檢查 firewall 設定

最後要確認 host machine 可以將 virtual machine 對外的流量透過 virbr0 送出，因此要確認防火牆的設定：

```bash
# iptables -t nat -L -vn
Chain PREROUTING (policy ACCEPT 29 packets, 4041 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain INPUT (policy ACCEPT 5 packets, 1044 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 39 packets, 2906 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain POSTROUTING (policy ACCEPT 39 packets, 2906 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     all  --  *      *       192.168.122.0/24     224.0.0.0/24
    0     0 RETURN     all  --  *      *       192.168.122.0/24     255.255.255.255
    0     0 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    0     0 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24
```

> 從上面可看出，來自於 192.168.122.0/24 網段且對外的流量，會以 NAT 的形式連外

## 建立可運行 NAT mode 的環境

確認好目前的環境後，我們便根據環境設定了兩支 script，分別用來產生 & 移除 tap interface for NAT mode：

```bash
# 配置 tap interface for NAT mode 的 script
$ cat <<\EOF >/etc/qemu-ifup-NAT
#!/bin/bash

# network information
BRIDGE=virbr0
NETWORK=192.168.122.0
NETMASK=255.255.255.0
GATEWAY=192.168.122.1

if [ -n "$1" ]; then
  # 啟用 bridge device for NAT mode
  brctl stp ${BRIDGE} on
  ifconfig ${BRIDGE} ${GATEWAY} netmask ${NETMASK} up

  ifconfig "$1" 0.0.0.0 up
  brctl addif ${BRIDGE} "$1"
  exit 0
else
  echo "Error: no interface specified."
  exit 1
fi
EOF

# 移除 tap interface for NAT mode 的 script
$ cat <<\EOF >/etc/qemu-ifdown-NAT
#!/bin/bash

# network information
BRIDGE=virbr0

if [ -n "$1" ]; then
  echo "Tearing down network bridge for $1"
  ip link set "$1" down
  brctl delif ${BRIDGE} "$1"
  exit 0
else
  echo "Error: no interface specified."
  exit 1
fi
EOF

# 為目前的使用者加入
$ chmod +x /etc/qemu-if{up,down}-NAT
```

> 以上的 script 可能會因為自身的環境不同而有增減，需要特別注意一下

## 啟動 NAT-mode virtual machine

當環境都準備好後，可以使用以下指令啟動 virtual machine：

```bash
# -net nic -net tap => 設定 tap interface
# script=/etc/qemu-ifup-NAT,downscript=/etc/qemu-ifdown-NAT => 使用指定的 script 配置網路
$ kvm -vnc 0.0.0.0:1 -m 2048 /kvm/storage/vm_disks/ubnutu1604.img -net nic -net tap,script=/etc/qemu-ifup-NAT,downscript=/etc/qemu-ifdown-NAT --daemonize

# 從下面結果可看出 tap0 是與 virbr0 這個 bridge device 相連
$ brctl show
bridge name     bridge id               STP enabled     interfaces
br0             8000.2c600cb163d5       yes             ens1f0
virbr0          8000.1a507abb7dff       yes             tap0
                                                        virbr0-nic
```

最後確認一下 virtual machine 的狀況：

![NAT mode - virtual machine](https://github.com/godleon/godleon.github.io/blob/master/_posts/images/2016/KVM-Basic-Concept-Networking/kvm_network-NAT-mode-VM.PNG?raw=true)

從上圖可看出 virtual machine 的確拿到了 192.168.122.0/24 網段中的 ip address，表示 host machine 中的 dnsmasq 運作正常；此外連外也正常，表示防火牆的設定部分也沒有問題

-------------------------------------------------------------------------

設定用戶模式(user-mode)網路(內部網路)
===================================

若在啟動 virtual machine 沒加上 `-net` 參數，預設 QEMU 就會使用 `-net nic -net user` 把 virtual machine 設定為 user mode。

user mode network 完全是由 QEMU 透過 [SLiRP](http://wiki.qemu.org/Documentation/Networking) 所實現的一個虛擬 NAT 網路，不用依賴先前所安裝的 **bridge-utils**、**dnsmaq**、**iptables** 等工具，也不需要 root 的權限就可執行。

以下是 user mode network 的架構圖：

![SLiRP concept](http://wiki.qemu.org/images/9/93/Slirp_concept.png)

使用參數：`-net user[,option][,option][,...]`

以下列出比較常用的 option：

- `vlan=n`：將 user mode network 連接到 VLAN ID=n 的 VLAN

- `net=addr[/mask]`：設定 virtual machine 的 ip address (預設為 **<font color='red'>10.0.2.0/24</font>**)

- `host=addr`：設定 host machine 的 ip address (預設為 **<font color='red'>10.0.2.2</font>**)

- `dns=addr`：設定 virtual DNS 的 ip address (預設為 **<font color='red'>10.0.2.3</font>**)

- `dhcpstart=addr`：設定 DHCP 的第一個 ip address (預設為 **<font color='red'>10.0.2.15</font>**)

- `restrict=y|yes|n|no`：若設定 yes，virtual machine 將無法與 host machine 通訊，但不會影響 hostfwd 的設定

- `hostfwd=[tcp|udp]:[hostaddr]:hostport-[guestaddr]:guestport`：將 host machine 中的 hostport 導向 virtual machine 的 guestport (可一次定義多個 hostfwd 設定)

以下是簡單的應用範例：

```bash
# -net nic -net user => user mode network
# hostfwd=tcp::5022-:22 => 將 host machine 的 tcp port 5022 導向 virtual machine 的 tcp port 22
$ kvm -vnc 0.0.0.0:1 -m 2048 /kvm/storage/vm_disks/ubnutu1604.img -net nic -net user,hostfwd=tcp::5022-:22 --daemonize

# 使用 tcp port 5022 連線到 virtual machine
$ ssh -p 5022 ubuntu@10.20.190.2
ubuntu@10.20.190.2\'s password:
.......

# 查詢 ip address
ubuntu@vm-ubuntu1604:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe12:3456/64 scope link
       valid_lft forever preferred_lft forever

# 實際上是可以上網的
ubuntu@vm-ubuntu1604:~$ curl www.google.com.tw
..........(省略一大堆內容)
```

必須一提的是，使用 user mode network 也是有些缺點在的，像是：

1. 由於網路都是由 QEMU 虛擬出來的，因此效能比起其他模式相對差一些

2. 不支援部分網路功能(例如：ICMP)，所以也就無法使用 ping command

3. 無法從 host machine or 外部直接存取 virtual machine

-------------------------------------------------------------------------

References
==========

- [qemu-kvm(1): QEMU Emulator User Documentation - Linux man page](http://linux.die.net/man/1/qemu-kvm)

- [QEMU Emulator User Documentation](http://wiki.qemu.org/download/qemu-doc.html)

- [桥网络配置 - info5 - IT610.com](http://www.it610.com/article/4332744.htm)

- [KVM “qemu-ifup: could not configure /dev/net/tun: Operation not permitted”解决方案 – 笑遍世界](http://smilejay.com/2012/03/kvm_qemu_network_error/)
