---
layout: post
title:  "[Linux KVM] 半虛擬化驅動(Paravirtualization Driver)"
description: "This article introduces paravirtualization drivers(virtio) in QEMU/KVM"
date: 2016-08-20 15:30:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM]
---

介紹如何使用 QEMU/KVM 中的半虛擬化，並了解此技術如何帶來效能上的提升

QEMU I/O Overview
=================

QEMU/KVM 是屬於全虛擬化的解決方案，若沒有硬體加速輔助的情況下，所有的工作都必須透過軟體模擬，其實效率是很不好的，特別是 device I/O 的部分。

以下的圖可以說明純 QEMU 模擬下的 device I/O 狀況：

![KVM virtio](http://smilejay.b0.upaiyun.com/wp-content/uploads/2012/11/qemu-emulated-io.jpg)

一個 virtual machine 的 I/O request 會透過以下流程完成：

1. 被 KVM module 中的 I/O trap 捕捉到 & 處理

2. 將處理結果放到 I/O sharing page 中

3. 通知 QEMU process 來取得 I/O 資訊，並交由 QEMU I/O Emulation Code 來模擬 I/O request

4. 完成後將結果放回 I/O sharing page

5. 通知 KVM module 中的 I/O trap 將處理結果取回並回傳給 virtual machine

透過 QEMU 可以模擬出各式各樣的 I/O device，甚至很老舊的設備都沒有問題；但從上面複雜的步驟不難看出為何使用 QEMU 模擬 device I/O 會效率不彰，除了每次 I/O request 處理的流程繁複之外，過多的 VMEntry, VMExit, context switch，也都是拖垮 QEMU 效能的原因。

--------------------------------------------------------------------------

virtio Overview
===============

有鑑於此，[virtio](http://www.linux-kvm.org/page/Virtio) 被提出來，作為運行在 Hypervisor 上的一組 API interface，讓 virtual machine 知道自己運行在虛擬環境中，並根據 virtio 標準與 hypervisor 互動，藉此達到更好的運作效能(I/O 效能提升最為明顯)。

以下是純 QEMU 模擬 & virtio 的架構比較，可以看出 virtio 省略了 I/O trap，讓 virtual machine 可以直接與 QEMU 的 I/O 模組通訊：

![QEMU v.s. virtio](http://images0.cnblogs.com/blog2015/697113/201506/011807259737673.jpg)

更細部一點檢視 virtio 的架構：

![virtio architecture](http://smilejay.b0.upaiyun.com/wp-content/uploads/2012/11/qemu-kvm-virtio.jpg)

1. 第一層 **virtio_blk**、**virtio_net**、**virtio_scsi** .... 等等屬於 **<font color='red'>virtio Frontend，存在於</font>** virtual machine OS kernel module 中

2. 最下面一層稱為 **<font color='red'>virtio Backend</font>**，是在 QEMU 中實作，讓 I/O request 可以透過 QEMU 直接送給 host machine 中的 device driver，減少整體 I/O 的 overhead

3. **<font color='red'>virtio</font>** 屬於虛擬佇列，目的是將 Frontend 的驅動程序附加到 Backend 的處理程序 (一個 Frontend 的驅動程序可以根據需求使用 0 個或多個佇列，例如 virtio_net 同時需要傳送 & 接收用的兩個虛擬佇列，virtio_blk 則僅需要一個)

4. **<font color='red'>virtio-ring</font>** 透過實作 ring buffer 的機制，讓 I/O request 得以批次處理，藉以提升 virtual machine 與 hypervisor 之間訊息交換的效率

雖然 virtio 可以大幅的提升 I/O 性能，但不僅需要 host machine 的 OS kernel 有支援，連 virtual machine 也要同時有安裝 virtio driver，不過目前較新的 Linux 都已經支援 virtio 了! 所以若使用的是最近幾年的 Linux 版本，在 I/O 裝置的部分應該都可以選擇以 virtio 的方式運行。

> Windows 也都有對應的 virtio driver 可對應下載

--------------------------------------------------------------------------

檢查 virtio 環境
===============

## 1.1 virtio Backend

由於 host machine 安裝的是 CentOS 7，因此都已經預設安裝 virtio driver 了! 可以用以下指令查詢：

```bash
$ find / -name 'virtio*.ko' | grep $(uname -r)
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/virtio/virtio_pci.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/virtio/virtio.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/virtio/virtio_ring.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/virtio/virtio_input.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/virtio/virtio_balloon.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/char/virtio_console.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/char/hw_random/virtio-rng.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/scsi/virtio_scsi.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/block/virtio_blk.ko
/usr/lib/modules/3.10.0-327.28.2.el7.x86_64/kernel/drivers/net/virtio_net.ko
```

## 1.2 virtio Frontend

### 1.2.1 Linux

目前新版本的 Linux(Ubuntu / CentOS / .... 等等)都已經內建 virtio driver 了，不需要再額外安裝囉!

### 1.2.2 Windows

若是要安裝 Windows 的 virtio driver，可以使用以下指令：

```bash
$ wget https://fedorapeople.org/groups/virt/virtio-win/virtio-win.repo -O /etc/yum.repos.d/virtio-win.repo

$ yum info virtio-win
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: ftp.yzu.edu.tw
 * epel: mirror01.idc.hinet.net
 * extras: ftp.yzu.edu.tw
 * updates: ftp.yzu.edu.tw
Available Packages
Name        : virtio-win
Arch        : noarch
Version     : 0.1.102
Release     : 1
Size        : 75 M
Repo        : virtio-win-stable
Summary     : VirtIO para-virtualized drivers for Windows(R)
URL         : http://www.redhat.com/
License     : GPLv2
Description : VirtIO para-virtualized Windows(R) drivers for 32-bit and 64-bit
            : Windows(R) guests.

$ yum -y install virtio-win

# 查詢目前 virtio-win 的支援程度
$ ls -R /usr/share/virtio-win
/usr/share/virtio-win:
drivers  guest-agent  virtio-win-0.1.102_amd64.vfd  virtio-win-0.1.102.iso  virtio-win-0.1.102_x86.vfd  virtio-win_amd64.vfd  virtio-win.iso  virtio-win_x86.vfd

/usr/share/virtio-win/drivers:
amd64  i386

/usr/share/virtio-win/drivers/amd64:
Win2003  Win2008  Win2008R2  Win2012  Win2012R2  Win7  Win8  Win8.1
.......
/usr/share/virtio-win/drivers/amd64/Win2012R2:
netkvm.cat  netkvm.inf  netkvm.sys  vioscsi.cat  vioscsi.inf  vioscsi.sys  viostor.cat  viostor.inf  viostor.sys
.......
/usr/share/virtio-win/drivers/amd64/Win7:
netkvm.cat  netkvm.inf  netkvm.sys  qxl.cat  qxldd.dll  qxl.inf  qxl.sys  vioscsi.cat  vioscsi.inf  vioscsi.sys  viostor.cat  viostor.inf  viostor.sys
.......
```

> 從上面的檔案列表得知，目前 stable 的 virtio-win 支援到 Windows 2012R2 & Windows 8.1，若要更新的 Windows OS 需求，可能要使用 latest 版本

若希望可以在最新版的 Windows 上安裝 virtio driver，那就要升級到最新版的 virtio-win，可以透過以下的指令來啟用最新的 repository & 升級：(目前最新版已經支援 Windows 10)

> yum --enablerepo=virtio-win-latest update virtio-win

使用方式很容易，只要在啟動 windows virtual machine 時，指定 CDRom 裝置掛載 <font color='red'>**/usr/share/virtio-win/virtio-win-0.1.102.iso**</font> 後，進入 Windows 把相關的 virtio 裝置驅動即可，詳細的使用方式可以參考 => [傲笑紅塵路: 架設 Linux KVM 虛擬化主機 (Set up Linux KVM virtualization host)](http://www.lijyyh.com/2015/12/linux-kvm-set-up-linux-kvm.html)

--------------------------------------------------------------------------

使用 virtio_balloon
===================

## balloonning Overview

![KVM ballooning](http://smilejay.com/wp-content/uploads/2012/11/linux-ballooning-demo.jpg)

透過 balloon 的技術，可以如上圖所示，在 virtual machine 運行時動態調整記憶體的配置，而不需要 virtual machine 關機。

情境如下：

1. 當 host machine 記憶體空間不足時，在 virtual machine 中的記憶體 balloon 會膨脹(inflate)，讓 virtual machine 實際上無法使用到太多的記憶體，進而讓記憶體空間可以讓 host machine 暫時利用

2. 當 virtual machine 記憶體空間不足時，在 virtual machine 中的記憶體 balloon 則會壓縮(deflate)，讓 host machine 可以分配閒置的記憶體給 virtual machine

> 以上功能必須透過 `virtio-balloon driver` 來達成

## 使用 balloonning 技術的優缺點

使用 balloon 的技術肯定也是有優缺點的，管理者可以根據實際需求評估使用：

優點如下：

- 由於 balloonning 是能夠被監控 & 控制的(不同於不可控制的 KSM 技術)，因此能夠有效率的節省記憶體的實際耗用

- balloonning 對於記憶體的調度是很靈活的

- hypervisor 透過 balloonning 從 virtual machine 中取得歸還的部分記憶體空間，不一定一定要用在其他地方，可自己保留住，端看管理者想要如何管理記憶體空間

但 balloonning 同樣也是有些缺點存在的的：

- virtual machine 必須安裝 virtio_balloon driver 才可使用此功能(新版的 Linux 有內建，但 Windows 就必須要另外安裝了)

- 若透過 balloonning 從 virtual machine 中取回大量的記憶體空間，可能會提升 virtual machine 對於 swap 的 I/O 存取而導致效能降低；也有可能會造成 virtual machine 中的某些 process 運作時發生記憶體不足的況狀而失敗

- 雖然 balloonning 可被監控 & 控制，但目前尚缺乏有效的自動化機制，對於大規模佈署使用上是不方便的

- 記憶體動態調整的過於頻繁，可能會使記憶體空間配置上過於零散而不連續，如此一來記憶體的使用效率就會降低

## 在 QEMU/KVM 中使用 balloonning

首先要先檢查 host machine 是否支援 balloonning：

```bash
cat /boot/config-`uname -r` | grep VIRTIO
CONFIG_VIRTIO_BLK=m
CONFIG_SCSI_VIRTIO=m
CONFIG_VIRTIO_NET=m
CONFIG_VIRTIO_CONSOLE=m
CONFIG_HW_RANDOM_VIRTIO=m
CONFIG_VIRTIO=m
CONFIG_VIRTIO_PCI=m
CONFIG_VIRTIO_PCI_LEGACY=y
CONFIG_VIRTIO_BALLOON=m
CONFIG_VIRTIO_INPUT=m
# CONFIG_VIRTIO_MMIO is not set
```

> CONFIG_VIRTIO_BALLOON=m 表示已經支援(以 module 方式載入)

接著要在 virtual machine 中啟用 balloonning，則必須在啟動 virtual machine 的指令中加上以下參數：(預設為 `-balloon none`)

> -balloon virtio[,addr=addr]

參數中的 addr 是可用來指定 virtual machine 中 balloon device 的 PCI address

也可以使用 <font color='red'>**-device**</font> 參數來統一設定不同的 device，使用方法如下：

> -device driver[,prop[=value][,...]]

了解啟用 balloon 的參數後，可以用以下指令啟動搭載 balloonning 功能的 virtual machine：

```bash
$ kvm -vnc 0.0.0.0:1 -m 2048 /kvm/storage/vm_disks/ubnutu1604.img \
  -net nic -net tap,script=/etc/qemu-ifup \
  -device virtio-balloon-pci --daemonize
```

接著連線到 virtual machine 中，查詢 balloonning device 的狀態：

```bash
# balloon 功能已開啟
ubuntu@vm-ubuntu1604:~$ cat /boot/config-`uname -r` | grep VIRTIO
.....
CONFIG_VIRTIO_BALLOON=y
.....

# 檢視目前 pci device 狀態
ubuntu@vm-ubuntu1604:~$ lspci
00:00.0 Host bridge: Intel Corporation 440FX - 82441FX PMC [Natoma] (rev 02)
00:01.0 ISA bridge: Intel Corporation 82371SB PIIX3 ISA [Natoma/Triton II]
00:01.1 IDE interface: Intel Corporation 82371SB PIIX3 IDE [Natoma/Triton II]
00:01.3 Bridge: Intel Corporation 82371AB/EB/MB PIIX4 ACPI (rev 03)
00:02.0 VGA compatible controller: Cirrus Logic GD 5446
00:03.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 03)
00:04.0 Unclassified device [00ff]: Red Hat, Inc Virtio memory balloon

# 查詢 balloonning device 的詳細資訊
ubuntu@vm-ubuntu1604:~$ lspci -s 00:04.0 -v
00:04.0 Unclassified device [00ff]: Red Hat, Inc Virtio memory balloon
        Subsystem: Red Hat, Inc Virtio memory balloon
        Physical Slot: 4
        Flags: bus master, fast devsel, latency 0, IRQ 11
        I/O ports at c040 [size=32]
        Kernel driver in use: virtio-pci

# 檢視 virtual machine 記憶體狀態
ubuntu@vm-ubuntu1604:~$ free -m
              total        used        free      shared  buff/cache   available
Mem:           2000          40        1851           3         108        1823
Swap:          2045           0        2045
```

接著在 QEMU monitor 中透過 `balloon 512` 把 virtual machine 的記憶體限縮到僅能使用 512 MB：

![balloon](https://github.com/godleon/godleon.github.io/blob/master/_posts/images/2016/KVM-Paravirtualization-Drivers/kvm_virtio_balloon.PNG?raw=true)

以下是 virtual machine 中目前的記憶體狀況：

```bash
# 經過 balloon 縮小可用記憶體空間後，檢視記憶體狀態
ubuntu@vm-ubuntu1604:~$ free -m
              total        used        free      shared  buff/cache   available
Mem:            464          39         316           3         108         288
Swap:          2045           0        2045
```

> 可以看出 total memory 已經變少，可見 balloon 的確是起了作用

另外一個比較需要注意的是，balloon 沒辦法用於增加 virtual machine 的記憶體配置；例如，啟用 virtual machine 時配置了 2048 MB 的記憶體，即使在 QEMU monitor 中使用 `ballon 4096`，也無法讓 virtual machine 的記憶體變成 4096 MB。

--------------------------------------------------------------------------

使用 virtio_net
===============

使用 virtio network device，有提高 throughput & 降低 latency 的兩項優點，可以達到接近原生網卡的效能，因此通常是配置網路時的優先選擇。

以下指令可以查詢是否支援 virtio_net：

```bash
# 有出現 virtio 表示支援 virtio network device
$ kvm -net nic,model=?
qemu: Supported NIC models: ne2k_pci,i82551,i82557b,i82559er,rtl8139,e1000,pcnet,virtio
```

## 1、啟用支援 virtio_net 的 vitual machine

接著可以用以下指令啟動一個使用 virtio_net 的 virtual machine：

```bash
$ kvm -vnc 0.0.0.0:1 -m 2048 --daemonize \
  -drive format=raw,file=/kvm/storage/vm_disks/ubnutu1604.img \
  -device virtio-balloon-pci \
  -net nic,model=virtio -net tap
```

當 virtual machine 啟動後，登入檢查 virtio_net 的狀態：

```bash
# ubuntu 16.04 原生已經搭載 virtio driver
ubuntu@vm-ubuntu1604:~$ cat /boot/config-`uname -r` | grep VIRTIO
.....
CONFIG_VIRTIO_NET=y
.....

# 可看出目前 virtual machine 的網卡已經以 virtio 模式運作
ubuntu@vm-ubuntu1604:~$ lspci
....
00:03.0 Ethernet controller: Red Hat, Inc Virtio network device
00:04.0 Unclassified device [00ff]: Red Hat, Inc Virtio memory balloon

# virtio_net 詳細資訊
ubuntu@vm-ubuntu1604:~$ lspci -vv -s 00:03.0
00:03.0 Ethernet controller: Red Hat, Inc Virtio network device
        Subsystem: Red Hat, Inc Virtio network device
        Physical Slot: 3
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR+ FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 11
        Region 0: I/O ports at c000 [size=32]
        Region 1: Memory at febd1000 (32-bit, non-prefetchable) [size=4K]
        Expansion ROM at feb80000 [disabled] [size=256K]
        Capabilities: <access denied>
        Kernel driver in use: virtio-pci
```

## 2、進一步提升半虛擬化網卡效能

### (1) 關閉 TSO & GSO 以提升 virtio_net 的效能

為了提升 virtual machine 網卡的效能，除了使用 virtio 半虛擬化的技術外，還可以透過關閉 host machine 的 TSO & GSO 功能來更進一步提升

以下可以檢查 host machine 的網卡是否支援 TSO & GSO：

```bash
$ ethtool -k ens1f1
Features for ens1f1:
.....
tcp-segmentation-offload: on
        tx-tcp-segmentation: on
        tx-tcp-ecn-segmentation: off [fixed]
        tx-tcp6-segmentation: on
.....
generic-segmentation-offload: on
.....
```

從上面可以看出目前 TSO & GSO 的功能都是開啟的，可以透過以下方式關閉：

```bash
$ ethtool -K ens1f1 tso off
$ ethtool -K ens1f1 gso off
```

如此一來 virtio_net 的效能就可以進一步的提升。

### (2) 使用 vhost-net 後端驅動

一般來說，virtio 在 host machine 是由每個 user space 中的 QEMU 來進行後端處理的，但若是可以將 network I/O 的部分移到 kernel space 來處理，不僅可以提高 network throughput，還可以降低 latency，進而提升網路的效率。

目前比較新的 Linux kernel 中都有搭載稱為 **<font color='red'>vhost-net</font>** 的 module，可用來將 virtio_net 的後端處理移到 Linux kernel 中進行來提升 network I/O 的效率。

首先先來檢查 host machine 是否支援 **vhost-net** 的功能：

```bash
$ cat /boot/config-`uname -r` | grep VHOST
CONFIG_VHOST_NET=m
.....

$ lsmod | grep vhost
vhost_net              18152  1
....
```

從上面可以看出 host machine 有支援 vhost-net，接著就可以使用以下指令啟動支援 vhost-net 的 virtual machine：

```bash
$ kvm -vnc 0.0.0.0:1 -m 2048 --daemonize \
  -drive format=raw,file=/kvm/storage/vm_disks/ubnutu1604.img,if=virtio \
  -device virtio-balloon-pci \
  -netdev tap,id=net0,vhost=on -device virtio-net-pci,netdev=net0
```

--------------------------------------------------------------------------

使用 virtio_blk
===============

virtio_blk 提供了 virtual machine 可以透過 virtio API 進行 block device I/O 的相關驅動程式，藉以提升存取 block device 的效能。

同樣的，要使用 virtio_blk，host machine & virtual machine 都必須要同時支援才行，目前新版的 Linux 都已經預設搭載 virtio driver 了，我們用以下的指令啟動支援 virtio_blk 的 virtual machine：

```bash
$ kvm -vnc 0.0.0.0:1 -m 2048 --daemonize \
  -drive format=raw,file=/kvm/storage/vm_disks/ubnutu1604.img \
  -device virtio-balloon-pci \
  -netdev tap,id=net0,vhost=on -device virtio-net-pci,netdev=net0
```

進入到 virtual machine 之後，可以透過以下指令確認 block device 的確是以 virtio driver 所驅動：

```bash
ubuntu@vm-ubuntu1604:~$ cat /boot/config-`uname -r` | grep VIRTIO_BLK
CONFIG_VIRTIO_BLK=y

ubuntu@vm-ubuntu1604:~$ lspci | grep -i virtio
00:03.0 Unclassified device [00ff]: Red Hat, Inc Virtio memory balloon
00:04.0 Ethernet controller: Red Hat, Inc Virtio network device
00:05.0 SCSI storage controller: Red Hat, Inc Virtio block devicei

ubuntu@vm-ubuntu1604:~$ lspci -vv -s 00:05.0
00:05.0 SCSI storage controller: Red Hat, Inc Virtio block device
        Subsystem: Red Hat, Inc Virtio block device
        Physical Slot: 5
        Control: I/O+ Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR+ FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 10
        Region 0: I/O ports at c000 [size=64]
        Region 1: Memory at febd2000 (32-bit, non-prefetchable) [size=4K]
        Capabilities: <access denied>
        Kernel driver in use: virtio-pci
```

--------------------------------------------------------------------------

References
==========

- [Virtio 基本概念和設備操作 - 壹讀](https://read01.com/4aJdOL.html)

- [KVM 介绍（3）：I/O 全虚拟化和准虚拟化 \[KVM I/O QEMU Full-Virtualizaiton Para-virtualization\] - SammyLiu - 博客园](http://www.cnblogs.com/sammyliu/p/4543657.html)

- [傲笑紅塵路: 架設 Linux KVM 虛擬化主機 (Set up Linux KVM virtualization host)](http://www.lijyyh.com/2015/12/linux-kvm-set-up-linux-kvm.html)

- [Windows Virtio Drivers - FedoraProject](https://fedoraproject.org/wiki/Windows_Virtio_Drivers)

- [Virtio - KVM](http://www.linux-kvm.org/page/Virtio)

- [网卡TSO/GSO/LRO/GRO简要介绍 | Chenny的部落格](http://seitran.com/2015/04/13/01-gso-gro-lro/)

- [小蘿蔔工作室 Little Robot Studio: TSO, GSO, LRO, GRO](http://lirobo.blogspot.tw/2014/12/tso-gso-lro-gro.html)

- [KVM Network Performance : TSO and GSO - Turn it off - kris.io : virtualization & cloud](https://kris.io/2015/10/01/kvm-network-performance-tso-and-gso-turn-it-off/)
