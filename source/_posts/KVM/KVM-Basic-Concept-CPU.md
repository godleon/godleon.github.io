---
layout: post
title:  "[Linux KVM] Linux KVM concept - CPU"
description: "This article introduces what should to learn about CPU when learning Linux KVM virtualization"
date: 2016-07-30 22:25:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM]
---

此篇文章介紹學習 Linux KVM 時所需要了解的 CPU 相關知識

前言
====

KVM 在 Linux x86 硬體平台上提供了全虛擬化(Full Virtualization)的 solution，透過 QEMU 的模擬，顯示特定數量的 CPU & 相關 feature 給使用者，而在支援 KVM 的前提下，virtual machine 的 CPU 指令則是直接由 CPU 來輔助執行，藉此大幅提升運作效率。

---------------------------------------------------------------------

vCPU
====

QEMU/KVM 提供每一台 virtual machine 有一個模擬的完整硬體環境，在 virual machine 中所看到的 CPU，即是 host machine 上的 vCPU。

在 KVM 環境中，每一台 virtual machine 都是一個的 QEMU userspace process，而 vCPU 則是 QEMU process 中的 thread。

![KVM environment](http://image.slidesharecdn.com/els305-100323102407-phpapp02/95/virtualization-with-kvm-kernelbased-virtual-machine-4-728.jpg)

vCPU 一共有以下三種執行模式：

1. User Mode

2. Kernel Mode

3. Guest Mode

其中前兩個執行模式(**User Mode** & **Kernel Mode**)是一般的 process 所擁有的執行模式，詳細的說明可以參考下面連結：

- [[系程] 教學: 簡介 Kernel/User Mode 的概念 - 看板 b97902HW - 批踢踢實業坊](https://www.ptt.cc/bbs/b97902HW/M.1267018497.A.3B1.html)

- [企鵝幫魚，魚幫兔: User mode vs. Kernel mode](http://peachwaneversay.blogspot.tw/2007/05/user-mode-vs-kernel-mode.html)

而 KVM 多了一個 **Guest Mode**，功能是用來**執行關於 virtual machine 中的相關 I/O request**，無法直接；所有 Memory & CPU 的 I/O request，會透過 **/dev/kvm(QUME)** 來模擬完成，並可透過 QEMU 執行一些特權指令來存取 host machine 的資源。

![KVM Guest Mode](http://benjr.tw/wp-content/uploads/2013/11/kvm_qemu01.png)

---------------------------------------------------------------------

SMP(Symmetric Multi-Processor)
==============================

由於現在 multiple core、hyper threading 等相關技術已經很普遍，這讓作業系統可以進行真正的平行處理；而現在較新的作業系統都已經有對 SMP 的支援(Linux kernel 2.6 以上)，這對虛擬化的推展有相當大的助益。

透過以下指令，可以用來檢查目前 host machine 對 SMP 的支援程度：

```bash
# 查詢 logic cpu 數量
$ cat /proc/cpuinfo | grep "processor" | wc -l

# 實體 CPU 數量
$ cat /proc/cpuinfo | grep "physical id" | sort | uniq | wc -l

# 每個 CPU 上的 core 數量
$ cat /proc/cpuinfo | grep "core id" | sort | uniq | wc -l

# 每個 physical cpu 上分配的 logic cpu 的數量
$ cat /proc/cpuinfo | grep "siblings" | sort | uniq | awk -F: '{print $2}'
```

以上一篇文章中的範例來說明：

```bash
$ kvm -smp 4 -m 2048 \
  -vnc 0.0.0.0:5 -boot order=cd \
  -hda /kvm/storage/vm_disks/ubnutu1604.img \
  -cdrom /kvm/os_images/ubuntu-16.04.1-server-amd64.iso
```

其中 `-smp` 參數就是指定要使用多少的 vCPU 支援，完整的使用設定如下：(上例為使用 4 個 vCPU)

> -smp n[,**cores=**cores][,**threads=**threads][,**sockets=**sockets][,**maxcpus=**maxcpus]

## 範例 1 (僅使用 -smp)

而在 Linux 系統中，每一個 vCPU 的分配都會成為一個 process 運行在 host machine 中，以下開啟一個 vCPU=8 的 virtual machine：

> kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img

可以從 QEMU monitor 中觀察 vCPU 對應到 kvm 的 process：

![QEMU monitor - cpu infos](https://lh3.googleusercontent.com/hHP9T9hWydFXixyMUCuCVFDkl7hlZ6Yv_lbljFDdv_azyt_rh01M6X8bHI1fRMV82a2UObBH6wQgAElavdVxByZ99u0cDBToT-t9OIqgWpSY6ZGreqooEuin8WvQqXpQ9g84uwed3-qO2akJatJTCJqpY0Xt_xU9J1ak4702nyJicXQ7h6HqppYXz0G_86NhyQMd6tv2w5venHGgaBoOF46L8UcYYTskX5rPptWqlmhJtNQSkFGj8F6t3DVVGpOJQfgErjdrfFm162spjhQGwZJ6OWiKTsdBAuXbeUscC-NUTPtGbVjSvAHXa-MVo3-h6jjWZ-TlNkjSmQYCICiRiTGIcFsLAkbIVMGw7-PXkPdqXhJyaC9bEWplxhPzhgGptGUeJCAOrdIhvF_l8lXzaSqeWGL-BSnBbA5qCuV-9UUZVGrcHgiQo2p5_Q_iUBBazlkqnRuM69ULxGEeM-q82gIztEmQiqqOrynKeAf9w5MZnJeb_w29u6qu6B-6d81hRGXwc68A1YHKH256EWxgtrd7_aVfsu08fLEIsPwo5qEuDnkef9G_hGNUHccLcWMSIOIr4J1ViP1vOrDsLpAA_qW-08QfZ4Q=w463-h210-no)

以下則可以看出在 host machine 有相對應的 process 存在：

```bash
ps -efL | grep kvm
root      1122     2  1122  0    1 Jul27 ?        00:00:00 [kvm-irqfd-clean]
root     14606 11601 14606  3   11 17:46 pts/0    00:00:02 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14610 10   11 17:46 pts/0    00:00:06 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14611  2   11 17:46 pts/0    00:00:01 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14612  1   11 17:46 pts/0    00:00:01 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14613  1   11 17:46 pts/0    00:00:00 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14614  1   11 17:46 pts/0    00:00:00 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14615  1   11 17:46 pts/0    00:00:00 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14616  1   11 17:46 pts/0    00:00:00 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14617  1   11 17:46 pts/0    00:00:01 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14619  1   11 17:46 pts/0    00:00:00 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14606 11601 14631  0   11 17:46 pts/0    00:00:00 kvm -vnc 0.0.0.0:1 -smp 8 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img
root     14618     2 14618  0    1 17:46 ?        00:00:00 [kvm-pit/14606]
root     14633 12908 14633  0    1 17:47 pts/1    00:00:00 grep --color=auto kvm
```

最後可以用上面的 grep 查詢 physical / cores / threads 等資訊：

![SMP only](https://lh3.googleusercontent.com/mD_mnwyxWHinCr-2GSCfaMs-kyhi7p5c9cspDW5go_wM_76oHUj0Phj4n5dAahx39fazYXbMYa5xknxe1oRcoErP0cG84fMRu0WRwm9A1-_p4_bH0XediFmabBPzQQ6omUqJeBgJkwuyj0YRQ8-5kQ6zrE6elocc5sAt4q4tVKOXISDO0CZzSiYU9jHr6HeQVWO4xyLcBBkEUTZEeCfeR_tIrJzZF23nMGWDLI9CSu84ADbuL6KilIp4zDwQPNGrZmxNEwi2kUWyKcxC9-iZBm1W3wNdDYdIESmnVkTQuMe19XnTYsGdPiyRiOtMYv01WQ_KmBoCdRRZ1vfn0syyYdkt9PagLY3TjJ75dt57p918PI4z04YUZomaZ0lZ6Pqb8bt1Iaz1oljrslZpyveJ2AtAGWIdO9hFvE_9r3uvzK-ND77ZL1nAMbvuBPcD3rxuAGGr9ZqZTH_RACGOKtUN44cffI9VMvDp261xOj2NvYUDAVKxLogUNUj_ltOSyLGZ3uSffQtBkybY8OBAR3o3ycSsVy6Riwk8x2n6buCDR05dA4oTlaUqmR_o39TyTKsF6c1hzgoiJN9d_tc4cLS2c1PINXUh7W8=w785-h194-no)

- logic cpu = 8
- physical cpu = 8
- 每個 CPU 上的 core 數量 = 1
- 每個 physical cpu 上分配的 logic cpu 的數量 = 1

## 範例 2 (使用 -smp，搭配 sockets=2,cores=2,threads=2)

此範例搭配 socket, cores, threads 等 smp 相關參數

>  kvm -vnc 0.0.0.0:1 -smp 8,sockets=2,cores=2,threads=2 -m 2048 -hda /kvm/storage/vm_disks/ubnutu1604.img

用 grep 查詢 physical / cores / threads 等資訊：

![SMP with sockets, cores, and threads](https://lh3.googleusercontent.com/JmmR_lsIlsIsr66SlLZXaBJfVKdo6gim3rLxj7oMrVrTbDEYnluhtz-uj5FdlYwUAN4_W-FZDL2a6ykDgFBdwIlYNVrorpLPc7olTSl7oOsHlEoUVT1U9o9yDroSYZFNEGoJVJaMcKoIrtRr9HKB7R9uKrz5VNLR6iX_n2OZs5gL_UxPrxL5UOeYeqyyuZwswNDTyqh61qmqlJ2i0KBAHkSauTcUbASZScuk1rCL_wlp0B0VguRIhd2WZtq2oEe_G9nL6zRMclrXsOqhWsVBy0DyBibyGtqsUPIG7-7rYHcSGs1ao8j_fuTELJVA0P4nWoPpAX5sRI9p84M32vGuppudVSkFDdzACIo43FdB6ZkFftc4sPfRLk-qLXldYJ5ss_waqmqcIA2Qr-be1M443d0QiqYV6BF4XkIn6cKud7gCzy15I30CdM31gVZwK7rkQ6O_eWSCLYhiQoBR1y-B9jXV5qnN7axQfTC8hL46gRlQSyozEG4HXp-X2UmU12rfMWv8rZop2alI6pLa29CE5oywHl40BYI8qKgok52V0znrX1klcmgiCn5iYYdEFPYELqHwDW_EQv0-Z8oAXxmyIpk8tKnSR0E=w777-h190-no)

- logic cpu = 8
- physical cpu = 2
- 每個 CPU 上的 core 數量 = 2
- 每個 physical cpu 上分配的 logic cpu 的數量 = 4

---------------------------------------------------------------------

Over-Commit
===========

一般正常使用情況下，每一台 virtual machine 不會總是在高負載狀況，很多時間會是閒置的；此時透過 over-commit 的方式，可以分配比 host machine 中所有的 vCPU 給 virtual machine。

但不建議分配給單一 virtual machine 超過 host machine 所有 vCPU 的數量，因為這會大大降低 virtual machine 的效能，例如：host machine 總共有 4 個 vCPU，但分配 8 個 vCPU 給 virtual machine。

若是 4 個 vCPU，分配 1 個 vCPU 給 virtual machine，但分配到 8 台 virtual machine，這樣的效能會比上面的配置更有效率。

> 若是在 production 的環境，建議還是不要 over-commit

----------------------------------------------------------------------

References
==========

- [qemu-kvm(1): QEMU Emulator User Documentation - Linux man page](http://linux.die.net/man/1/qemu-kvm)

- [虛擬化技術與應用](https://ncucsie.hackpad.com/ep/pad/static/QrwxkWD88gd)

- [[系程] 教學: 簡介 Kernel/User Mode 的概念 - 看板 b97902HW - 批踢踢實業坊](https://www.ptt.cc/bbs/b97902HW/M.1267018497.A.3B1.html)

- [企鵝幫魚，魚幫兔: User mode vs. Kernel mode](http://peachwaneversay.blogspot.tw/2007/05/user-mode-vs-kernel-mode.html)
