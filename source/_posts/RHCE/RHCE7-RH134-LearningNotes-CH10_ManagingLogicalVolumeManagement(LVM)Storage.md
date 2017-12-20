---
layout: post
title:  "[RHCE7] RH134 Chapter 10. Managing Logical Volume Management(LVM) Storage 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 10. Managing Logical Volume Management(LVM) Storage 留下的內容"
date: 2016-05-12 04:45:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

10.1 Logical Volume Management Concepts
=======================================

可以從下圖很清楚看出 Physical Drivers / Partitions / PV(Physical Volume) / VG(Volume Group) / LV(Logical Volume) 的相互關係

![LVM](http://mo.morsi.org/blog/files/lvm1.png)

此外，還有 PE(Physical Extent) & LE(Logical Extent) 與上述其他概念的組成關係：

![PE_LE_1](https://camo.githubusercontent.com/d8809c16fe3e3f5532e87165dfe4012346b3b13e/687474703a2f2f73332e353163746f2e636f6d2f7779667330322f4d30302f37322f33352f774b696f4c31586574513777746d7a6641414a364d4351464339303134332e6a7067)

![PE_LE_2](http://www.linux-tips-and-tricks.de/images/stories/lvm.jpg)

----------------------------------------------------------------

10.2 Managing Logical Volumes
=============================

## 10.2.1 Create PV

1. 在 /dev/sda 中建立一個 partition，使用所有空間，partition type = 8e

2. 第一個硬碟建立 PV：`sudo pvcreate /dev/sda1`

3. 第二個硬碟建立 PV：`sudo pvcreate /dev/sdb`(沒有 partition 的狀況下)

### 其他相關指令

- `pvscan`：目前有哪些 PV

- `pvdisplay [pv_name]`：顯示 PV 的詳細資訊


## 10.2.2 Create VG

1. 建立 Volume Group：`sudo vgcreate -s 16M vg0 /dev/sda1 /dev/sdb` (**<font color='red'>設定 PE 大小為 16M</font>**)
> 相同 VG 中的 PE 都一樣大

2. 檢查 PV：`sudo pvscan` (顯示 PV 被使用中)

3. 檢查 VG：`sudo vgscan`

4. 檢視 VG 詳細資料：`sudo vgdisplay [vg_name]`

## 10.2.3 Create LV

1. 建立 LV(從 vg0 切割 200M 空間，給 LV 使用，並命名為 lv-1)：`sudo lvcreate -L 200M -n lv-1 vg0`

2. 建立 LV(使用 PE 個數指定大小)：`sudo lvcreate -l 13 -n lv-2 vg0`

3. 檢查 LV：`sudo lvscan`

4. 檢視 LV 詳細資料：`sudo lvdisplay /dev/vg0/lv-1` & `sudo lvdisplay /dev/vg0/lv-2`

> 要使用 LV 前要記得進行 format 的動作

----------------------------------------------------------------

Practice: Adding a Logical Volume
=================================

## 目標

1. 建立一個 volume group，名稱為 `shazam`，由兩個大小為 `256 MB` 的 physical partition 組成(來源為 `/dev/vdb`)

2. 從 volume group 中建立一個 `400 MB` 的 logical volume，名稱為 `storage`

3. 將 logical volume 掛載在 `/storage` 目錄

## 實作

### 1、建立一個 volume group，名稱為 `shazam`，由兩個大小為 `256 MB` 的 physical partition 組成(來源為 `/dev/vdb`)

1.1 分割硬碟：

```bash
$ sudo fdisk /dev/vdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0xa2e37afd.

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p):
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519): +256M
Partition 1 of type Linux and of size 256 MiB is set

Command (m for help): n
Partition type:
   p   primary (1 primary, 0 extended, 3 free)
   e   extended
Select (default p):
Using default response p
Partition number (2-4, default 2):
First sector (526336-20971519, default 526336):
Using default value 526336
Last sector, +sectors or +size{K,M,G} (526336-20971519, default 20971519): +256M
Partition 2 of type Linux and of size 256 MiB is set

Command (m for help): t
Partition number (1,2, default 2): 1
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): t
Partition number (1,2, default 2): 2
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

$ sudo partprobe -s
/dev/vda: msdos partitions 1
/dev/vdb: msdos partitions 1 2
```

1.2 建立 physical volume

```bash
$ sudo pvcreate /dev/vdb1 /dev/vdb2
  Physical volume "/dev/vdb1" successfully created
  Physical volume "/dev/vdb2" successfully created
```

1.3 建立 volume group

```bash
$ sudo vgcreate shazam /dev/vdb1 /dev/vdb2
  Volume group "shazam" successfully created
```

### 2、從 volume group 中建立一個 `400 MB` 的 logical volume，名稱為 `storage`

```bash
$ sudo lvcreate -n storage -L 400M shazam
  Logical volume "storage" created

# 格式化 LV
$ sudo mkfs.ext4 /dev/shazam/storage
  mke2fs 1.42.9 (28-Dec-2013)
  Filesystem label=
  OS type: Linux
  Block size=1024 (log=0)
  Fragment size=1024 (log=0)
  Stride=0 blocks, Stripe width=0 blocks
  102400 inodes, 409600 blocks
  20480 blocks (5.00%) reserved for the super user
  First data block=1
  Maximum filesystem blocks=34078720
  50 block groups
  8192 blocks per group, 8192 fragments per group
  2048 inodes per group
  Superblock backups stored on blocks:
          8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

  Allocating group tables: done
  Writing inode tables: done
  Creating journal (8192 blocks): done
  Writing superblocks and filesystem accounting information: done
```

### 3、將 logical volume 掛載在 `/storage` 目錄

```bash
$ sudo blkid
/dev/vda1: UUID="9bf6b9f7-92ad-441b-848e-0257cbb883d1" TYPE="xfs"
/dev/vdb1: UUID="MxwktT-fxCb-KjzV-muCq-Dk7z-IHZU-WdclEG" TYPE="LVM2_member"
/dev/vdb2: UUID="mQaiiR-3LEA-XhT4-cd36-gT4T-7jjR-HrQreq" TYPE="LVM2_member"
/dev/mapper/shazam-storage: UUID="4b5e590a-10af-4e44-8619-aba91e436d17" TYPE="ext4"

$ sudo mkdir /storage
$ echo "UUID=4b5e590a-10af-4e44-8619-aba91e436d17 /storage ext4 defaults 0 2" | sudo tee --append /etc/fstab
$ sudo mount -a
$ sudo mount | grep storage
/dev/mapper/shazam-storage on /storage type ext4 (rw,relatime,seclabel,data=ordered)
```

----------------------------------------------------------------

10.3 Extending Logical Volumes
==============================

## 10.3.1 Extending and reducing a volume group

1. 建立新的 PV：`sudo pvcreate /dev/sdc`

2. 將新的 PV(/dev/sdc) 加到 VG 中：`sudo vgextend vg0 /dev/sdc`

> 若要把 PV 移出 VG 可以使用類似的指令：`sudo vgreduce vg0 /dev/sda1`

## 10.3.2 Extend a logical volume and XFS file system

目標：**將 LV 容量加大到 300M**

1. LV 放大可以 online，縮小需要 offline：`sudo lvextend -L +100M /dev/vg0/lv-1` or `sudo lvextend -L 300M /dev/vg0/lv-1`

2. 也可以用指定 PE 的個數來放大：`sudo lvextend -l +7 /dev/vg0/lv-2` or `sudo lvextend -l 19 /dev/vg0/lv-2`

3. 檢查 LV 狀態：`sudo lvscan`

4. 放大 XFS 檔案系統：`sudo xfs_growfs /dev/vg0/lv-1`

5. 放大 EXT4 檔案系統：`sudo resize2fs /dev/vg0/lv-2`

### 如何縮小 LV

**<font color='red'>【註】</font>** XFS 檔案系統僅能放大，不能縮小(上面的 lv-1 已經無法縮小)

目標：縮小 EXT4 到 200M

1. umount EXT4 partition：`sudo umount /lv-2`

2. 檢查 LV：`sudo fsck -f /dev/vg0/lv-2`

3. 縮小 EXT4 檔案系統：`sudo resize2fs /dev/vg0/lv-2 200M`

4. 縮小 LV：`sudo lvreduce -L 200M /dev/vg0/lv-2`

5. 重新掛載磁區：`sudo mount -a`

### 如何縮小 VG & 移除 PV(硬碟)

目標：移除第一個 PV(`/dev/sda1`)

1. 移動指定 PV 中的檔案到其他 PV 上：`pvscan` -> `sudo pvmove /dev/sda1 /dev/sdc`(也可以不指定目的裝置)

2. 移除 VG 中的 PV：`sudo vgreduce vg0 /dev/sda1` -> `pvscan`

3. 徹底移除 PV：`sudo pvremove /dev/sda1`

最後就可以把電腦關機並移除硬碟 /dev/sda 囉!


### 移除 LV

目標：**移除 /dev/vg0/lv-1**

1. 卸載檔案系統：`sudo umount /lv-1`
2. 移除 LV：`sudo lvremove /dev/vg0/lv-1` -> `sudo lvscan`

----------------------------------------------------------------

Practice: Extending a Logical Volume
====================================

## 目標

1. 以上一個練習為基礎，增加一個容量為 `800MB` 的 physical volume

2. 把原有的 logical volume(`storage`)大小增加為 1GB

### 1、以上一個練習為基礎，增加一個容量為 `800MB` 的 physical volume

1.1 分割硬碟

```bash
$ sudo fdisk /dev/vdb
[sudo] password for student:
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p):
Using default response p
Partition number (3,4, default 3):
First sector (1050624-20971519, default 1050624):
Using default value 1050624
Last sector, +sectors or +size{K,M,G} (1050624-20971519, default 20971519): +800M
Partition 3 of type Linux and of size 800 MiB is set

Command (m for help): t
Partition number (1-3, default 3):
Hex code (type L to list all codes): 8e
Changed type of partition 'Linux' to 'Linux LVM'

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.

WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at
the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.

$ sudo partprobe -s
/dev/vda: msdos partitions 1
/dev/vdb: msdos partitions 1 2 3
```

1.2 加入 physical volume

```bash
$ sudo pvcreate /dev/vdb3
  Physical volume "/dev/vdb3" successfully created
```

1.3 擴充 volume group 空間

```bash
$ sudo vgextend shazam /dev/vdb3
  Volume group "shazam" successfully extended
$ sudo vgdisplay shazam
  --- Volume group ---
  VG Name               shazam
  System ID
  Format                lvm2
  Metadata Areas        3
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                3
  Act PV                3
  VG Size               1.27 GiB
  PE Size               4.00 MiB
  Total PE              325
  Alloc PE / Size       100 / 400.00 MiB
  Free  PE / Size       225 / 900.00 MiB
  VG UUID               s4g8vT-JCW9-p84f-wVU1-0dsn-xkrh-vytTVc
```

### 2、把原有的 logical volume(`storage`)大小增加為 1GB

```bash
$ sudo lvextend -L 1G /dev/shazam/storage
  Extending logical volume storage to 1.00 GiB
  Logical volume storage successfully resized

$ sudo resize2fs /dev/shazam/storage
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/shazam/storage is mounted on /storage; on-line resizing required
old_desc_blocks = 4, new_desc_blocks = 8
The filesystem on /dev/shazam/storage is now 1048576 blocks long.

$ df -hT | grep storage
/dev/mapper/shazam-storage ext4      984M  2.8M  932M   1% /storage
```

----------------------------------------------------------------

10.4 Snapshot (補充)
====================

假設：有個 20T 的 MySQL 資料庫，需要備份

1. 停掉 MySQL 服務

2. 建立 Snapshop：`sudo lvcreate -L 100M -s -n backup /dev/vg0/lv-2` -> 'sudo lvscan'

3. 啟動 MySQL 服務

4. 檢查 LV 之間的關聯：`sudo lvs`

> 只有在 PE 中的資料異動前，才會有資料複製的動作發生

## 老師設計的實驗

1. reset server

2. 幫 server 加上 200M/300M/500M IDE 硬碟共三顆

3. 開機，按 F12 選擇 virtio(4) 開機
