---
layout: post
title:  "[Linux KVM] Linux KVM concept - Memory"
description: "This article introduces what should to learn about memory when learning Linux KVM virtualization"
date: 2016-08-02 03:40:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM]
---

此篇文章介紹學習 Linux KVM 時所需要了解的 Memory 相關知識

前言
====

Memory 在作業系統是用來暫時存放 cpu 要執行的指令以及資料，所有 process 都必須先載入到 memory 中才能正確執行。

而在虛擬化的環境中，virtual machine 中 memory 的使用是需要額外的 mapping 機制對應到 host machine，在這個部分的效率的高低，當然也就會決定 virtual machine 整體的系統性能。

---------------------------------------------------------------------

VM 可以使用多少記憶體?
===================

也許這是很多人想要了解的，在 host machine 上安裝了一大堆實體記憶體，究竟可以分配給 VM 的可以有多少呢? 以下有兩個簡單公式可以計算：

1. 實體記憶體 <= 64 GB
> RAM - 2 GB = Amount of RAM available to VMs in GBs

2. 實體記憶體 > 64GB
> RAM - (2 GiB + .5* (RAM/64)) = Amount of RAM available to VMs in GBs

假設 host machine 記憶體有 32GB，則一共可以配置 `32 - 2 = 10`GB 的記憶體給 VM。

假設 host machine 記憶體有 256GB，則一共可以配置 `256 - (2 + 0.5 * (256 / 64)) = 252`GB 的記憶體給 VM

---------------------------------------------------------------------

配置 VM 記憶體
==============

幫 virtual machine 配置記憶體是很容易的，只要使用 **<font color='red'>-m</font>** 參數即可，預設的格式是：

> -m megs

預設是以 **<font color='red'>MB</font>** 作為預設的單位，也可以使用 **<font color='red'>G</font>** 表示要使用 GB 為單位，例如：

```bash
# 記憶體大小為 2048 MB
$ kvm -vnc 0.0.0.0:1 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img

# 記憶體大小為 4 GB
$ kvm -vnc 0.0.0.0:1 -m 4G -hda /kvm/storage/vm_disks/ubnutu1604.img
```

---------------------------------------------------------------------

EPT
===

傳統要把 virtual machine 中運行的應用程式所使用的 memory 對應到 host machine 的 memory，一共是有三層關係的，下圖可做個簡單說明：

![VM memory mapping](http://m.eet.com/media/1200411/CloudFig4a.jpg)

> Guest Virtual Address(GVA) <--> Guest Physical Address(GPA) <--> Host Physical Address(HPA)

但由於上面三層的轉換效率是很差的，因此後來透過軟體實作了稱為 **<font color='red'>Shadow Page Tables</font>** 的機制，將三層中的第二層拿掉，直接讓 Guest Virtual Address 與 Host Physical Address 可以有直接對應的機制：

![Shadow Page Tables](http://image.slidesharecdn.com/windays-virtualization-090428034228-phpapp01/95/windows-server-virtualization-hyperv-2008-r2-30-728.jpg?cb=1240890219)

此時 hypervisor 就可以把 shadow page tables 載入到 MMU(Memory Management Unit) 中進行 address translation 的工作。

但 shadow page tables 實作起來不僅複雜，也會 memory 的額外消耗(每一個 virtual machine 都需要一個 shadow page table)；因此 Intel 提出了 EPT(Extended Page Tables)，AMD 提出了 NPT(Nested Page Tables)，在硬體層直接提供了 `GVA <--> GPA <--> HPA` 的轉換，不僅提升了 memory 虛擬化的效能，也降低了 memory 虛擬化的複雜度。

以下是 Intel EPT 技術的概觀：

![Intel EPT](http://virtualization.info/images/EPT-716833.png)

其中可以看到 Intel 在硬體中增加了 CR3(Control Registor 3) 來處理 GVA <--> GPA 的轉換，以及 EPT 來處理 GPA <--> HPA 的轉換。

由於所有的轉換都在硬體層級完成，因此速度很快；而且由於整個轉換過程只需要一個 EPT Page Table，因此在 memory 的消耗上也相對的低。

----------------------------------------------------------------------

VPID
====

要了解 VPID，首先要知道 TLB(Translation Lookaside Buffer) 是什麼? 可以參考以下連結：

- [TLB 轉譯後備緩衝區 - Wikiwand](http://www.wikiwand.com/zh-hk/%E8%BD%89%E8%AD%AF%E5%BE%8C%E5%82%99%E7%B7%A9%E8%A1%9D%E5%8D%80)

> 分配給 virtual machine 的每一個 vCPU 都會有一個 TLB

而 VPID 則是在硬體層級對 TLB 資源管理的優化，在 virtual machine 進行 migration / VM Entry / VM Exit 時，避免對 TLB 進行轉存 & 清除，進而降低 memory 的額外消耗，對於 live migration 有顯著的效能提升。

----------------------------------------------------------------------

查詢 EPT & VPID 的支援度
=======================

```bash
# 查詢 CPU 是否支援 EPT & VPID
$  grep -E "\sept\s|\svpid\s" /proc/cpuinfo | uniq

# 查詢 kvm_intel 模組是否有開啟 EPT & VPID 的功能
$ cat /sys/module/kvm_intel/parameters/ept
Y
$ cat /sys/module/kvm_intel/parameters/vpid
Y

# 開啟 kvm_intel 模組 EPT & VPID 的功能
$ modprobe kvm_intel ept=1,vpid=1
```

----------------------------------------------------------------------

Huge Page
=========

x86 架構的 CPU 預設的 memory page table 大小為 4KB，而 x86-64 則可以支援到 2MB 大小的 memory page table(亦稱為 **<font color='red'>Huge Page</font>**)，在 Linux 2.6 以上的 kernel 都支援這個特性。

而使用 Huge Page 有何優缺點呢?

### 優點：

- page table 數量減少，更節省 memory

- 由於 memory address translation 的工作減少了，因此 page fault 機率降低為 **<font color='red'>Huge Page Size / 4KB</font>** 分之一

- 提升了 memory 存取的效能

- 提高 TLB 命中率，因而減少 CPU cache 的使用，最後提升了系統整體效能

- 適合用在 memory 存取密集的 virtual machine 上

### 缺點：

- Huge Page 無法被 swap out 到硬碟上

- 無法使用 Ballooning 的方式自動增長

- 並非適合所有不同工作類型的 virtual machine

## 在 KVM 中使用 Huge Page

### 1、檢查 host machine 中 Huge Page 的設定資訊：

```bash
# 目前預設的 page size
$ getconf PAGESIZE
4096

# 記憶體資訊中 Huge Page 的相關資訊 (size = 2048 KB)
$ cat /proc/meminfo | grep Huge
AnonHugePages:     14336 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

### 2、掛載 hugetlbfs 檔案系統

```bash
# 掛載 hugetlbfs 檔案系統
$ mount -t hugetlbfs hugetlbfs /dev/hugepages

# 查詢檔案系統資訊
$ mount | grep huge
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,seclabel)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,seclabel)
```

### 3、設定 Huge Page 的數量

假設要啟動一個 memory 為 2048 MB 的 virtual machine，可以算出 Huge Page(2048 KB) 的數量為：

> 2048 * 1024 / 2048 = 1024

因此這邊設定 Huge Page 的數量為 1024 個：

```bash
# 設定 Huge Page 的數量為 1024 個
$ sysctl vm.nr_hugepages=1024
vm.nr_hugepages = 1024

# 此時 host machine 中的記憶體資訊已經有 Huge Page 的數量資訊
$ cat /proc/meminfo | grep Huge
AnonHugePages:     14336 kB
HugePages_Total:    1024
HugePages_Free:     1024
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

### 4、啟動 virtual machine 使用 Huge Page

```bash
$ kvm -vnc 0.0.0.0:1 -smp 4 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img -mem-path /dev/hugepages

$ cat /proc/meminfo | grep Huge
AnonHugePages:     26624 kB
HugePages_Total:    1024
HugePages_Free:      738
HugePages_Rsvd:      738
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

可以看出 virtual machine 的確消耗了一些 huge page，但其實並不是全部，因為 virtual machine 實際上並沒有完整分配到 2048 MB 的 memory；若要完整分配指定的 memory，則需要加上 `-mem-prealloc` 參數。

----------------------------------------------------------------------

Memory Overcommit
=================

除了之前介紹 CPU 可以 overcommit 之外，memory 也可以設定一定程度的 overcommit，原因是因為每台電腦在運作時一般都不會耗盡記憶體。對 host machine 來說，virtual machine 也只是一個 QEMU process，在啟動的當下是不會分配完整記憶體的，而是隨著 virtual machine 的更多要求下逐步分配到位，因此可以在此行為的前提下設定 memory overcommit。

在 KVM 中有三種方式可以達到 memory overcommit：

1. **<font color='red'>Swapping</font>**：透過 system swap(一般為硬碟空間) 來彌補 memory 不足的問題

2. **<font color='red'>Ballooning</font>**：透過 `virtio_balloon` driver 來達成

3. **<font color='red'>Page Sharing</font>**：使用 `KSM(Kernel Samepage Merging)` 合併多台 virtual machine 中相同的 memory page

關於第一個方式的 swap，根據 [RedHat RHEL 7 官方文件](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Storage_Administration_Guide/ch-swapspace.html#tb-recommended-system-swap-space)所提供的建議，swap 大小的設定建議如下：

| 系統記憶體 | 建議 swap 大小 |
|-----------|----------------|
| ⩽ 2 GB | 2 x memory size |
| > 2 GB – 8 GB   | 等同 memory size |
| > 8 GB – 64 GB  | 至少 4 GB |
| > 64 GB  | 至少 4 GB |


----------------------------------------------------------------------

References
==========

- [qemu-kvm(1): QEMU Emulator User Documentation - Linux man page](http://linux.die.net/man/1/qemu-kvm)

- [精品：KVM學習筆記 : 歌穀穀](http://www.gegugu.com/2016/03/22/9088.html)

- [Huge Page 是否是拯救性能的萬能良藥？ - 王朝網路 - wangchao.net.cn](http://tc.wangchao.net.cn/it/detail_128378.html)

- [隨意寫寫: Compound Page, Huge Page, 和Transparent Huge Page(THP)](http://brandon-hy-lin.blogspot.tw/2016/04/compound-page-huge-page-transparent.html)

- [IT 研究室 ( 前IT DBA's 資訊站): 如何在Linux 使用大記憶體(huge memory greater then hundred of GB)](http://jaychu649.blogspot.tw/2014/09/linux-huge-memory-greater-then-hundred.html)

- [RedHat RHEL 7 Support > Product Documentation > Storage > Administration Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Storage_Administration_Guide/ch-swapspace.html#tb-recommended-system-swap-space)
