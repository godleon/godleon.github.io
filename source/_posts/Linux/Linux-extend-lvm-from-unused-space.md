---
layout: post
title:  "[Linux] 利用未分割的硬碟空間，擴充 LVM 空間"
description: "本篇文章介紹硬碟若有未分割的空間時，如何利用這些空間，擴充到原有的 LVM 空間上"
date: 2019-11-15 17:10:00
published: true
comments: true
categories: [Linux]
tags: [Linux]
---


Preface
=======

因為剛剛發現監控主機空間不夠了，記得之前是以 LVM 的規劃儲存空間的，因此進行了擴充空間的操作。

實際情境：

- 現有的硬碟還有未分割的空間

- 以 LVM 的方式規劃儲存空間，目前 LV 空間已經快要耗盡


Scenario
========

- host 上只有一個 200GB disk，但**當初分割時只使用了約 60GB 的空間，還有 140GB 左右未分割空間**

- PV 已經沒有空間可用

```bash
$ pvdisplay 
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               ubuntu-vg
  PV Size               <63.00 GiB / not usable 0   
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              16127
  Free PE               3327
  Allocated PE          12800
  PV UUID               cs39RZ-Dfac-qED7-KHNz-Liht-QMtu-FvhS04
```

- VG 可分配的空間也沒很多了

```bash
$ vgdisplay 
  --- Volume group ---
  VG Name               ubuntu-vg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <63.00 GiB
  PE Size               4.00 MiB
  Total PE              16127
  Alloc PE / Size       12800 / 50.00 GiB
  Free  PE / Size       3327 / <13.00 GiB
  VG UUID               LRE3Y3-QvmP-mCI0-D0PT-Hk9w-3tqb-129LhN
```

- LV 當初只有分配了 50 GB 空間

```bash
$ lvdisplay 
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  LV UUID                xWDQsf-VimA-iX00-Kx6G-NzmJ-yaol-XVZr3L
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2019-08-05 10:16:13 +0800
  LV Status              available
  # open                 1
  LV Size                50.00 GiB
  Current LE             12800
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
```


Operations
==========

目標是將所有未分割的空間加入到 PV，讓 VG 可以有更多的空間可以分配給 LV，以下是操作步驟：

## 擴充原有 partition，使用剩餘空間

```bash
# 使用 parted 工具進行擴充
$ parted
GNU Parted 3.2
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print                                                            
Model: VMware Virtual disk (scsi)
Disk /dev/sda: 215GB    #硬碟容量一共有 215GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 
# 其中 number 3 只使用了 67.6GB
Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  ext4
 3      1076MB  68.7GB  67.6GB

# 執行 partition resize 操作                                                       
(parted) resizepart                                                       
Partition number? 3                                                       
End?  [68.7GB]? 215GB   #使用所有空間
(parted) print                                                            
Model: VMware Virtual disk (scsi)
Disk /dev/sda: 215GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 
# 此時 number 3 partition 已經使用了所有空間
Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1076MB  1074MB  ext4
 3      1076MB  215GB   214GB

# 離開 parted
(parted) quit                                                             
Information: You may need to update /etc/fstab.
```

## 擴充 PV 空間

原本上面的 number 3 partition 已經是現存的 PV，既然 partition 已經變大，接著就可以直接進行 resize 的操作：

```bash
# pv 尚未進行 resize 前 
$ pvdisplay 
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               ubuntu-vg
  PV Size               <63.00 GiB / not usable 0   
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              16127
  Free PE               3327
  Allocated PE          12800
  PV UUID               cs39RZ-Dfac-qED7-KHNz-Liht-QMtu-FvhS04

# 執行 pvresize
$ pvresize /dev/sda3
  Physical volume "/dev/sda3" changed
  1 physical volume(s) resized / 0 physical volume(s) not resized

# pv 已經確定空間變大(從 63GB -> 199GB)
$ pvdisplay 
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               ubuntu-vg
  PV Size               <199.00 GiB / not usable 16.50 KiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              50943
  Free PE               38143
  Allocated PE          12800
  PV UUID               cs39RZ-Dfac-qED7-KHNz-Liht-QMtu-FvhS04
```

## 檢查 VG 空間

從下面的執行結果可以看出，包含上面 PV 的 VG 容量也變大了：

```bash
$ vgdisplay 
  --- Volume group ---
  VG Name               ubuntu-vg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <199.00 GiB     # 變成 199GB 了
  PE Size               4.00 MiB
  Total PE              50943
  Alloc PE / Size       12800 / 50.00 GiB
  Free  PE / Size       38143 / <149.00 GiB
  VG UUID               LRE3Y3-QvmP-mCI0-D0PT-Hk9w-3tqb-129LhN
```

## 擴充 LV 空間

接著擴充 LV 的空間：

```bash
# 原本的 LV 空間狀況
$ lvdisplay 
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  LV UUID                xWDQsf-VimA-iX00-Kx6G-NzmJ-yaol-XVZr3L
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2019-08-05 10:16:13 +0800
  LV Status              available
  # open                 1
  LV Size                50.00 GiB  # 只有 50GB
  Current LE             12800
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0

 # 將 LV 空間擴充成 100GB
$ lvextend -L 100G /dev/ubuntu-vg/ubuntu-lv 
  Size of logical volume ubuntu-vg/ubuntu-lv changed from 50.00 GiB (12800 extents) to 100.00 GiB (25600 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.

# 重新檢視 LV 狀態，空間已經成功的擴充
$ lvdisplay 
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  LV UUID                xWDQsf-VimA-iX00-Kx6G-NzmJ-yaol-XVZr3L
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2019-08-05 10:16:13 +0800
  LV Status              available
  # open                 1
  LV Size                100.00 GiB     # 目前是 100GB
  Current LE             25600
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
```


References
==========

- [[RHCE7] RH134 Chapter 10. Managing Logical Volume Management(LVM) Storage 學習筆記 - 小信豬的原始部落](https://godleon.github.io/blog/RHCE/RHCE7-RH134-LearningNotes-CH10_ManagingLogicalVolumeManagement\(LVM\)Storage)

- [Linux 的 Parted 指令教學：建立、變更與修復磁碟分割區 - 頁3，共3 - G. T. Wang](https://blog.gtwang.org/linux/parted-command-to-create-resize-rescue-linux-disk-partitions/3/)


- [virtualization - How to extend a Linux PV partition online after virtual disk growth - Server Fault](https://serverfault.com/questions/378086/how-to-extend-a-linux-pv-partition-online-after-virtual-disk-growth)