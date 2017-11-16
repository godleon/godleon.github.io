---
layout: post
title:  "[Linux KVM] Template & Snapshot 的運用"
description: "This article introduces how to use template and snapshot in KVM"
date: 2017-01-01 23:00:00
published: true
comments: true
categories:
  - KVM
tags:
  - Linux
  - KVM
---

此篇文章介紹如何使用 KVM 中的 template & snapshot 功能

Overview
========

虛擬化技術有些非常吸引人使用的特性，例如：

- Fast Provisioning

- Snapshots

- 不複雜的 backup & recovery 方式

以上這些特性都不是在實體環境上容易實現的，但透過 KVM 中的 template，可以實現 fast provisioning，而透過 snapshot 則可簡單實現 backup & recovery。


----------------------------------------------------------


VM Templates
============

template 有別於一般的 VM clone，clone 只是從其他 VM 複製成另外一個完整的 VM；而 template 則是可以作為其他 VM 的 master copy，並用來產生很多個 clone

## 建立 template

template 是由以存在的 VM 所轉換過來的，因此在建立 template 之前，我們必須先完成以下步驟：

1. 安裝 & 設定 VM，確認上面已經安裝所需要的所有軟體套件

2. 移除所有系統特定的設定，確保只與此 VM 相關的設定(例如：固定 IP)不會被複製到其他 VM 上

3. 修改 VM 名稱讓其容易辨識，例如以 **template** 作為開頭


### (1) 建立 Centos Template

要建立 Linux template，必須透過 `virt-sysprep` 工具的協助。

這工具由 `libguestfs-tools-s` 套件所提供，可以移除 VM 中系統特定的資訊，以便於轉換成 template 之用；此外也可以客製化 VM，例如加上 SSH Key、加入使用者、設定 Logo .... 等等。

輸入 `virt-prep --help` 可以知道 virt-sysprep 支援哪些調整選項：

```bash
[root@ocp-kvm-host ~]# virt-sysprep -help
virt-sysprep: reset or unconfigure a virtual machine so clones can be made

 virt-sysprep [--options] -d domname

 virt-sysprep [--options] -a disk.img [-a disk.img ...]

A short summary of the options is given below.  For detailed help please
read the man page virt-sysprep(1).

  -a file                             Add disk image file
  --add file                          Add disk image file
  -c uri                              Set libvirt URI
  --chmod PERMISSIONS:FILE            Change the permissions of a file
  --connect uri                       Set libvirt URI
  -d domain                           Set libvirt guest name
  --debug-gc                          Debug GC and memory allocations (internal)
  --delete PATH                       Delete a file or directory
  --domain domain                     Set libvirt guest name
  --dry-run                           Perform a dry run
  --dryrun                            Perform a dry run
  --dump-pod                          Dump POD (internal)
.....
```

從 help 中的說明可以看出，virt-sysprep 有兩個參數分別是 `-d` & `-a`，其中 `-d` 所處理的對象是 VM，而 `-a` 所處理的對象則是獨立的 virtual disk。

以下使用對 VM 為範例來操作：

```bash
[root@ocp-kvm-host ~]# virsh list --all
 Id    Name                           State
----------------------------------------------------
 -     centos7                        shut off
.....

[root@ocp-kvm-host ~]# virt-sysprep -d centos7
[   0.0] Examining the guest ...
[  43.0] Performing "abrt-data" ...
[  43.0] Performing "bash-history" ...
........
[  43.0] Performing "udev-persistent-net" ...
[  43.0] Performing "utmp" ...
[  43.0] Performing "yum-uuid" ...
[  43.0] Performing "customize" ...
[  43.0] Setting a random seed
[  43.0] Performing "lvm-uuids" ...
```

> 到此步驟時，VM template 已經準備完成；還有一點更重要的是，**不要再啟動 template VM**，否則將會失去之前 virt-sysprep 所完成的結果，甚至有可能會在使用 thin method(參考下方) 產生 VM 時發生問題 


### (2) 建立 Windows Template

要製作 template，可以透過 `virt-clone` + `virt-sysprep` 的方式

要透過 template 建立新的 VM，有以下兩種方式可以進行：

## 透過 template 佈署 VM

1. **thin method**
> 此方式會以 template image 為 base image以 read-only(template VM image) 搭配 copy-on-write(新的 VM) 來處理，所需要的磁碟空間比較小，但需要確保 base image 可以被存取

2. **clone method**
> 此方式會完整複製一份 template image 作為 新 VM 的 image，會消耗較多的磁碟空間，但可以完全獨立不依賴原本的 base image


-------------------------------------------------

Snapshots
=========

snapshot 是以檔案為基礎，用來表示 VM 在某個特定時間點的狀態，包含了相關設定檔 & disk 資料；而透過 snapshot，管理者可以隨時將 VM 還原到當時建立 snapshot 時的狀態，而這功能在進行對於 VM 要進行重大變更前時特別好用。

此外，libvirt 還提供了 live snapshot 的功能，可以針對執行中的 VM 進行 snapshot，但這功能不建議使用在 I/O 工作頻繁的 VM 上，建議這類的 VM 還是 shutdown or suspend 之後再來做 snapshot 會較好。

libvirt 支援兩種 snapshop，分別如下：

### internal snapshot

internal snapshot 的 snapshot 資訊會存在於同一個 qcow2 檔案中(before/after snapshot bit)，有以下的限制需要注意：

- 僅支援 qcow2 format

- 當建立 snapshot 時，VM 會進入暫停狀態

- 無法使用在 LVM storage pool 上

### external snapshot

external snapshot 是以 copy-on-write 的概念進行的，當 VM 進行 snapshot 後，system disk 就會進入 read-only 的模式，後續 VM guest 新增的資料就會放在 overlay disk image 上，以下有個圖示來說明：

![copy-on-write overlay disk image](https://github.com/godleon/godleon.github.io/blob/master/_posts/images/2017/KVM-Template-And-Snapshot/copy-on-write_overlay-disk0-image.png?raw=true)

external snapshot 也有以下特點：

1. overlay disk image 的起始大小為 0，最大可以到 original disk 的大小

2. base disk 可以是任何的格式(例如：raw, qcow2, 或是其他 libvirt 支援的格式)

3. overlay disk image 一定是 qcow2 格式


## VM Disk Image Format

libvirt 支援很多種不同的 disk foramt，包含 iso, dmg, qcow2, raw, vmdk, vpc ... 等等，但與 libvirt 搭配起來運作的最好的是 **raw** & **qcow2** 兩種：

### (1) **raw**

- 效能最佳，performace overhead 最低，適合給有高度 I/O 需求的 VM 使用

- 完全佔據 disk 所指定的空間

- 沒有 snapshot & compression 的功能

### (2) **qcow2**

- 完全以 cloud 架構出發所設計的格式

- 支援 read-only backing、snapshot、compression、encryption、pre-allocation ... 等功能


## 轉換 Disk Image Format

以下介紹如何透過 `qemu-img` 指令在 **raw** & **qcow2** 之間進行轉換：

```bash
# raw -> qcow2
$ qemu-img convert -f raw -O qcow2 vm_disk.img vm_disk.qcow2

# qcow2 -> raw
$ qemu-img convert -f qcow2 -O ram vm_disk.qcow2 vm_disk.img
```


## 操作 Internal Snapshot

### (1) 建立 & 檢視 snapshot

```bash
$ virsh snapshot-list rhel7.3
 Name                 Creation Time             State
------------------------------------------------------------

# 建立 internal snapshot
$ virsh snapshot-create rhel7.3
Domain snapshot 1481514143 created

# 建立 internal snapshot (使用 snapshot-create-as 搭配 --name, --description 等參數)
# --atomic 可以確保 snapshot 是可以用的 (若是 snapshot 無法使用，建立時就會 fail)
$ virsh snapshot-create-as rhel7.3 --name "rhel7.3_snapshot_1" --description "First named snapshot" --atomic
Domain snapshot rhel7.3_snapshot_1 created

$ virsh snapshot-list rhel7.3
 Name                 Creation Time             State
------------------------------------------------------------
 1481514143           2016-12-12 11:42:23 +0800 shutoff
 rhel7.3_snapshot_1   2016-12-14 08:32:23 +0800 shutoff

# 顯示 snapshot 資訊
$ virsh snapshot-info rhel7.3 --snapshotname 1481514143
Name:           1481514143
Domain:         rhel7.3
Current:        no
State:          shutoff
Location:       internal
Parent:         -
Children:       1
Descendants:    1
Metadata:       yes

# 檢視指定 VM 的 current snapshot 資訊
$ virsh snapshot-current rhel7.3
<domainsnapshot>
  <name>rhel7.3_snapshot_1</name>
  <description>First named snapshot</description>
  <state>shutoff</state>
  <parent>
    <name>1481514143</name>
  </parent>
  <creationTime>1481675543</creationTime>
  <memory snapshot='no'/>
  <disks>
    <disk name='vda' snapshot='internal'/>
  </disks>
  <domain type='kvm'>
    ....
  </domain>
</domainsnapshot>

# 顯示 snapshot XML 資訊
$ virsh snapshot-dumpxml rhel7.3 --snapshotname 1481514143
<domainsnapshot>
  .... 與上面類似
</domainsnapshot>

# 顯示 snapshot 的 parent snapshot (第一個 snapshot 沒有 parent snapshot)
$ virsh snapshot-parent rhel7.3 --snapshotname 1481514143
error: snapshot '1481514143' has no parent

# 第二個 snapshot 就有 parent snapshot 了
$ virsh snapshot-parent rhel7.3 --snapshotname rhel7.3_snapshot_1
1481514143

# 顯示每個 snapshot 的 parent
$ virsh snapshot-list rhel7.3 --parent
 Name                 Creation Time             State           Parent
------------------------------------------------------------
 1481514143           2016-12-12 11:42:23 +0800 shutoff         (null)
 rhel7.3_snapshot_1   2016-12-14 08:32:23 +0800 shutoff         1481514143

# 以樹狀結構顯示每個 snapshot 的關係
$ virsh snapshot-list rhel7.3 --tree
1481514143
  |
  +- rhel7.3_snapshot_1
```

> 若是在 VM 正在 running 的狀態下建立 snapshot，會需要多花一點時間，因為 VM 必須先進入到 **pause** 的狀態，當 snapshot 完成後才會回到 running 的狀態；而這一段時間的長短取決於 VM 當時耗用了多少記憶體，以及當時對記憶體存取的頻繁程度。

另外還可以透過 `qemu-image` 工具來查詢 VM image 的狀態：

```bash
# 查詢 image 的狀態
$ qemu-img info /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.qcow2 
image: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.qcow2
file format: qcow2
virtual size: 40G (42949672960 bytes)
disk size: 572M
cluster_size: 65536
Snapshot list:
ID        TAG                 VM SIZE                DATE       VM CLOCK
1         1481514143                0 2016-12-12 11:42:23   00:00:00.000
2         rhel7.3_snapshot_1        0 2016-12-14 08:32:23   00:00:00.000
Format specific information:
    compat: 0.10
    refcount bits: 16

# 檢查 image 是否有錯誤
# 適合用在 running 過程中建立 snapshot 後，檢查有無錯誤發生
$ qemu-img check /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.qcow2 
No errors were found on the image.
19467/655360 = 2.97% allocated, 93.71% fragmented, 90.63% compressed clusters
Image end offset: 599916544
```

### (2) 透過 snapshot 還原 VM

接著介紹如何透過 internal snapshot 還原 VM：

```bash
# 從指定的 snapshot 還原 VM
$ virsh snapshot-revert rhel7.3 --snapshotname rhel7.3_snapshot_1

# 從指定的 snapshot 還原 & 自動啟動 VM
$ virsh snapshot-revert rhel7.3 --snapshotname rhel7.3_snapshot_1 --running
```

### (3) 刪除 snapshot

```bash
# 刪除指定的 snapshot
$ virsh snapshot-delete rhel7.3 1481514143
Domain snapshot 1481514143 deleted

# 顯示目前 snapshot 的狀態
[root@ocp-kvm-host ~]# virsh snapshot-list rhel7.3 --parent
 Name                 Creation Time             State           Parent
------------------------------------------------------------
 rhel7.3_snapshot_1   2016-12-14 08:32:23 +0800 shutoff         (null)
```


## 操作 External Snapshot

external snapshot 的原理是由 `overlay_image` & `backing_file` 兩個所組成；其中 backing file 會變成 read-only，後續使用者針對 VM 的變更都會寫到 overlay_image 中。

而 external snapshot 比較有優勢的地方，在於支援各種不同的 disk image type(raw, qcow, vmdk ... etc)，不僅有 qcow2 而已。

但由於目前 virsh 還不完全支援 external snapshot 的操作，這邊就先保留，等到 virsh 可以完整支援 external snapshot 之後再來補。

### (1) 建立 external snapshot

```bash
$ virsh list
 Id    Name                           State
----------------------------------------------------
 32    rhel7.3                        running

$ virsh domblklist rhel7.3 --details
Type       Device     Target     Source
------------------------------------------------
file       disk       vda        /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.qcow2

$ virsh snapshot-create-as rhel7.3 ext_snapshot1 "first external snapshot" --disk-only --atomic
Domain snapshot ext_snapshot1 created

$ virsh snapshot-list rhel7.3
Name                 Creation Time             State
------------------------------------------------------------
ext_snapshot1        2016-12-31 21:00:23 +0800 disk-snapshot

$ virsh snapshot-info rhel7.3 ext_snapshot1
Name:           ext_snapshot1
Domain:         rhel7.3
Current:        yes
State:          disk-snapshot
Location:       external
Parent:         -
Children:       0
Descendants:    0
Metadata:       yes
```

利用上面的命令就可以建立 VM external snapshot；此外，還有幾點需要注意：

1. external snapshot 可以在 VM running 的狀態下建立(若是 disk I/O 頻繁的 VM，建議還是 shutdown 之後再作)

2. `--disk-only` 表示只針對 disk 作 snapshot

3. `--atomic` 會確保 snapshot 建立的工作執行成功後會得到一個可用的 snapshot；若是中途發生任何問題，就不會有 snapshot 的產生，也不會對原有的 VM 產生任何異動，藉此確保建立 snapshot 不會損壞原有的 VM


```bash
# 檢視目前 VM disk 資訊
# 已經被 external disk 取代了!
$ virsh domblklist rhel7.3 --details
Type       Device     Target     Source
------------------------------------------------
file       disk       vda        /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot1
 
# 可看出原本的 disk 已經變成了 backing file
$ qemu-img info /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot1
image: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot1
file format: qcow2
virtual size: 40G (42949672960 bytes)
disk size: 2.0M
cluster_size: 65536
backing file: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.qcow2
backing file format: qcow2
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

從上面可看出，從此刻開始對 VM 的變更將會直接寫入 external snapshot，而原本的 disk image 變成了 backing file。

接著再建立兩個 external snapshot 試試看：

```bash
$ virsh snapshot-create-as rhel7.3 ext_snapshot2 "second external snapshot" --disk-only --atomic
Domain snapshot ext_snapshot2 created

# 透過 "--quiesce" 參數，確保連同記憶體內的資訊都一併進入到 snapshot 中
$ virsh snapshot-create-as rhel7.3 ext_snapshot3 "third external snapshot" --disk-only --quiesce
Domain snapshot ext_snapshot3 created

$ virsh snapshot-list rhel7.3
 Name                 Creation Time             State
------------------------------------------------------------
 ext_snapshot1        2016-12-31 21:00:23 +0800 disk-snapshot
 ext_snapshot2        2016-12-31 22:16:42 +0800 disk-snapshot
 ext_snapshot3        2016-12-31 22:17:53 +0800 disk-snapshot
```

在第三個 snapshot 中使用了 `--quiesce` 參數，目的就是要讓尚未寫入 disk(記憶體中的資料) 一併進入到 snapshot 中，這樣就可以確保 snapshot 是最完整的狀態，但要使用這參數必須預先在 VM 上安裝 `qemu-guest agent` 才可以。

最後，每個 snapshot 的相依性可以使用下列命令觀察：

```bash
$ qemu-img info --backing-chain /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot3 | grep backing
backing file: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot2
backing file format: qcow2
backing file: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot1
backing file format: qcow2
backing file: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.qcow2
backing file format: qcow2
```

### (2) 從 external snapshot 還原

external snapshot 看似不錯，但其實無法透過 virsh 指令直接還原 VM：

```bash
$ virsh snapshot-revert rhel7.3 --snapshotname "ext_snapshot3"
error: unsupported configuration: revert to external snapshot not supported yet
```

但這並不代表沒辦法從 external snapshot 還原，只是要透過以下步驟來完成：(假設要還原到 `ext_snapshot2`)

1. 關閉 VM (**這是必須的!**)

2. 檢查要還原的 external snapshot overlay image 有無損毀

3. 若 snapshot 完整無誤，編輯 VM XML 定義檔，將 boot disk 指向 `ext_snapshot2`

4. 確認 external snapshot image 格式

5. 從 VM XML 定義中移除原本的 disk，改成指定要還原的 `ext_snapshot2`

6. 透過 `domblklist` 參數確認 VM 使用的 disk 已經指向 ext_snapshot2

7. 重新啟動 VM

以下是實際操作步驟：

```bash
# 關閉 VM
$ virsh shutdown rhel7.3

# 取得 ext_snapshot2 的詳細路徑
$ virsh snapshot-dumpxml rhel7.3 --snapshotname ext_snapshot2 | grep ext_snapshot2
  <name>ext_snapshot2</name>
      <source file='/var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot2'/>

# 先使用 qemu-img 工具檢查 overlay image 有無損毀
$ qemu-img check /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot2
No errors were found on the image.
11/655360 = 0.00% allocated, 36.36% fragmented, 0.00% compressed clusters

# 確認 overlay image 格式
$ qemu-img info /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot2 | grep backing
backing file: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot1
backing file format: qcow2

# 移除原本的 disk
$ virt-xml rhel7.3 --remove-device --disk target=vda
Domain 'rhel7.3' defined successfully.

# 換上要還原的 ext_snapshot2
$ virt-xml rhel7.3 --add-device --disk /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot2,format=qcow2,bus=virtio
Domain 'rhel7.3' defined successfully.

# 確認目前 VM 所使用的 disk
$ virsh domblklist rhel7.3
Target     Source
------------------------------------------------
vda        /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot2

# 重新啟動 VM
$ virsh start rhel7.3
Domain rhel7.3 started
```

### (3) 刪除 external snapshot

由於 virsh 不支援 external snapshot 的刪除，所以刪除 snapshot 就必須自己來了!

假設要移除所有的 snapshot，但又想要保留在 snapshot 上完整的變更，此時必須把 snapshot merge 到 base image 上，以下是操作步驟：

```bash
# 目前 disk 所使用的 snapshot
$ virsh domblklist rhel7.3
Target     Source
------------------------------------------------
vda        /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot2

# 查詢 snapshot overlay image 的相依關係
$ qemu-img info --backing-chain /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot2 | grep backing
backing file: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.ext_snapshot1
backing file format: qcow2
backing file: /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.qcow2
backing file format: qcow2

# 將 base image & snapshot overlay image 合併!
$ virsh blockcommit rhel7.3 vda --verbose --pivot --active
Block commit: [100 %]
Successfully pivoted

# 可看出 VM disk 已經變成 base image
$ virsh domblklist rhel7.3
Target     Source
------------------------------------------------
vda        /var/lib/libvirt/images/rhel-guest-image-7.3-35.x86_64.qcow2

$ virsh snapshot-list rhel7.3
 Name                 Creation Time             State
------------------------------------------------------------
 ext_snapshot1        2016-12-31 21:00:23 +0800 disk-snapshot
 ext_snapshot2        2016-12-31 22:16:42 +0800 disk-snapshot
 ext_snapshot3        2016-12-31 22:17:53 +0800 disk-snapshot
```

確認好 VM disk 已經不是指向 external snapshot 了，就可以開始進行刪除 snapshot 的動作，而刪除 external snapshot 必須從 metadata 下手，以下為操作範例：

```bash
# 指定刪除從 ext_snapshot1 開始的所有 children snapshot 的 metadata & image
$ virsh snapshot-delete rhel7.3 ext_snapshot1 --children --metadata
Domain snapshot ext_snapshot1 deleted

# 確認所有 external snapshot 都已經被刪除
$ virsh snapshot-list rhel7.3
 Name                 Creation Time             State
------------------------------------------------------------
```


## 使用 snapshot 時所需的正確觀念

1. 不建議在 production 的環境中讓 VM 去 attach 之前做好的 snapshot 來使用

2. 不要把 snapshot 當作是備份的方式，只是用來留下當時 VM 的狀態做後續使用而已

3. snapshot 不要保留太久，若確定不需要的就把 snapshot commit(for external snapshot) or 刪除

4. external snapshot 出現故障的機率比 internal snapshot 還低，因此建議優先使用 external snapshot

5. snapshot 數量要控制，太多的 snapshot 可能會倒置系統效能低落

6. 建立 snapshot 前要先安裝 guest agent

7. 建立 snapshot 確認都有帶上 `--quiesce` && `--atomic` 兩個參數