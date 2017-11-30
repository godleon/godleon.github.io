---
layout: post
title:  "[Linux] PXE Booting"
description: "This article introduces the booting process of PXE and iPXE"
date: 2016-07-01 14:30:00
published: true
comments: true
categories: [Linux]
tags: [Linux]
---

此篇文章介紹 PXE & iPXE 的開機流程

PXE
===

首先以下圖說明一下傳統 PXE 的流程：

![PXE](https://github.com/coreos/coreos-baremetal/raw/master/Documentation/img/pxelinux.png)

PXE 的開機流程，簡單來說就是幾個步驟：

1. network client 從 DHCP service 取得 IP & 其他 metadata(例如：TFTP 資訊 & NBP 檔名)

2. network client 從 TFTP 取得 NBP(Network Boot Program，上面中的 `pxelinux.0`)，並啟動

3. network client 使用 NBP 載入 configs, scripts, 以及執行 OS 需要的 kernel(範例中的 `kernel.vmlinuz`) & ramfs image(上圖中的 `initrd.cpio.gz`)

------------------------------------------------------------------------------

Network Boot Program (NBP, 也稱為 bootloader)
============================================

CoreOS 可用多種不同的 bootloader 開機 & 設定，如果是一個全新的設定環境，[iPXE](http://ipxe.org/) 是個不錯的選擇。

## PXELINUX

[PXELINUX](http://www.syslinux.org/wiki/index.php?title=PXELINUX) 是個相當普遍被使用的 bootloader(檔名為 `pxelinux.0`)，會自動從 `/tftp_bootdir/pxelinux.cfg` 目錄中載入設定檔。

若要控制特定機器使用特定的設定檔，可以利用 network client 的 UUID、MAC address、IP or default 來進行設定，因此設定檔就會變成如下面的範例：

```bash
/tftp_bootdir/pxelinux.cfg/b8945908-d6a6-41a9-611d-74a6ab80b83d
/tftp_bootdir/pxelinux.cfg/01-88-99-aa-bb-cc-dd
/tftp_bootdir/pxelinux.cfg/default
```

按照上圖的流程，可以很清楚知道，設定檔的內容肯定就是會包含了像是 config / script / kernel / ramfs image .... 等檔案的位置資訊(可能還會包含**開機選單**)，以下是個簡單範例：

```bash
default coreos
prompt 1
timeout 15

display boot.msg

label coreos
  menu default
  kernel coreos_production_pxe.vmlinuz
  append initrd=coreos_production_pxe_image.cpio.gz cloud-config-url=http://example.com/pxe-cloud-config.yml
```

> 上面範例中的檔案若沒有指定目錄，就表示應該將其放在 `/tftp_bootdir` 目錄中
> 其中設定了一個簡單的 menu、顯示訊息、kernel(coreos_production_pxe.vmlinuz)、ramfs(coreos_production_pxe_image.cpio.gz) & 設定 CoreOS 用的 cloud-config.yaml

PXE 雖然普遍使用，但的確是有些缺點存在的，例如：

1. TFTP 速度慢

2. 若有許多針對不同機器的客製化設定檔需求，會需要撰寫很多份 pxelinux config


## iPXE

[iPXE](http://ipxe.org/) 可是視為加強版的 PXE bootloader，使用的並非是設定檔，而是 **iPXE script**，而且 iPXE script & image 都可以透過 HTTP 下載

以下是 iPXE 開機流程示意圖：

![iPXE](https://github.com/coreos/coreos-baremetal/raw/master/Documentation/img/ipxe.png)

從上圖可以看出，為了可以運作在原有的環境中，使用了一個名稱為 [undionly.kpxe](http://boot.ipxe.org/undionly.kpxe) 的 bootloader 來協助開機，接著會發生以下的事情：

1. bootloader undionly.kpxe 會提供給 machine 上網 & 處理後續 iPXE script 的能力

2. 透過網路取得 iPXE script `boot.ipxes`

3. 並會使用檔名為 `boot.ipxe`(來源可以是 HTTP) 的 iPXE script 來繼續執行後續的開機流程

以下是個簡單的 iPXE script 範例：

```bash
#!ipxe

set base-url http://stable.release.core-os.net/amd64-usr/current
kernel ${base-url}/coreos_production_pxe.vmlinuz cloud-config-url=http://provisioner.example.net/cloud-config.yml
initrd ${base-url}/coreos_production_pxe_image.cpio.gz
boot
```

> 透過 iPXE script，可以用程式化的方式進行更多動態的開機設定

在 iPXE 開機環境 for CoreOS 的架構中，有一些事情是值得注意一下的：

1. TFTP 只用來提供 [undionly.kpxe](http://boot.ipxe.org/undionly.kpxe) bootloader，目的是為了讓老舊的 PXE firmware client 也可以使用 iPXE

2. CoreOS 提供了 [bootcfg](coreos-baremetal/bootcfg.md at master · coreos/coreos-baremetal) 工具，可根據硬體的屬性，用來產生相對應的 iPXE script (把 iPXE script 的來源指向 bootcfg iPXE endpoint)

------------------------------------------------------------------------------

References
==========

- [PXELINUX - Syslinux Wiki](http://www.syslinux.org/wiki/index.php?title=PXELINUX)

- [iPXE - open source boot firmware [howto:chainloading]](http://ipxe.org/howto/chainloading)

- [coreos-baremetal/network-booting.md at master · coreos/coreos-baremetal](https://github.com/coreos/coreos-baremetal/blob/master/Documentation/network-booting.md)
