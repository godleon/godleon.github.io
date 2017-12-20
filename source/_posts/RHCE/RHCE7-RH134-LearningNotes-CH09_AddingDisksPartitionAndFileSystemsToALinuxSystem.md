---
layout: post
title:  "[RHCE7] RH134 Chapter 09. Adding Disks, Partitions, and File Systems to a Linux System 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 09. Adding Disks, Partitions, and File Systems to a Linux System 留下的內容"
date: 2016-05-10 04:45:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---


9.1 Adding Partitions, File Systems, and Persistent Mounts
===========================================================

## 9.1.1 Disk partitioning

Hard Disk 的分割結構：(從頭到尾)

1. MBR(512 bytes) = Boot Loader(446 byte) + Partition Table(64 bytes)

Partition Type：

- `Primary`：一個硬碟最多可以切 4 個 primary partition (系統要啟動必須裝載 Primary Partition)

- `Extended`：若要超過 4 個 partition，則要將 primary partition 來換成 extended partition

- `Logical`：在 extended partition 中的 partition (Logical Partition Table 存在 Extended Partition 的前面)，且**<font color='red'>代碼從 5 開始(/dev/sda5)</font>**

備份 /dev/sda1：`dd if=/dev/sda1 of=/tmp/sda1.dd`

還原 /dev/sda1：`dd if=/tmp/sda1.dd of=/dev/sda1`

### MBR Partition

- 支援 2^32 個定址空間

- 傳統硬碟最大容量為 2T：2^32 x 512 bytes / 1024 / 1024 /1024 / 1024 = 2T

- 超過 2T 的硬碟 -> Advanced Format(block size 從 512 bytes 變成 4K) = 最大可支援到 16T (過渡時期的解決方案)

- 備份 MBR：`dd if=/dev/sda of=/tmp/mbr.dd bs=512 count=1`

### GPT Partition

- 若 OS 裝在 GPT Partition 上，沒辦法在舊電腦上用(因為傳統的 int 13h 問題)；但開機後，OS level 可以認得超過 2T 的硬碟

- UEFI BIOS 上的 int 13h 改寫過，因此同時支援 MBR & GPT 兩種 partition

- 最多支援 128 個 partition，支援 2^64 個定址空間，因此最大可以支援到 8ZB(512 bytes block size)，若是使用 4K block size 則可支援到 64ZB

- GPT 為每一個 partition 提供 128 bits GUID 作為辨識之用

- GPT partition table 在硬碟的頭尾各有一個，並搭配 CRC checksum，因此有備援的機制


## 9.1.2 Managing MBR partitions with fdisk/gdisk

- 若沒有使用 `w` 儲存 partition 分割設定，就不會實際切割硬碟

```bash
# 查詢目前的 partition table
[vagrant@localhost ~]$ cat /proc/partitions
major minor  #blocks  name

   8        0   83886080 sda
   8        1     512000 sda1
   8        2   83373056 sda2
 253        0   52428800 dm-0
 253        1    1048576 dm-1
 253        2   29827072 dm-2
```

切割完 partition 後，記得執行 `sudo partprobe [device name]` or `sudo partprobe -s` 強制 kernel 重新讀取最新的 partition table

## 9.1.4 Creating file systems

硬碟分割完後需要進行格式化，一般會使用 `mkfs` 指令：

`sudo mkfs -t xfs /dev/vdb1`：格式化成 XFS，也可以寫成 `sudo mkfs.xfs /dev/vdb1`

## 9.1.4 Mounting file systems

```bash
$ sudo mkdir /XFS

$ sudo mkdir /EXT4

# 掛載 XFS partition
$ sudo mount /dev/vdb1 /XFS

# 掛載 EXT4 partition
$ sudo mount /dev/vdb2 /EXT4-t

# 顯示 partition type(-T) & human readable(-h)
$ df -T -h

# 查詢目前在 mount point 的 user 有哪些
$ sudo fuser -v /XFS
                     USER        PID ACCESS COMMAND
/mnt:                root     kernel mount /XFS
                     student    1663 ..c.. bash
                     root       1936 ..c.. sudo

# 踢掉目前在指定 mount point 的 user
$ sudo fuser -km /NFS
```

### /etc/fstab

若要讓 partition 開機後就固定掛載在指定目錄，就要使用 **/etc/fstab** 來達成：

```bash
[vagrant@server ~]$ cat /etc/fstab | grep '^[^#]'
# 1. partition = /dev/mapper/centos-root (也可以用 partition UUID)
# 2. mount point = /
# 3. partition type = xfs
# 4. mount options = defaults (rw,suid,dev,exec,auto,nouser,async)
# 5. 指定是否需要 dump = 0 (不需 dump)
# 6. 系統開機時進行 fsck 檢查的順序
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=cb6b4419-b7cc-4315-b5bd-5926d21e944e   /boot    xfs     defaults        0 0
/dev/mapper/centos-home /home                   xfs     defaults        0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0

/dev/vdb1   /XFS    xfs   defaults    1   2
/dev/vdb2   /EXT4   ext4  defaults    1   2
```

編輯完 `/etc/fstab` 後，必須執行 `sudo mount -a` 檢查是否 partition 可以正確掛載

> 建議 partition 透過 UUID 指定，可使用 `blkid [partition_name]` 來取得 partition UUID 的資訊(但似乎只對已經 mount 到特定目錄的 partition 有效)

也可以使用 LABEL 的方式設定在 /etc/fstab 中：

```bash
# 為 XFS partition 命名
$ sudo xfs_admin -L XFS /dev/vdb1
# 為 EXT4 partition 命名
$ sudo e2label /dev/vdb2 EXT4

# 查詢結果
$ sudo blkid

$ sudo mount -L XFS /XFS
$ sudo mount -L EXT4 /EXT4

# /etc/fstab 的設定可以改成如下
LABEL=XFS   /XFS    xfs   defaults    1   2
LABEL=EXT4  /EXT4   ext4  defaults    1   2
```

> Label 還是有可能重複，因此原廠建議用上面的 UUID 掛載，是最保險沒問題的

--------------------------------------------------------------------

Practice: Adding Partitions, File Systems, and Persistent Mounts
================================================================

## 目標：從 `/dev/vdb` 中切割 1GB 的空間，並掛載在 `/archive` 目錄

### 1、切割 1GB partition

```bash
# 從 /dev/vdb 中切割出 1GB 空間
$ sudo fdisk /dev/vdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x1825a331.

Command (m for help): bn
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519): +1G
Partition 1 of type Linux and of size 1 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

# 強制 kernel 重新讀取 partition Table
$ sudo partprobe -s
/dev/vda: msdos partitions 1
/dev/vdb: msdos partitions 1

$ cat /proc/partitions
major minor  #blocks  name

 253        0   10485760 vda
 253        1   10484142 vda1
 253       16   10485760 vdb
 253       17    1048576 vdb1
```

### 2、格式化 partition

```bash
# 格式化 partition
$ sudo mkfs.ext4 /dev/vdb1
$ sudo blkid /dev/vdb1
/dev/vdb1: UUID="200ad7d4-aeef-4ffc-ac27-25fbcef5d5b2" TYPE="ext4"
```

### 3、建立目錄並掛載

```bash
# 建立掛載目錄
$ sudo mkdir /archive

# 編輯 /etc/fstab 以達成永久掛載的目的
$ echo "UUID=200ad7d4-aeef-4ffc-ac27-25fbcef5d5b2 /archive ext4 defaults 0 2" | sudo tee --append /etc/fstab
$ sudo mount -a
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
......
/dev/vdb1       976M  2.6M  907M   1% /archive
$ mount
.....
/dev/vdb1 on /archive type ext4 (rw,relatime,seclabel,data=ordered)
```

--------------------------------------------------------------------

9.2 Managing Swap Space
=======================

以下以切割 /dev/vdb 為例：

1. 使用 `fdisk`(82) or `gdisk`(8200) 進行硬碟分割

2. 將 partition 格式化為 swap：`sudo mkswap /dev/vdb1`

3. 掛載 swap partition：`sudo swapon /dev/vdb1`

若是要在 **/etc/fstab** 中掛載 fstab，在 dump & fsck 的欄位都要給 0，標準設定方式如下：

`UUID=e02aae2f-70a9-4b1b-a1e1-9016b0acc3e4  swap  swap  defaults  0 0`

### 其他 swap 相關指令

- `sudo swapon -a`：檢查 `/etc/fstab` 中的 swap partition 設定並掛載

- `sudo swapon -s`：檢視目前系統中 swap partition 的資訊以及使用優先權

--------------------------------------------------------------------

Practice: Adding and Enabling Swap Space
========================================

## 目標：在第二個硬碟中切割出 500 MB 的空間作為 swap

### 1、切割 500MB partition

```bash
$ sudo fdisk /dev/vdb
Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x7db6d48a.

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1):
First sector (2048-20971519, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-20971519, default 20971519): +500MB
Partition 1 of type Linux and of size 500 MiB is set

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): 82
Changed type of partition 'Linux' to 'Linux swap / Solaris'

Command (m for help): p

Disk /dev/vdb: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x7db6d48a

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048     1026047      512000   82  Linux swap / Solaris

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

### 2、格式化 partition

```bash
$ sudo mkswap /dev/vdb1
Setting up swapspace version 1, size = 511996 KiB
no label, UUID=ffb162b9-c1e6-4ce9-bde7-964b3ce9c43a
```

### 3、編輯 /etc/fstab 並掛載

```bash
$ sudo mkswap /dev/vdb1
Setting up swapspace version 1, size = 511996 KiB
no label, UUID=ffb162b9-c1e6-4ce9-bde7-964b3ce9c43a

$ sudo blkid /dev/vdb1
/dev/vdb1: UUID="ffb162b9-c1e6-4ce9-bde7-964b3ce9c43a" TYPE="swap"

[student@desktop0 ~]$ echo "UUID=ffb162b9-c1e6-4ce9-bde7-964b3ce9c43a swap swap defaults 0 0" | sudo tee --append /etc/fstab
UUID=ffb162b9-c1e6-4ce9-bde7-964b3ce9c43a swap swap defaults 0 0

$ sudo mount -a
$ sudo swapon /dev/vdb1
$ sudo swapon -s
Filename                                Type            Size    Used    Priority
/dev/vdb1                               partition       511996  0       -1
```

--------------------------------------------------------------------

Lab: Adding Disks, Partitions, and File Systems to a Linux System
=================================================================

## 目標：

1. 在第二個硬碟中新增一個 2GB 的 XFS partition，並永久掛載於 /backup 目錄

2. 在第二個硬碟中新增 512MB swap，擁有預設的 priority

3. 在第二個硬碟中新增 512MB swap，priority 為 1

### 1、建立三個 partition

```bash
$ sudo gdisk /dev/vdb
Command (? for help): n
Partition number (1-128, default 1):
First sector (34-20971486, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-20971486, default = 20971486) or {+-}size{KMGTP}: +2G
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300):
Changed type of partition to 'Linux filesystem'

Command (? for help): n
Partition number (2-128, default 2):
First sector (34-20971486, default = 4196352) or {+-}size{KMGTP}:
Last sector (4196352-20971486, default = 20971486) or {+-}size{KMGTP}: +512M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 8200
Changed type of partition to 'Linux swap'

Command (? for help): n
Partition number (3-128, default 3):
First sector (34-20971486, default = 5244928) or {+-}size{KMGTP}:
Last sector (5244928-20971486, default = 20971486) or {+-}size{KMGTP}: +512M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 8200
Changed type of partition to 'Linux swap'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/vdb.
The operation has completed successfully.
```

### 2、格式化硬碟

```bash
$ sudo partprobe -s
/dev/vda: msdos partitions 1
/dev/vdb: gpt partitions 1 2 3

$ cat /proc/partitions
major minor  #blocks  name

 253        0   10485760 vda
 253        1   10484142 vda1
 253       16   10485760 vdb
 253       17    2097152 vdb1
 253       18     524288 vdb2
 253       19     524288 vdb3

 $ sudo mkfs.xfs /dev/vdb1
meta-data=/dev/vdb1              isize=256    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

$ sudo mkswap /dev/vdb2
Setting up swapspace version 1, size = 524284 KiB
no label, UUID=a875704a-d0ba-4542-98a6-f201f1b3a539

$ sudo mkswap /dev/vdb3
Setting up swapspace version 1, size = 524284 KiB
no label, UUID=9a906cce-e50e-44ee-a880-b3811252e5e1

$ sudo blkid
/dev/vda1: UUID="9bf6b9f7-92ad-441b-848e-0257cbb883d1" TYPE="xfs"
/dev/vdb1: UUID="796676d5-0e5c-4023-82e3-3417e8d00952" TYPE="xfs" PARTLABEL="Linux filesystem" PARTUUID="bf36a837-22e7-44c5-8c74-9d9fe8161aa3"
/dev/vdb2: UUID="a875704a-d0ba-4542-98a6-f201f1b3a539" TYPE="swap" PARTLABEL="Linux swap" PARTUUID="794f5ef2-2fec-410d-a72d-9195ded90386"
/dev/vdb3: UUID="9a906cce-e50e-44ee-a880-b3811252e5e1" TYPE="swap" PARTLABEL="Linux swap" PARTUUID="ff0fe2fa-a946-4d83-9290-4a0629767548"
```

### 3、建立目錄並掛載 partition

```bash
$ sudo mkdir /backup
$ echo "UUID=796676d5-0e5c-4023-82e3-3417e8d00952 /backup xfs defaults 0 2" | sudo tee --append /etc/fstab
$ echo "UUID=a875704a-d0ba-4542-98a6-f201f1b3a539 swap swap defaults 0 0" | sudo tee --append /etc/fstab
$ echo "UUID=9a906cce-e50e-44ee-a880-b3811252e5e1 swap swap pri=1 0 0" | sudo tee --append /etc/fstab

$ sudo mount -a
$ sudo swapon -a
```

### 4、驗證是否設定完成

```bash
$ sudo swapon -s
Filename                                Type            Size    Used    Priority
/dev/vdb2                               partition       524284  0       -1
/dev/vdb3                               partition       524284  0       1

$ sudo mount
...
/dev/vdb1 on /backup type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
```
