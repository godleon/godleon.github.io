---
layout: post
title:  "[Linux KVM] 使用 Linux KVM 啟用第一個 virtual machine"
description: "This article introduces how to initialize a kvm-enhanced virtual machine by usung QEMU/KVM"
date: 2016-07-27 06:00:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM]
---

介紹如何使用 QEMU/KVM 在 Linux 上啟用第一個 virtual machine

前言
====

## KVM

KVM 屬於全虛擬化(Full Virtualization) 的技術，因此在上面運行的 OS 不需要經過任何修改。

原本因為 Full Virtualization 效能應該是很差的，但因為硬體虛擬化的支援(例如：[Intel VT-d](http://stenlyho.blogspot.tw/2009/01/vt-xvt-d-intel.html)，因此大幅提升了 KVM 的效能；此外 KVM 與 QEMU(負責周邊設備的模擬) 的搭配，提供了使用者在 CPU、Memory、Storage、Network、Display 上都有完全相同的虛擬化體驗。

## Linux kernel

關於 Linux kernel 上，建議使用目前較新的 kernel，在以下的環境中，會選用安裝 CentOS 7 作為 host machine，搭配 3.10 版的 Linux kernel 做為測試環境之用。

## 環境說明

### 硬體

CPU：Intel(R) Xeon(R) CPU E5-2660 v3 @ 2.60GHz

### 軟體

- OS: CentOS 7

- Linux Kernel: 3.10

- qemu-kvm: 1.5.3 (預設開啟 KVM 加速)

- qemu-img: 1.5.3

---------------------------------------------------------------------

前置環境設定
===========

## 安裝套件

首先要安裝 KVM、QEMU、libvirtd 相關套件 & 啟動 libvirtd service：

```bash
# KVM v2.3 需要使用此 repository
$ bash -c "echo '[kvm-common]
name=Latest KVM rpms
baseurl=http://mirror.centos.org/centos-7/7/virt/x86_64/kvm-common/
enabled=1
gpgcheck=0' > /etc/yum.repos.d/kvm.repo"

# 安裝 KVM 1.5
$ yum install -y qemu-kvm \
    qemu-img \
    qemu-system-x86 \
    libvirt \
    virt-install \
    libvirt-python \
    virt-manager \
    python-virtinst \
    libvirt-client \
    bridge-utils \
    net-tools \
    libguestfs-tools-c \
    iptables-services

# 也可使用 groupinstall 來批次安裝
$ yum groupinstall "virtualization" -y

# 若要安裝 KVM 2.3 可使用下面指令
$ yum install -y qemu-kvm-ev \
    qemu-img-ev \
    qemu-system-x86 \
    libvirt \
    virt-install \
    libvirt-python \
    virt-manager \
    python-virtinst \
    libvirt-client \
    bridge-utils \
    net-tools \
    libguestfs-tools-c \
    iptables-services
```

也可以順便設定 [nested virtualization](http://www.linux-kvm.org/images/3/33/02x03-NestedVirtualization.pdf)，可以在虛擬環境中再虛擬一層而不會有太多的 performance lose：(此步驟可以略過)

```bash
# 目前 nested virtualization 是關閉的 
$ cat /sys/module/kvm_intel/parameters/nested
N

# 開啟 nested virtualization
$ sudo rmmod kvm-intel
$ sudo sh -c "echo 'options kvm-intel nested=y' >> /etc/modprobe.d/dist.conf"
$ sudo modprobe kvm-intel

# 開啟 nested virtualization 成功
$ cat /sys/module/kvm_intel/parameters/nested
Y
```


## 防火牆設定

停用預設的 **firewalld.service**，並啟用 **iptables.service**：

```bash
# 關閉 firewalld
$ systemctl stop firewalld.service
$ systemctl disable firewalld.service

# 啟用 iptables
$ systemctl stop iptables.service
$ systemctl disable iptables.service
```

改用傳統的 itpables 來進行防火牆設定，並使用以下 script 建立防火牆：

```bash
#!/bin/bash

# global variables
IIF="ens1f0"

# 防止 sync flooding 攻擊(開啟 tcp sync cookie)
echo 1 > /proc/sys/net/ipv4/tcp_syncookies

iptables -t filter -F

# 設定 connection track
iptables -t filter -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# 避免 INVALID 封包被其他服務所接收
iptables -t filter -A INPUT -m state --state INVALID -j DROP

iptables -t filter -A INPUT -p tcp --dport 22 -j ACCEPT
# 提供 vnc access
iptables -t filter -A INPUT -p tcp --dport 5900:5910 -j ACCEPT

# 用來取代 chain default policy
iptables -t filter -A INPUT -i ${IIF} -j DROP
```

> 上面的套件安裝 & 防火牆設定完成後，要將 KVM host 重新開啟，並啟動 **<font color='red'>libvirtd.service</font>**

---------------------------------------------------------------------

驗證環境
=======

安裝好 QEMU/KVM 相關套件後，我們可以來檢查目前的環境是否可以正確的運行虛擬化功能：

```bash
$ virt-host-validate
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking for device /dev/kvm                                         : PASS
  QEMU: Checking for device /dev/vhost-net                                   : PASS
  QEMU: Checking for device /dev/net/tun                                     : PASS
   LXC: Checking for Linux >= 2.6.26                                         : PASS
```

```bash
$ virsh nodeinfo
CPU model:           x86_64
CPU(s):              48
CPU frequency:       1200 MHz
CPU socket(s):       1
Core(s) per socket:  12
Thread(s) per core:  2
NUMA cell(s):        2
Memory size:         263930636 KiB

$ virsh domcapabilities
<domainCapabilities>
  <path>/usr/bin/qemu-system-x86_64</path>
  <domain>qemu</domain>
  <machine>pc-i440fx-2.0</machine>
  <arch>x86_64</arch>
  <vcpu max='255'/>
  <os supported='yes'>
    <loader supported='yes'>
      <enum name='type'>
        <value>rom</value>
        <value>pflash</value>
      </enum>
      <enum name='readonly'>
        <value>yes</value>
        <value>no</value>
      </enum>
    </loader>
  </os>
  <devices>
    <disk supported='yes'>
      <enum name='diskDevice'>
        <value>disk</value>
        <value>cdrom</value>
        <value>floppy</value>
        <value>lun</value>
      </enum>
      <enum name='bus'>
        <value>ide</value>
        <value>fdc</value>
        <value>scsi</value>
        <value>virtio</value>
        <value>usb</value>
      </enum>
    </disk>
    <hostdev supported='yes'>
      <enum name='mode'>
        <value>subsystem</value>
      </enum>
      <enum name='startupPolicy'>
        <value>default</value>
        <value>mandatory</value>
        <value>requisite</value>
        <value>optional</value>
      </enum>
      <enum name='subsysType'>
        <value>usb</value>
        <value>pci</value>
        <value>scsi</value>
      </enum>
      <enum name='capsType'/>
      <enum name='pciBackend'>
        <value>default</value>
        <value>vfio</value>
      </enum>
    </hostdev>
  </devices>
</domainCapabilities>
```


---------------------------------------------------------------------

啟動第一個 virtual machine
=========================

由於 RedHat 建議使用 virsh，因此 **<font color='red'>qemu-kvm</font>** 就不存在於預設路徑中，用以下指令把它找出來：

```bash
# 將 qemu-kvm 以 symlink 的形式複製到 $PATH
# qemu-kvm 指令已經預設啟用 KVM 支援
$ ln -sf /usr/libexec/qemu-kvm /usr/bin/kvm

# 製作一個 size = 8GB 的 raw image 作為 virtual machine disk
$ dd if=/dev/zero of=/kvm/storage/vm_disks/ubnutu1604.img bs=1M count=8192

# 目前系統中存在的 iso
$ ls /kvm/os_images
ubuntu-16.04.1-server-amd64.iso

# 啟動 VM (以光碟開機，安裝作業系統)
# -smp 4 => vCPU=4
# -m 2048 => RAM=2048MB
# -vnc 0.0.0.0:5 => 將 vnc 開在第五個 console，因此透過 vnc viewer 連線要使用 "ip:5" 來進行連線
# -boot order=cd => 開機順序為 cdrom > hdd
# -hda /kvm/storage/vm_disks/ubnutu1604.img => 指定 hdd raw image
# -cdrom /kvm/os_images/ubuntu-16.04.1-server-amd64.iso => 指定開機光碟 iso
$ kvm -smp 4 -m 2048 \
  -vnc 0.0.0.0:5 -boot order=cd \
  -hda /kvm/storage/vm_disks/ubnutu1604.img \
  -cdrom /kvm/os_images/ubuntu-16.04.1-server-amd64.iso
```

接著可以使用 [TigerVNC](http://tigervnc.org/) or [VNC® Viewer for Google Chrome](https://chrome.google.com/webstore/detail/vnc%C2%AE-viewer-for-google-ch/iabmpiboiopbgfabjmgeedhcmjenhbla) 來進行連線，使用的的連線位置為 `server_ip:5`；連線進入後，就可以按照一般程序進行 OS 的安裝。

安裝完成後，`/kvm/storage/vm_disks/ubnutu1604.img` 將會是一個已經安裝好 OS 的硬碟 image 檔案，我們可以透過以下指令使用此 image 檔案來啟動系統：

```bash
# -smp 4 => vCPU=4
# -m 2048 => RAM=2048MB
# -vnc 0.0.0.0:5 => 將 vnc 開在第五個 console，因此透過 vnc viewer 連線要使用 "ip:5" 來進行連線
$ kvm -smp 4 -m 2048 -vnc 0.0.0.0:5 -hda /kvm/storage/vm_disks/ubnutu1604.img
```


使用 VNC viewer 連到 virtual machine 之後，可使用 `Ctrl + Alt + 2` 切到 QEMU monitor，輸入 `kvm info` 就可以檢視目前 KVM 是否被使用，或是完全在 QEMU 模擬下產生：

![QEMU Monitor](https://lh3.googleusercontent.com/Or8AH3hxAJbdXsIMADxmVkvUKlFYe_-DwTpSuw878fpP3bDAqyLv_ql_7W_HIrLYGHqc1hha7aecKMM6lytj2Wkv-NEyoXsZPuHE5RKa9mp_6apKEoPn7h-tp2DSjLcHaj72ByMefPKXRFKFYCLmYdbhsax0Ro8A-UCjSweOuB03zEL44VM7YbkxNE85vTmFv-JMUUERlX3CAdDotSWpvzl-ztgHzoU2E0iqwgLawhHAhn1JOX19Mn0ib3J0vxqZyLI6CNqaXEIc-7v5QNhAmAysEdWd3AMVKqkPVI41v8FDiUam2G_MDUkWOsAW4aQjnwrAbiw3-Ljpi72gjsM_iJWnsLF5nCuREGQxWC_LOrhRo7-AKLgU_XmzJuobqLRi6LoMzy_BKva5jI7nSgmgAfXDDTWpsXyGoKSgexpg_J7SRuLVZR1iZZ8HCZ9FZpre9qNIdr6xsvnH1NoUobJ73IKMbKn4YmGD0Z9Odi51bpWSh9eV1kOyCMFnF7Q5KZoPk-9lZwChK0gVUj6eObbJDKhlrFTpSvO7vYGJOmIBtfgCfKB9tAZ1-fpDSWxv8tGP38OQmKirl5oh0ozkGGQ-Z9QAr0mC31U=w642-h143-no)

> 從上圖來看，可看出關鍵字 `kvm support: enabled`，表示 KVM 加速是開啟的

> 由於我們在上面所產生的 kvm 指令是來自於 qemu-kvm，因此已經預設啟用 KVM 加速 (若單純使用 QEMU，可搭配 `--enable-kvm` 參數來啟用 KVM 加速)

---------------------------------------------------------------------

References
==========

## Installation

- [玩具烏托邦: 五分鐘開始玩 qemu-kvm 虛擬機](http://newtoypia.blogspot.tw/2015/02/qemu-kvm.html)

- [在 CentOS 6.2 上安装和配置 KVM @vpsee.com](http://www.vpsee.com/2012/04/install-kvm-on-centos-6-2/)

- [使用 libvirt 與 qemu-kvm 開啓 VM (內含 libvirt sample XML for KVM ) « OT Coding Note](http://ot-note.logdown.com/posts/64644/launch-a-vm-with-qemu-kvm)


## VNC

- [KVM/QEMU 設置 VNC 連線密碼 « Jamyy's Weblog](http://jamyy.us.to/blog/2011/10/3365.html)
