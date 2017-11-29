---
layout: post
title:  "[Linux KVM] VM Life Cycle 管理"
description: "This article introduces how to manage the lifecycle of virtual machines in KVM"
date: 2016-11-30 08:30:00
published: true
comments: true
categories: [KVM]
tags: [Linux, KVM, SPICE, VNC]
---

此篇文章將會介紹 VM 生命周期管理、Qemu Guest Agent、Virtual Video Card、Graphic Server ... 等主題。

VM Lifecycle
============

在 KVM 中的 virtual machine 共會有以下幾種狀態：

- **Undefined**：未定義

- **Defined / Shutoff**：已定義，libvirtd 已經知道有此 VM 存在，但狀態為關機中(Stopped or Shutdown)

- **Running**：執行中

- **Shutdown**：關機中

- **Paused**：暫停中，VM 記憶體中的資料暫時被保留，並且可以在 guest OS 無法知悉的狀況下回復

- **Saved**：VM 處於完全暫停的狀態，記憶體中的資料被存到一般檔案中，並存在於 persistent storage (可回復到進行 save 時的狀態)

- **Idle**：等待 I/O，或是因為沒有工作需要進行而休眠中

- **Crashed**：可能因為 QEMU process 被強制移除 or core dump 所造成的 VM 損毀

- **Dying**：在 shutdown 的過程中失敗所產生的況狀

- **Pmsuspended**：透過 guest OS 中的電源管理功能進行 suspend 後進入的狀態

有了上面概念後，可以透過 virsh 來檢視目前 domain(virtual machine) 的狀態：


--------------------------------------------------------------------


檢視 VM
=======

```bash
# 顯示所有 VM (包含關機中的)
$ virsh -c qemu+ssh://root@10.20.190.3/system list --all

# 顯示特定狀態的 VM
$ virsh -c qemu+ssh://root@10.20.190.3/system list --state-[running|paused|shutoff|]

# 顯示(未)包含 snapshot 的 VM
$ virsh -c qemu+ssh://root@10.20.190.3/system list --[with|without]-snapshot

# 顯示(未)包含 managed save 狀態的 VM
$ virsh -c qemu+ssh://root@10.20.190.3/system list --[with|without]-managed-save

# 僅顯示 uuid or name
$ virsh -c qemu+ssh://root@10.20.190.3/system list --[uuid|name]
```

> 還有許多其他參數可用，使用者可透過 `virsh help` or `virsh help list` 來查詢更多使用方式


--------------------------------------------------------------------

操作 VM
=======

了解 VM 有這麼多狀態之後，自然就會有相對應的操作了，virsh 提供以下幾項對 VM 的操作：

- **start**：啟動 VM

- **shutdown**：關閉 VM (正常關機程序)

- **reboot**：重新啟動 VM

- **reset**：與 power cycle 相同效果

- **save**：將 VM 狀態儲存到檔案中，並關閉 VM

- **restore**：從指定的檔案將 VM 狀態回復為執行中

- **suspend**：暫停 VM 運作

- **resume**：回復 VM 運作

- **destroy**：直接刪除 QEMU process (類似直接拔掉電腦電源線的效果)

- **create**：使用指定的 XML 建立 VM，並啟動 VM

- **define**：使用指定的 XML 建立 VM，但不啟動 VM

- **undefine**：將 VM 從 libvird 的控制中移除


--------------------------------------------------------------------


QEMU guest agent
================

在安裝 virtualbox or VMware 的 VM 時，安裝結束之後都會詢問要不要額外安裝 agent 在 guest VM 中，透過這個 agent，hypervisor 可以更有效率的管理每一個 VM；同樣的在 KVM 中，也是有相同的作法，稱為 **QEMU guest agent**。

**QEMU guest agent** 是一個裝在 guest VM 中的套件，會以 service 的形式存在於背景，接著 service 就變成了 hypervisor & guest OS 之間溝通的橋樑(channel)，hypervisor 會透過這個 channle 取得 VM 的資訊，以及對 VM 進行後續更多的操作；而兩者相互通訊的協定稱為 **Qemu Machine Protocol(QMP)**。

其中 Hypervisor 與 guest agent 是透過一個名稱為 `org.qemu.guest_agent.0` 的 virtio-serial channel 或是 isa-serial channel 來處理；而在 Hypervisor 這一端，相對應處理這些資訊交換的 socket file 則是存在於 `/var/lib/libvirt/qemu/channel/target` 目錄中，且同一個 socket file 可以同時被多個 VM 所共享，因此不會產生太多檔案。

若 guest VM 為 Linux，可安裝名稱為 `qemu-guest-agent` 的套件，若是 windows，則可以參考以下連結：

- [WindowsGuestDrivers/Download Drivers - KVM](http://www.linux-kvm.org/page/WindowsGuestDrivers/Download_Drivers)

- [Windows Virtio Drivers - FedoraProject](https://fedoraproject.org/wiki/Windows_Virtio_Drivers)

除了安裝 `qemu-guest-agent` 套件之外，還要確認 `qemu-guest-agent` service 是否有正確啟動：

```bash
$ systemctl status qemu-guest-agent
● qemu-guest-agent.service - QEMU Guest Agent
   Loaded: loaded (/usr/lib/systemd/system/qemu-guest-agent.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2016-11-24 22:25:07 EST; 35min ago
   ......
```

確認服務沒有問題，我們就可以使用類似以下的命令在 KVM host 端取得 guest VM 的相關資訊：

```bash
$ virsh qemu-agent-command <GUEST_VM_NAME> '{"execute": "guest-info"}' --pretty
{
    "return": {
        "version": "2.3.0",
        "supported_commands": [
            {
                "enabled": true,
                "name": "guest-get-memory-block-info",
                "success-response": true
            },
.......
```

> QMP 資料通訊使用 JSON format

--------------------------------------------------------------------

Virtual Video cards & graphics
==============================

為了可以"看見" VM 的運作狀態，QEMU 需要提供兩個元素來達成這件任務：

1. **virtual video card**：讓每個 VM 都擁有一張虛擬的顯示卡

2. 從遠端存取 VM 虛擬顯示卡的方式 or 協定


## Virtual Video Card

顯示卡的用途在於顯示圖形資料到顯示設備上，而虛擬顯示卡同樣也是為了這個目的而存在的。

然而在虛擬環境中當然沒有實體顯示卡，因此 QEMU 支援模擬以下幾種顯示卡：

- **Cirrus**：libvirt 預設的顯示卡，可模擬 **Cirrus Logic GD5446** 顯示卡，Windows 95 之後的作業系統都支援這張顯示卡

- **VGA**：搭配 Bochs VBE extensions 的標準顯示卡，Windows XP 以後的作業系統可以支援(可設定 >= 1280x1024x16 解析度 & 大小的畫質)

- **VMVGA**：在 VGA 再更高階的虛擬顯示卡

- **QXL**：半虛擬化顯示卡，搭配在 VM 內安裝 QXL guest driver，可得到很不錯的顯示效果，而且是搭配 **spice** protocol 的最佳選擇

在安裝 VM 時，libvirt 就會根據安裝的 OS，協助選擇一個最合適的顯示卡，通常安裝最近發佈的 OS，都會直接選配 QXL；若是 Windows 或是比較舊版的 Linux，可能就會使用 Cirrus。


## Graphics

當 VM 有了顯示卡之後，接著需要一個可以存取圖形訊號的方法，在 KVM 中是透過 **graphic server** 的方式，目前提供了 **VNC** & **SPICE* 兩種 graphic server。

當 VM 確定好顯示卡之後，QEMU 就會啟動相對應的 Spice or VNC server，並與 VM 的顯示卡進行連接，以便讓外部的 client 可以使用圖形化的方式存取 VM。

### (1) VNC

要為 VM 設定一個 VNC graphic server，可在 XML 定義檔中的 `<device>` section 內部加入類似以下的定義：(IP & port 都可以自己定義)

```xml
<graphics type='vnc' port='-1' autoport='yes' listen='192.168.122.1'>
    <listen type='address' address='192.168.122.1'/>
</graphics>
```

### (2) SPICE

SPICE(Simple Protocol for Independent Computing Environment) 僅有在 Linux 上支援，有以下特色：

1. 可以提供雙向的 audio

2. 高效率的 2D 圖像繪製能力

3. 可利用到 client 端的顯示卡的能力

4. 支援加密、資料壓縮

5. 支援透過網路的 USB passthrough

因此若要規劃把 KVM 用在 VDI 的應用上，使用 `QXL` + `SPICE` 目前是最好的組合。

要為 VM 設定一個 SPICE graphic server，可在 XML 定義檔中的 `<device>` section 內部加入類似以下的定義：(IP & port 都可以自己定義)

```xml
<graphics type='spice' autoport='yes' listen='0.0.0.0'>
    <listen type='address' address='0.0.0.0'/>
</graphics>
```

--------------------------------------------------------------------


References
==========

- [Virtualization Administration Guide > Running the QEMU Guest Agent on a Windows Guest](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Administration_Guide/sect-QEMU_Guest_Agent-Running_the_QEMU_guest_agent_on_a_Windows_guest.html)

- [WindowsGuestDrivers/Download Drivers - KVM](http://www.linux-kvm.org/page/WindowsGuestDrivers/Download_Drivers)

- [Windows Virtio Drivers - FedoraProject](https://fedoraproject.org/wiki/Windows_Virtio_Drivers)
