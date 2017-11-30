---
layout: post
title:  "[Linux KVM] libvirt & network"
description: "This article introduces how to manage virtual network environment by libvirt in QEMU/KVM"
date: 2016-10-22 06:00:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM, libvirt, Network]
---

介紹如何使用 libvirt 來管理 KVM virtual network

Overview
========

以下圖片來說明 libvirt 在整個虛擬化架構中所扮演的角色：

![libvirt 1](http://smilejay.com/wp-content/uploads/2013/03/libvirt-manage-hypervisors.jpg)

> 其中 other tools 的部份，甚至可以是 **OpenStack**、**oVirt** ... 等工具


libvirt 的管理功能一共包含五個部分：

1. Virtual Machine
> 包含 VM lifecycle(啟動/停止/暫停/保存/恢復/Live Migration .... 等等) 的管理，也支援對各種設備的熱插拔(不同的 hypervisor 對熱插拔的支援程度不一)

2. Remote Node
> 只要 remote node 上執行了 libvirtd 服務，libvirt 就可以用遠端的方式進行管理，支援 SSH / TCP socket ... 等不同的連線方式，以 SSH 為例，可用像是 `virsh -c qemu+ssh://root@remotehost.com/system` 的命令進行連線

3. Storage
> 與上面相同，只要 remote node 上執行了 libvirtd 服務，就可以透過 libvirt 來管理不同類型的 storage，例如：建立不同格式的 virtual machine image(qcow2 / raw / vmdk ... 等等)、掛載 remote NFS/iSCSI share、磁碟分割 .... 等等

4. Network
> 與上面相同，只要 remote node 上執行了 libvirtd 服務，就可以透過 libvirt 進行像是配置 tap device、建立 virtual network tap、bridge device 設定、VLAN/NAT 管理...等功能

5. 提供穩定、高效 API interface

更清楚一點解釋 libvirt 在整個 QEMU/KVM 虛擬環境中所扮演的角色，可參考下圖：

![libvirt](https://github.com/godleon/godleon.github.io/blob/master/_posts/images/2016/KVM-libvirt/libvirt.png?raw=true)


-----------------------------------------------------------------

遠端操作 libvirtd
================

以下用個很簡單的範例，透過 virsh 命令連線到遠端主機的 libvirtd 並執行命令：

```bash
# 遠端主機為 10.20.190.3
# 執行命令 "list --all"
$ virsh --connect qemu+ssh://root@10.20.190.3/system list --all
 Id    Name                           State
----------------------------------------------------
 3     centos7                        running
```

用圖來表示的話，遠端操作 libvirtd 的流程如下：

![Remote Accrss libvirtd](https://github.com/godleon/godleon.github.io/blob/master/_posts/images/2016/KVM-libvirt/remote_access_libvirtd.png?raw=true)

-----------------------------------------------------------------

Linux Virtual Networking
========================

libvirt networking 的主要元件是 virtual network switch，也就是 Linux 中的 **<font color='red'>bridge devices</font>**，而連接在 bridge 上的 interface，我們稱為 **<font color='red'>TAP devices</font>**。

> TAP device 是由 Linux 的 TUN/TAP 模組所實作出來，其中 TUN(tunnel) 用來模擬 layer 3 的設備，而 TAP(network tap) 則是用來模擬 layer 2 設備

以下示範如何建立 bridge device & TAP device：

```bash
# 查詢是否已經掛載 bridge 模組
$ lsmod | grep bridge
bridge                119562  0
.....

# 新增 bridge device
$ brctl addbr tester
$ brctl show
bridge name	  bridge id		       STP enabled	interfaces
.......
tester		    8000.000000000000	 no		

# 檢查 TUN/TAP 模組是否已經載入到 kernel 中
$ lsmod | grep tun
tun                    27141  4 vhost_net

# 建立 TAP device(vm-vnic)
$ ip tuntap add dev vm-vnic mode tap
$ ip addr show vm-vnic
21: vm-vnic: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN qlen 500
    link/ether 92:ee:17:b7:2b:c4 brd ff:ff:ff:ff:ff:ff

# 將 TAP device 與 bridge 相連
$ brctl addif tester vm-vnic
$ brctl show
bridge name	bridge id		       STP enabled	 interfaces
.......
tester		  8000.92ee17b72bc4	 no		         vm-vnic

# 檢視目前連接到 bridge 的 TAP devices
$ brctl showmacs tester
port no	mac addr		is local?	ageing timer
  1	92:ee:17:b7:2b:c4	yes		   0.00
  1	92:ee:17:b7:2b:c4	yes		   0.00
```

若以上的 vm-vnic 連接到 VM 後，整個 network topology 會變成類似如下圖：

![Linux Bridge & TAP devices](https://github.com/godleon/godleon.github.io/blob/master/_posts/images/2016/KVM-Virtual-Networking/Linux_bridge_taps.png?raw=true)

測試完畢後，我們可以用以下指令移除上面建立的 TAP device & bridge：

```bash
$ brctl delif tester vm-vnic

$ brctl show
bridge name	bridge id		       STP enabled	interfaces
........
tester		  8000.000000000000	 no		

$ ip tuntap del dev vm-vnic mode tap
$ brctl delbr tester
```

-----------------------------------------------------------------

Virtual Network Types
=====================

在 KVM 中，virtual network 分為以下幾種類型：

- **NATed**：以 host 作為 NAT server 的方式提供網路

- **Routed**：透過設定在 hypervisor 上的 routing rules，允許 VM 與實體網路卡相連進行資料傳輸

- **Isolated**：與外界完全區隔的內部私有網路，只有在裡面的 VM 可以互相通訊

透過以下指令可以查詢 virtual network 相關的資訊：

```bash
$ virsh -c qemu+ssh://root@10.20.190.3/system net-list --all
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes

$ virsh --connect qemu+ssh://root@10.20.190.3/system net-info default
Name:           default  # virtual network 名稱
UUID:           83bea138-4c65-4ca2-8199-db3400d87fa5
Active:         yes      # 目前的啟動狀態
Persistent:     yes      
Autostart:      yes      # 是否隨著 libvird 一起啟動
Bridge:         virbr0   # 使用那個 device 進行對外通訊 

$ virsh --connect qemu+ssh://root@10.20.190.3/system net-dumpxml default
<network>
  <name>default</name>
  <uuid>83bea138-4c65-4ca2-8199-db3400d87fa5</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:f8:50:f2'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>

# 啟動 default virtual network
$ virsh net-start default

# 刪除 default virtual network
$ virsh net-destroy default
```

系統預設的 **<font color='red'>default</font>** virtual network 屬於 NATed，host 會自動派發 IP 給此網路中 VM，並設定相關的 firewall rule 讓 VM 可以透過 host 對外通訊。

-----------------------------------------------------------------

使用 libvirt 管理 virtual network - Isolated
===========================================

顧名思義，isolated network 表示外面是無法與在 isolated network 中的 VM 進行通訊的，只有加入到此 network 的 VM 才可以相互通訊，而在 isolated network 內部的 VM 也無法對外連線。

![KVM isolated network](https://github.com/godleon/godleon.github.io/blob/master/_posts/images/2016/KVM-libvirt/kvm_isolated-network.jpg?raw=true)

建立 isolated network 的方式如下：

```bash
$ bash -c 'cat <<EOF > /tmp/isolated.xml
<network> 
  <name>isolated</name>
</network>
EOF'

# 定義 virtual network 
$ virsh --connect qemu+ssh://root@10.20.190.3/system net-define /tmp/isolated.xml 
Network isolated defined from /tmp/isolated.xml

# 檢視目前的 virtual network 清單 (isolated 已經加入，但尚未啟動)
$ virsh --connect qemu+ssh://root@10.20.190.3/system net-list --all
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
 isolated             inactive   no            yes

# libvirt 會自動指派 UUID, virtual bridge ...等資訊
$ virsh --connect qemu+ssh://root@10.20.190.3/system net-dumpxml isolated
<network>
  <name>isolated</name>
  <uuid>765e746b-334d-4acf-8b9c-cb32be5b77f5</uuid>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:70:8b:87'/>
</network>
```

> 定義一個新的 virtual network 後，會在 KVM host 上的 `/etc/libvirt/qemu/networks` 目錄產生對應的 XML 設定檔，以上面為例，XML 設定檔完整名稱是 `/etc/libvirt/qemu/networks/isolated.xml`

完成 virtual network define 之後，接著要啟動 virtual network：

```bash
$ virsh --connect qemu+ssh://root@10.20.190.3/system net-start isolated
Network isolated started

# network isolated 已經啟動
$ virsh --connect qemu+ssh://root@10.20.190.3/system net-list --all
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes
 isolated             active     no            yes
```

完成了 isolated virtual network 的配置後，以下是動態產生 & 刪除 NIC 並與 VM 相連的相關操作：

```bash
# 檢視 domain interface list (目前只有連接 default network)
$ virsh --connect qemu+ssh://root@10.20.190.3/system domiflist centos7
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      network    default    virtio      52:54:00:7e:f8:5c

# attach-interface => 產生一個 interface 並連接到後續指定的 virtual network
# --domain => 指定要變更設定的 VM (稱為 domain)
# --source isolated --type network => 使用 virtual network isolated
# --model virtio => 使用 virtio (效能較好)
# --config => 變更設定 & 儲存 (若沒使用此參數，NIC 僅會暫時出現)
# --live => 在 VM 執行狀態下動態新增 (若要針對停止中的 VM 新增 NIC 不需要使用此參數)
$ virsh --connect qemu+ssh://root@10.20.190.3/system attach-interface --domain centos7 --source isolated --type network --model virtio --config --live
Interface attached successfully

# 重新檢視 VM 的 interface list，發現新的 NIC 已經安裝上去了
$ virsh --connect qemu+ssh://root@10.20.190.3/system domiflist centos7
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      network    default    virtio      52:54:00:7e:f8:5c
vnet1      network    isolated   virtio      52:54:00:07:b8:72

# 移除指定的 NIC
$ virsh --connect qemu+ssh://root@10.20.190.3/system detach-interface --domain centos7 --type network --mac 52:54:00:07:b8:72 --config --live
Interface detached successfully

# 可看出其中一的 NIC 已經被移除
$ virsh --connect qemu+ssh://root@10.20.190.3/system domiflist centos7
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      network    default    virtio      52:54:00:7e:f8:5c
```

當透過 virsh 新增一個 NIC 與 VM 相連後，直接到 KVM host 上使用 `brctl show` 指令檢視目前與 bridge device 相連的 interface：

```bash
$ brctl show
bridge name	bridge id		      STP enabled	interfaces
......
virbr1		  8000.525400708b87	yes		      virbr1-nic
							                            vnet1
```

可以看出 interface 已經被 libvirt 自動建立且與 bridge 相連。(其中的 **virbr1-nic** 是在 isolated virtual network 建立時，libvirt 一併同時產生的)


--------------------------------------------------------


virtual network - NATed & Routed
=================================

## NATed virtual network

安裝好 KVM 後，預設就會附上一個名稱為 `default` 的 NATed virtual network，所有使用 default virtual network 的 VM，都可以透過 KVM host 中新增的 bridge device  `virbr0`，以 NAT 的方式連到外部網路。 

NATed virtual network 的設定步驟如下：

1. 建立 bridge device，並與特定的 nic 連結

2. 設定防火牆規則 for NAT masquerade

3. virtual network 中 XML 定義的 `forward mode` 要設為 yes 

接著就是 `virsh net-{define,start,autostart}` 的工作了，這與上面雷同，就不再贅述

## Routed virtual network

這在實際應用中很少遇到，因此就略過先不測試，有機會試過之後再來補!


--------------------------------------------------------


References
==========

- [Networking - KVM]()http://www.linux-kvm.org/page/Networking

- [libvirt: Network XML format](https://libvirt.org/formatnetwork.html)

- [KVM XML 設定檔基本內容](http://www.tfcis.org/~lantw44/download/notes/cs/libvirt.domain(5))