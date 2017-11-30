---
layout: post
title:  "[Linux KVM] libvirt & Storage"
description: "This article introduces how to use libvirt to manage storage environment in KVM"
date: 2016-10-27 09:50:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM, libvirt, Storage]
---

建立 & 使用 unmanaged storage
============================

建立 unmanaged storage 是幫 VM 增加 virtual disk 最快的方式，其中有兩種作法：

1. **preallocated**：效能好，但完全佔據磁碟空間

2. **thin-provisioned**：效能較差，但僅佔據實際使用到的磁碟空間

以下是兩種不同 image 的建立示範：

```bash
# 產生 preallocated image
$ dd if=/dev/zero of=/tmp/dbvm_disk1.img bs=1G count=10
10+0 records in
10+0 records out
10737418240 bytes (11 GB) copied, 8.32924 s, 1.3 GB/s

# 產生 thin-provisioned image
$ dd if=/dev/zero of=/tmp/dbvm_disk1_seek.img bs=1G seek=10 count=0
0+0 records in
0+0 records out
0 bytes (0 B) copied, 0.000307303 s, 0.0 kB/s

# 查詢 image 資訊 (preallocated image 已經完全佔據磁碟空間)
$ qemu-img info /tmp/dbvm_disk1.img 
image: /tmp/dbvm_disk1.img
file format: raw
virtual size: 10G (10737418240 bytes)
disk size: 10G
# 查詢 image 資訊 (thin-provisioned image 並未預先佔據磁碟空間)
$ qemu-img info /tmp/dbvm_disk1_seek.img 
image: /tmp/dbvm_disk1_seek.img
file format: raw
virtual size: 10G (10737418240 bytes)
disk size: 0
```

當 image 建立完成，可用以下指令直接掛載到執行中的 VM：

```bash
$ virsh domblklist centos7
Target     Source
------------------------------------------------
vda        /var/lib/libvirt/images/hdd/vmdisk/centos7.qcow2

# vdb => 指定 image 掛載為 vdb
# --live => 指定在執行中的 VM 掛載 image
# --config => 讓此掛載設定可以永久保留，不會因為 VM reboot 而消失
$ virsh attach-disk centos7 /tmp/dbvm_disk1.img vdb --live --config
Disk attached successfully

$ virsh domblklist centos7
Target     Source- [KVM XML 設定檔基本內容](http://www.tfcis.org/~lantw44/download/notes/cs/libvirt.domain(5))
------------------------------------------------
vda        /var/lib/libvirt/images/hdd/vmdisk/centos7.qcow2
vdb        /tmp/dbvm_disk1.img
```


--------------------------------------


建立 & 使用 managed storage
==========================

為了讓 storage 有個統一標準的管理，除非是很臨時的測試目的需求，不然上面 unmanage 的作法就盡量少作囉!

libvirt 支援了相當多種的 storage pool，以下一一列出：

- `-dir`：使用**標準的檔案系統目錄**儲存 virtual disk

- `-disk`：使用**實體磁碟機**來建立 virtual disk

- `-fs`：使用**預設格式化好的磁碟分割**來儲存 virtual disk

- `-netfs`：使用 **network-shared storage**(例如：NFS) 來儲存 virtual disk

- `-gluster`：使用 **glusterfs** 儲存 virtual disk

- `-iscsi`：使用 **iscsi storage** 儲存 virtual disk

- `-scsi`：使用 **本地端 scsi storage** 儲存 virtual disk 

- `-lvm`：使用 **LVM volume group** 儲存 virtual disk

- `-rbd`：使用 **Ceph storage** 儲存 virtual disk 

> 在 libvirt 中，managed storage 是以 `pool` + `volume` 所組合而成


```bash
# 查詢 pool 列表
$ $ virsh -c qemu+ssh://root@10.20.190.2/system pool-list
 Name                 State      Autostart 
-------------------------------------------
 default              active     yes       

# 檢視 pool 的詳細資訊
$ virsh -c qemu+ssh://root@10.20.190.2/system pool-info default
Name:           default
UUID:           0896bbcd-a502-4ed8-b484-34d8baf05e84
State:          running
Persistent:     yes
Autostart:      yes
Capacity:       1007.80 GiB
Allocation:     70.07 GiB
Available:      937.73 GiB
``` 

這些資訊會以 XML 的形式存在於 KVM host 上，以 `default` 為例，其 XML 定義檔的位置為 `/etc/libvirt/storage/default.xml`。


--------------------------------------


管理 Storage Pool
=================

以下使用 filesystem & LVM 作為建立 storage pool 的範例：

## (1) 建立 fife system directory backed storage pool

這是上述 `defautl` pool 的方式，使用的是 KVM host 上的 `/var/lib/libvirt/images` 資料夾作為儲存 volume 的位置

以下是建立一個名稱為 **dedicated_storage** 的簡單方式：

```bash
# 定義 storage pool (會產生 XML 定義檔案在 /etc/libvirt/storage 目錄中)
$ virsh -c qemu+ssh://root@10.20.190.2/system pool-define-as dedicated_storage dir - - - - "/vms"
Pool dedicated_storage defined

# 建立 storage pool (建立指定目錄 & 設定 SELinux 相關權限)
$ virsh -c qemu+ssh://root@10.20.190.2/system pool-build dedicated_storage
Pool dedicated_storage built

# 啟動 storage pool
$ virsh -c qemu+ssh://root@10.20.190.2/system pool-start dedicated_storage
Pool dedicated_storage started

# 設定 libvirtd 啟動時，同時啟動此 storage pool
$ virsh -c qemu+ssh://root@10.20.190.2/system pool-autostart dedicated_storage
Pool dedicated_storage marked as autostarted
```

## (2) 建立 LVM volume Group backed storage pool

使用 LVM 的優點就會有以下優點啦：

1. 彈性伸縮磁碟容量

2. 整合不同的磁碟

3. volume snapshots

4. 自定義的裝置名稱

5. data striping 提昇 I/O throughput

6. Mirror volumes

所以使用 LVM 作為 storage pool 也是個相當不錯的選項。

假設目前在 KVM host 中有 **/dev/sdb** & **/dev/sdc** 兩個硬碟可拿來作為 LVM volume，可用以下指令建立 LVM storage pool：

```bash
$ bash -c 'cat <<EOF > /tmp/storage-lvm.xml
<pool type="logical">
  <name>HostVG</name>
  <source>
    <device path="/dev/sdb"/>
    <device path="/dev/sdc"/>
  </source>
  <target>
    <path>/dev/HostVG</path>
  </target>
</pool>
EOF'

$ virsh -c qemu+ssh://root@10.20.190.2/system pool-define /tmp/storage-lvm.xml 
Pool HostVG defined from /tmp/storage-lvm.xml

$ virsh -c qemu+ssh://root@10.20.190.2/system pool-build HostVG
Pool HostVG built

$ virsh -c qemu+ssh://root@10.20.190.2/system pool-start HostVG
Pool HostVG started

$ virsh -c qemu+ssh://root@10.20.190.2/system pool-autostart HostVG
Pool HostVG marked as autostarted
```

## (3) 刪除 storage pool

刪除 storage pool 就相對簡單，假設要刪除上面的 **HostVG** pool，只要透過以下指令即可：

```bash
$ virsh -c qemu+ssh://root@10.20.190.2/system pool-destroy HostVG
$ virsh -c qemu+ssh://root@10.20.190.2/system pool-undefine HostVG
```


--------------------------------------


Storage Volume 的管理
====================

透過 virsh 建立 storage volume 的語法類似如下：

> virsh vol-create-as --pool POOL_NAME VOL_NAME VOL_SIZE --format raw\|qcow2\|qed 

因此假設我們要在 pool **dedicated_storage** 中建立一個格式為 **qcow2**，大小為 **10G** 的 volume，可用下列語法：

```bash
# 建立一個名稱為 vm_vol1.qcow2 的 storage volume
$ $ virsh -c qemu+ssh://root@10.20.190.2/system vol-create-as --pool dedicated_storage vm_vol1.qcow2 10G --format qcow2
Vol vm_vol1.qcow2 created

$ virsh -c qemu+ssh://root@10.20.190.2/system vol-info --pool dedicated_storage vm_vol1.qcow2
Name:           vm_vol1.qcow2
Type:           file
Capacity:       10.00 GiB
Allocation:     196.00 KiB
``` 

若要刪除 storage volume，則可用以下指令

```bash
$ virsh -c qemu+ssh://root@10.20.190.2/system vol-delete --pool dedicated_storage vm_vol1.qcow2
Vol vm_vol1.qcow2 deleted
```

----------------------------------------------------------

References
==========

- [Beakdoosan's Weblog: LVM 筆記 - 觀念篇](http://beakdoosan.blogspot.tw/2011/01/lvm.html)

- [libvirt: Storage Management](https://libvirt.org/storage.html)

- [Suse Doc: Virtualization with KVM - Managing Storage with virsh](https://www.suse.com/documentation/sles11/book_kvm/data/sec_libvirt_storage_virsh.html)

- [Using libvirt with Ceph RBD — Ceph Documentation](http://docs.ceph.com/docs/hammer/rbd/libvirt/)

- [KVM XML 設定檔基本內容](http://www.tfcis.org/~lantw44/download/notes/cs/libvirt.domain(5))