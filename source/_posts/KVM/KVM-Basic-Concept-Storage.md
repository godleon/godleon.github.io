---
layout: post
title:  "[Linux KVM] Linux KVM concept - Storage"
description: "This article introduces what should to learn about storage when learning Linux KVM virtualization"
date: 2016-08-03 08:00:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM]
---

此篇文章介紹學習 Linux KVM 時所需要了解的 Storage 相關知識

前言
====

在 Linux KVM 中，是由 QEMU 來提供 storage device 的模擬，可以模擬的 storage 類型很多，包含 IDE device / SCSI device / Floopy disk / USB disk / virtio disk .... 等等，可根據使用者需求的不同來提供不同組合以及不同的開機順序。

----------------------------------------------------------------------------

Host Machine Swap 空間應該要設定多大?
==================================

以前學 Linux 時知道，swap 是用來作為當記憶體不足時的 buffer 用，而現在虛擬化環境中，每台 host machine 的記憶體容量都很大，那 swap 應該如何規劃配置呢? 以下有幾個原則：

### 不使用 memory overcommit

1. 若實體記憶體 <= 4GB，則 swap space 設定為 2GB

2. 若 4GB < 實體記憶體 <= 16GB，則 swap space 設定為 4GB

3. 若 16GB < 實體記憶體 <= 64GB，則 swap space 設定為 8GB

4. 若 64GB < 實體記憶體 <= 526GB，則 swap space 設定為 16GB

### 使用 memory overcommit

假設配置 memory overcommit rate 為 0.5 (例如：128 GB 的記憶體卻配置 `128 * (1 + 0.5) = 192 GB`)，那 swap space 除了上面的大小外，還要額外在加上記憶體容量 x 0.5 的空間。

亦即設定 memory overcommit，還要多增加 **<font color='red'>physical memory x memory overcommit rate</font>** 的容量大小給 swap space。

舉個實際例子，假設 host machine 有 32GB 記憶體，memory overcommit ratio 設定為 0.5，則 swap space 的容量計算如下：

> (32 * 0.5) + 8 = 24 GB

----------------------------------------------------------------------------

Storage Device & 開機順序的配置
==============================

在 **qemu-kvm** 可以直接在命令行中進行 storage device & 開機順序的配置，以下是常用的參數：

## Storage Devices

- `-hd[a:d] file_path`：以 file_path 指定的 image 檔案作為系統的 IDE HDD，在 virtual machine 中會以 **/dev/hd[a:d]** or **/dev/sd[a:d]** 呈現

> 若在 qemu-kvm 中直接指定檔案而沒有設定 -hd[a:d]，則預設為 **-hda**

- `-fd[a:b] file_path`：以 file_path 指定的 image 檔案作為系統的軟碟機，在 virtual machine 中會以 **/dev/fd[a:b]** 呈現

- `-cdrom file_path`：用來指定 iso 檔案作為 virtual machine 的 **/dev/cdrom** 時使用：不可與 **-hdc** 參數同時使用，會衝突

- `-driver option[, option[, option[, option[,....]]]]`：透過 **-driver** 參數指定 storage device 的設定細節

由於以上的設定參數太多，所以這邊直接貼 man page 的內容來看：

```bash
-drive option[,option[,option[,...]]]
    Define a new drive. Valid options are:
    file=file
      This option defines which disk image to use with this drive. If the filename contains comma, you must double it (for instance, "file=my,,file" to use file "my,file").
    if=interface
      This option defines on which type on interface the drive is connected. Available types are: ide, scsi, sd, mtd, floppy, pflash, virtio.
    bus=bus,unit=unit
      These options define where is connected the drive by defining the bus number and the unit id.
    index=index
      This option defines where is connected the drive by using an index in the list of available connectors of a given interface type.
    media=media
      This option defines the type of the media: disk or cdrom.
    cyls=c,heads=h,secs=s[,trans=t]
      These options have the same definition as they have in -hdachs.
    snapshot=snapshot
      snapshot is "on" or "off" and allows to enable snapshot for given drive (see -snapshot).
    cache=cache
      cache is "none", "writeback", "unsafe", or "writethrough" and controls how the host cache is used to access block data.
    aio=aio
      aio is "threads", or "native" and selects between pthread based disk I/O and native Linux AIO .
    format=format
      Specify which disk format will be used rather than detecting the format. Can be used to specifiy format=raw to avoid interpreting an untrusted format header.
    serial=serial
      This option specifies the serial number to assign to the device.
    addr=addr
      Specify the controller\'s PCI address (if=virtio only).
    copy-on-read=copy-on-read
      copy-on-read is "on" or "off" and enables whether to copy read backing file sectors into the image file.
```

在預設情況下，QEMU 會為所有的 block device 自動帶上 writethrough caching 的功能，這功能是利用 host page cache 所達成的，效率較差，但安全性高。

而另外一個 Writeback caching 可以提供更好的效率，但安全性較低，因為若是 host machine 突然掛點或發生電力中斷的情況，cache 中的資料很有可能會遺失；若是加上 `snapshot` 參數，則會預設使用 Writeback caching。

若是要完全關閉 cache 功能，可以加上參數 `cache=none`。

此外，其實每一個 storage device 都可以用 `-drive` 來設定，舉例來說：

- `-cdrom file_path` = `-drive file=file,index=2,media=cdrom`

- `-hda file_path` = `-drive file=file,index=0,media=disk`

- `-fda file_path` = `-drive file=file,index=0,if=floppy`

## 開機順率

開機順序的參數設定如下：

> -boot [order=drives][,once=drives][,menu=on|off][,splash=sp_name][,splash-time=sp_time][,reboot-timeout=rb_timeout][,strict=on|off]

QEMU 的開機順序，是以英文字母表示：

- `a`：第一台軟碟機

- `b`：第二台軟碟機

- `c`：第一顆硬碟

- `d`：光碟機

- `n`：網路啟動

以下是幾個簡單的範例：

- `-boot order=d`：光碟開機

- `-boot once=d`：第一次以光碟開機，系統重開後以第一顆硬碟開機

此外其他參數的說明：

- `menu=on|off`：是否顯示開機選單

- `splash=sp_name` & `splash-time=sp_time`：在 `menu=on` 時才有效，`splash` 設定開機 logo，`splash-time` 設定開機 logo 時間

- `-boot order=dc,menu=on`：開機順序為光碟機 >> 硬碟，顯示開機選單

----------------------------------------------------------------------------

qemu-img
========

**[qemu-img](http://wiki.qemu.org/download/qemu-doc.html#qemu_005fimg_005finvocation)** 是 QEMU 的磁碟管理工具，安裝好 QEMU 相關套件時就會存在於系統中，基本使用語法如下：

> qemu-img command [command options]

以下就常用的相關命令 & 參數進行介紹：

- `create [-f fmt] [-o options] filename [size]`：建立 disk image；其中 `options` 的部分可以使用 `-o ?` 來查詢特定格式所支援的選項；`size` 則是支援 `k`(KB) / `M`(MB) / `G`(GB) / `T`(TB) 等大小。

```bash
# 顯示目前 qcow2 格式支援的 options
$ qemu-img create -f qcow2 -o ?
Supported options:
size             Virtual disk size
compat           Compatibility level (0.10 or 1.1)
backing_file     File name of a base image
backing_fmt      Image format of the base image
encryption       Encrypt the image
cluster_size     qcow2 cluster size
preallocation    Preallocation mode (allowed values: off, metadata, falloc, full)
lazy_refcounts   Postpone refcount updates

# 新增檔名為 ubuntu1604.qcow2，格式為 qcow2，大小為 10GB 的 disk image
$ qemu-img create -f qcow2 ubuntu1604.qcow2 10G
Formatting 'ubuntu1604.qcow2', fmt=qcow2 size=10737418240 encryption=off cluster_size=65536 lazy_refcounts=off

# 新增有 backing file(centos7.img，原生大小為 10GB) 的 disk image
# disk image 檔名為 centos7.qcow2，格式為 qcow2，大小為 20GB
$ qemu-img create -f qcow2 centos7.img 10G
Formatting 'centos7.img', fmt=qcow2 size=10737418240 encryption=off cluster_size=65536 lazy_refcounts=off
$ qemu-img create -f qcow2 -o backing_file=centos7.img,size=20G centos7.qcow2
Formatting 'centos7.qcow2', fmt=qcow2 size=21474836480 backing_file='centos7.img' encryption=off cluster_size=65536 lazy_refcounts=off
```

- `check [-f fmt] filename`：對 disk image 進行一致性 & 錯誤的檢查，但目前只有支援 qcow2 / qed / vdi 等格式；若不加上 `[-f fmt]` 參數則會自動偵測格式

```bash
$ qemu-img check centos7.qcow2
No errors were found on the image.
Image end offset: 262144
```

- `commit [-f fmt] filename`：將指定 disk image 的修改 commit 到 backing file 中(例如上面範例中，把 centos7.qcow2 的修改 commit 回 centos7.img 中)

- `convert [-c] [-f fmt] [-O output_fmt] [-o options] filename [filename2 [...]] output_filename`：轉換 disk image 的格式；input file 的格式可自動偵測，但 output file 則需要指定，不指定則預設為 `raw`；`-c` 則是指定對 output file 進行壓縮，但只支援 qcow & qcow2 格式的檔案；`-o options` 用來指定各種選項，例如 backing file，檔案大小、是否加密...等，附帶一提，使用 backing file 可以使產生的檔案變成 copy-on-write 的增量檔案

> 若是由 raw 轉成 qcow2，一般來說檔案還會有瘦身的效果

- `info [-f fmt] filename`：顯示 disk image 的詳細資訊

```bash
$ qemu-img info /kvm/storage/vm_disks/ubnutu1604.img
image: /kvm/storage/vm_disks/ubnutu1604.img
file format: raw
virtual size: 8.0G (8589934592 bytes)
disk size: 8.0G

$  qemu-img info centos7.qcow2
image: centos7.qcow2
file format: qcow2
virtual size: 20G (21474836480 bytes)
disk size: 196K
cluster_size: 65536
backing file: centos7.img
Format specific information:
    compat: 1.1
    lazy refcounts: false
```

- `snapshot [-l | -a snapshot | -c snapshot | -d snapshot] filename`：snapshot 的管理，包含查詢 snapshot 列表資訊(`-l`) / 使用快照作為 disk image(`-a snapshot`) / 建立 snapshot(`-c snapshot`) / 刪除 snapshot(`-d snapshot`)

- `resize filename [+ | -]size`：改變 disk image 的大小；縮小空間要確保 disk image 有足夠的空間，否則會有資料損毀的風險；qcow2 格式不支援縮小；增加空間後，在 virtual machine 中需要使用 fdisk or parted 才可以使用到增加的空間

> 強烈建議在 resize 之前要做好備份

```bash
[root@ocp-kvm-host ~]# qemu-img resize ubuntu1604.qcow2 +1G
Image resized.

[root@ocp-kvm-host ~]# qemu-img info ubuntu1604.qcow2
image: ubuntu1604.qcow2
file format: qcow2
virtual size: 11G (11811160064 bytes)
disk size: 200K
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
```

----------------------------------------------------------------------------

Storage Types
=============

在 QEMU/KVM 中，virual machine 的 disk image 其實可以用很多種不同方式來儲存，例如：

1. Local storage (最常用的方式)

2. physical disk/partition

3. LVM

4. NFS

5. iSCSI

6. 在 Local 端 or 透過 Fiber 連接的 LUN

7. GFS2

> 上面的幾個選項中，phtsical partition & LVM 由於無法存放 MBR，因此無法作為 virtual machine 的開機磁碟，只能用來存放資料用。

基本上，以 file-based 的方式管理 disk images 是比較有彈性的，可放在 Local / NFS / iSCSI / LUN / GFS2 ... 等，若是 virtual machine 對於 I/O 效能不是非常要求時，選擇 qcow2 則是較為建議的 disk image 格式，其優點如下：

- 儲存方便 (file-baed)

- 比起其他方式相對容易使用

- 可移動性佳

-  容易複製

- 可透過 sparse 的機制節省硬碟空間(類似 thin provision)

- 可放在透過網路連結的檔案系統中(例如：NFS，可把 backing file 放在 NFS，差異部分放在 local)，因此可以輕鬆達到 live migration 的效果

最後，若需要高效能的 I/O，則可以使用半虛擬化的 **<font color='red'>virtio</font>** 作為 disk image 的 driver。

----------------------------------------------------------------------------

References
==========

- [qemu-kvm(1): QEMU Emulator User Documentation - Linux man page](http://linux.die.net/man/1/qemu-kvm)

- [QEMU Emulator User Documentation](http://wiki.qemu.org/download/qemu-doc.html)

- [Cache 的 write back 和 write through – Benjr.tw](http://benjr.tw/20361)

- [請教Cache運作方式的Write Back與Write Through之分別 - PCDVD數位科技討論區](http://www.pcdvd.com.tw/showthread.php?t=285217)
