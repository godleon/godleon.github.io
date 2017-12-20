---
layout: post
title:  "[RHCE7] RH134 Chapter 1 Automating Installation with Kickstart 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 1 Automating Installation with Kickstart 留下的內容"
date: 2016-04-29 11:50:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

`dmesg` 觀察由 Linux Kernel 所產生的 log 檔

```bash
# 製作 USB 安裝裝置
# /dev/sr0 為光碟機裝置
# /dev/sdb1 為 USB 裝置
dd if=/dev/sr0 of=/dev/sdb1

# 將 DVD 光碟轉成 iso 檔
$ dd if=/dev/sr0 of=/tmp/rhel_dvd.iso
```

安裝 Linux 方式：
- DVD/USB
- Hard Disk
- Network (必須先安裝 YUM server)
  - FTP
  - HTTP
  - NFS
- PXE boot (找 CentOS PXE server)


Kickstart:
1. 使用 boot.iso 開機
2. 準備好 `ks.cfg`(安裝中需要的設定參數)
3. 會進入到 `Boot:`，並輸入 `Boot: linux ks=floppy`，接著程式就會去找 **<font color='red'>ks.cfg</font>** 並開始安裝

> ks.cfg 也可以放置於其他地方，不一定要放在磁碟片中

<a name="ch1.1" />
1.1 Defining the Anaconda Kickstart System
==========================================

## 1.1.1 Introduction to Kickstart installations

每個 seciton 由 `%` 開頭，並用 `%end` 結尾

`%package` section 指定要所安裝的軟體

`@` 開頭的設定表示指定 `package group`，可指定安裝 RedHat 預先設定的軟體群組，例如 core、Web Server .... 等等

`@^`開頭表示指定 `enrironmental group`(group in package group)

其他客製化的需求可以放到 `%pre` & `%post` script 中


## 1.1.2 Kickstart configuration file commands

### Installation commands:

- `url`：用來指定安裝媒體的位置(FTP/HTTP/NFS .... etc)
```bash
# 使用 --url 指定來源
url --url="ftp://installserver.example.com/pub/RHEL7/dvd"
```

- `repo`：用來指定要額外安裝的 package repository
```bash
# --name 指定名稱，--baseurl 指定 repository 位置
repo --name="Custom Packages" --baseurl="ftp://repo.example.com/custom"
```

- `text`：預設以圖形模式顯示，用 text 可改成強制文字模式顯示

- `vnc`：設定 vnc 密碼
```bash
vnc --password=redhat
```

### Partition commands

- `clearpart`：安裝前清除硬碟上所有的 partition
```bash
# 指定清除 sda, sdb 兩顆硬碟
clearpart --all --drivers=sda,sdb --initlabel
```

- `part`：設定 partition 要如何分割
```bash
# 指定 partition 目錄、檔案型態、大小....等資訊
part /home --fstype=ext4 --label=homes --size=4096 --maxsize=8192 --grow
```

- `ignoredisk`：安裝時忽略特定硬碟
```bash
# 忽略 sdc
ignoredisk --drivers=sdc
```

- `bootloader`：指定安裝 bootloader 的地方
```bash
# 在 sda 上的 mbr 位置安裝 bootloader
bootloader --location=mbr --boot-driver=sda
```

- `volgroup`, `logvol`：建立 LVM volume groups & logical volumes
```bash
part pv.01 --size=8192
volgroup myvg pv.01
logvol / --vgname=myvg --fstype=xfs --size=2048 --name=rootvol -grow
logvol /var --vgname=myvg --fstype=xfs --size=4096 --name=varvol
```

- `zerombr`：清除原有的 mbr 設定

### Network commands

- `network`：設定網路
```bash
# 設定 eth0 為 DHCP
network --device=eth0 --bootproto=dhcp
```

- `firewall`：設定防火牆，指定開啟(or 關閉)特定服務
```bash
firewall --enabled --services=ssh,cups
```

- `lang`：語系設定
```bash
lang en_US.UTF-8
```

- `keyboard`：鍵盤設定
```bash
keyboard --vckeymap=us --xlayouts='us'
```

- `timezon`：設定時區、NTP server 以及是否使用 UTC
```bash
# 使用 UTC, 指定 NTP server, 並設定時區
timezon --utc --ntpservers=time.example.com Asia/Taipei
```

- `auth`：認證方式的設定
```bash
# 設定使用一般登入方式 & 加密強度
auth --useshadow --enablemd5 --passalgo=sha512
```

- `rootpw`：設定 root 密碼
```bash
# 也可以使用 "--uscrypted" 參數搭配加密過的密碼
rootpwd --plaintext redhat
```

- `selinux`：設定 SELinux 的狀態
```bash
selinux --enforcing
```

- `services`：設定各種 service 的預設狀態
```bash
services --disabled=network,iptables,ip6tables
```

- `group`, `user`：建立指定的群組與使用者
```bash
group --name=admins --gid=10001
user --name=jdoe --gecos="John Doe" --group=admins --password=changeme --plaintext
```

### Miscelaneous commands

- `logging`：定義安裝時的 log 如何處理，可指定 remote logging server
```bash
# 指定 log level & 要儲存的地方
logging --host=loghost.example.com --level=INFO
```

- `reboot`, `poweroff`, `halt`：系統安裝完執行的動作

> 一堆設定不用記，需要的時候再到 [RedHat 官方網站](https://access.redhat.com/documentation) 查詢(Getting Started -> Installation Guide)就好


產生 ks.cfg 方式：

1. 透過 `system-config-kickstart` 可以透過圖形介面產生 ks.cfg
> 必須在 /etc/yum.repos.d/rhel-dvd.repo 中要有 `rawhide` section 的設定，不然出來的圖形介面會沒有 package 可以選

2. 安裝好一台新的 RHEL，並找到 `/root/anaconda-ks.cfg` 檔案，拿出來用


--------------------------------------------------------------------------

<a name="ch1.2" />
1.2 Deploying a New Virtual System with Kickstart
=================================================

`ksvalidator` 可用來檢查 ks.cfg 的格式是否正確 (什麼都結果都沒有表示正確)

### Publish the Kickstart configuration file to Anaconda

ks.cfg 可以放在很多不同的地方：

- 可放在 FTP/HTTP/NFS ... 等服務上

- DHCP/TFTP server

- USB disk or CD-ROM

- Local disk

- 與 PXE server 結合


## 補充說明

### 遠端安裝 scenario 1

- client: private/puiblic IP

- remote server: public IP

在 remote server 端執行如下：(光碟開機 -> ESC 跳到 boot 選項)

```bash
# 光碟開機 -> ESC 跳到 boot 選項：
boot: linux vncpassword=redhat ip=172.25.0.11 netmask=255.255.255.0 gateway=172.25.0.254
```

> 以上 IP 組態設定會根據不同的地點而不同

### 遠端安裝 scenario 2

- client: public IP

- remote server: private IP

在 client 端下指令：

```bash
$ vncviewer --listen
```

在 remote server 端執行如下：(光碟開機 -> ESC 跳到 boot 選項)

```bash
boot: linux vnc vncconnect=172.25.254.250 ip=172.25.0.11 netmask=255.255.255.0 gateway=172.25.0.254 dns=8.8.8.8
```

按下 Enter 後，client 會自動跑一個 VNC console 出來，並顯示 remote server 的安裝畫面。

> 也可以通過 direct TCP port 5901 達成第一個方式
