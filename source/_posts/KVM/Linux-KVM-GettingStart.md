---
layout: post
title:  "[KVM] Linux KVM 基本觀念"
description: "此文章記錄學習 Linux KVM 時所得到的基本觀念"
date: 2016-05-09 16:00:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM]
---

1、libvirt and libvirt tools
============================

## libvirt

`libvirt` 是一組獨立於 hypervisor 之外的 API：

1. 提供一個標準，一般化且穩定的虛擬層，且可安全的管理主機上的虛擬機器

2. 可用來管理本地系統以及透過網路相連主機的標準介面

3. 提供的 API 包含了 provision, create, modify, monitor, control, migrate, stop 虛擬主機等相關功能，但也必須要 hypervisor 支援的前提下才可使用，但這些 API 緊限定於單一主機上的操作

## virsh

virsh 是一個基於 libvirt management API 所打造出來的 CLI，提供非常多管理 hypervisor & guest VM 的指令，且相當方便搭配 script 一起使用，同時也是 `virt-manager` GUI tool 的基礎。

> 若不想用 virsh，也可以直接使用 `qemu-kvm` 指令

## virt-manager

圖形介面的管理工具，也是提供了很多管理 VM 的功能(基於 virsh 為基礎所開發出來的)，包含了 provision VM, 管理 virtual network, 存取 VM console, 檢視效能統計資訊....等等。

## virt-install

virt-install 是專門用來協助 provision VM 的 CLI，支援純文字 & 圖形安裝(可透過 serial, SPICE or VNC 等不同協定)，且可指定 local or remote(NFS, HTTP or FTP) 的安裝媒體，搭配 Kickstart 作大量自動化安裝很好用。

------------------------------------------------

2、Virtualized hardware devices
===============================

##  Para-virtualized devices

para-virtualized 技術提供了 VM 一個更快速且有效率的方式跟 host 主機溝通；而 KVM 透過 `virtio` API 作為中介層，提供 para-virtualized devices 給 VM 使用，

對於 I/O 工作頻繁的 VM 來說，建議使用 para-virtualized devices 來取代 emulated devices。

所有的 virtio device 都包含兩個部份：`host device` & `guest driver`，而 Para-virtualized device drivers 的目的是在於協助 guest VM 可以直接存取 host 主機上的實體裝置而不需要再經過模擬轉換。

目前有 virtio-net, virtio-blk(block device), virtio-scsi(效能比 virtio-blk 好很多), virtio-balloon ..... 等等。

## VFIO

Virtual Function I/O (VFIO) 是個 kernel driver，讓 guest VM 可直接高效率的存取 host 主機上的硬體裝置；VFIO 將 device assignment 的工作移出 KVM hypervisor 中，並將實體裝置在 kernel level 中獨立出來以達成直接被 VM 存取的目的。

## SR-IOV

SR-IOV (Single Root I/O Virtualization) 是用在 PCI-e 的裝置上，讓裝置可以分享自身的 virtual fcuntion(VF) 出來直接給 guest VM 使用的一種技術。

## NPIV

N_Port ID Virtualization (NPIV) 是種應用在高速企業級儲存裝置的功能(例如：SAN)，功能類似 SR-IOV，都是讓 VM 可以直接存取硬體支援的技術。

------------------------------------------------

3、Storage
==========

## Disk Images 的存在型式

Disk Image 會根據在本地 or 遠端存放的不同，而已不同的型式存在：

1. Image files

直接以 image 檔案的方式存在，這又可分為 raw & qcow2 兩種格式，其中 raw 格式速度快，但功能少；而 qcow2 則提供很多其他功能(例如：snapshot, compression, encryption, 從 template image 啟動 VM .... 等等)。

2. LVM volumes

Logical volume 可以直接作為 disk image 使用，同時也提供了 thin provision, snapshot 等功能。

3. Host devices

可以直接使用 CD-ROM, 或是透過 SAN or iSCSI 掛載的 LUN 作為 image

4. Distributed storage systems

在 RHEL7 中甚至支援把 disk image 放在 GlusterFS 上

------------------------------------------------

參考資料
=======

- [https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Getting_Started_Guide/index.html](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Virtualization_Getting_Started_Guide/index.html)
