---
layout: post
title:  "[RHCE7] RH134 Chapter 13. Controlling and Troubleshooting the RHEL Boot Process 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 13. Controlling and Troubleshooting the RHEL Boot Process 留下的內容"
date: 2016-05-14 04:45:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

13.1 The RHEL Boot Process
==========================

## 13.1.1 The RHCE 7 boot process

1. 電腦開機從 BIOS 開始

2. 進行 POST 檢查

3. stage-1：載入 Boot Loader 並開始執行

4. stage-2：讀取 `/boot/grub2/grub.cfg` 並執行

5. `grub.cfg` 解壓縮 `initramfs-xxxx`(很小的 gzip + cpio 的 Linux，若要額外加上 HW driver 就要包到這裡)，並載入記憶體中

6. `grub.cfg` 執行 `vmlinuz-xxx`(可執行的 Linux kernel 映像檔)，將記憶體的內容掛載到 `/`

7. 執行 `/init` (已經變成 soft link) 指向其他地方

8. `init` 會 `mount -o ro [HD root partition] /sysroot`

9. `chroot /sysroot`

10. 執行 `/lib/systemd/systemd`

boot load 解壓縮 initramfs-xxx 到記憶體中

第 8 個步驟 `-o ro` 的原因是開機時會作 fsck，因為無法 unmount 根目錄，因此只能用 read only 的方式

## 13.1.3 Selecting a systemd target

**systemd target** 是由一組 systemd unit 所組成，可以讓 user 進入到某一個特定狀態(例如：圖形介面)，比較常用的包含了：

- graphical.target：圖形 & 文字介面

- multi-user.target：文字介面

- rescue.target：初始化完成，會提示輸入 root 密碼

- emergency.target：initramfs 掛載 / 完成，只有 read only 權限，且會提示輸入 root 密碼

```bash
# 啟動 graphic.target 需要由那些 systemd target unit 所組成
$ systemctl list-dependencies graphical.target | grep target
graphical.target
└─multi-user.target
  ├─basic.target
  │ ├─paths.target
  │ ├─slices.target
  │ ├─sockets.target
  │ ├─sysinit.target
  │ │ ├─cryptsetup.target
  │ │ ├─local-fs.target
  │ │ └─swap.target
  │ └─timers.target
  ├─getty.target
  ├─nfs.target
  └─remote-fs.target

# 列出系統中所有的 systemd target unit
$ systemctl list-units --type=target --all
UNIT                   LOAD   ACTIVE   SUB    DESCRIPTION
basic.target           loaded active   active Basic System
cryptsetup.target      loaded active   active Encrypted Volumes
emergency.target       loaded inactive dead   Emergency Mode
.......
umount.target          loaded inactive dead   Unmount All Filesystems

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.

27 loaded units listed.
To show all installed unit files use 'systemctl list-unit-files'.

# 線上直接切換不同的 target (但這會停掉指定 target 中不需要的服務)
$ sudo systemctl isolate multi-user.target

# 將 graphical.target 設定為預設值
$ $ sudo systemctl set-default graphical.target
rm '/etc/systemd/system/default.target'
ln -s '/usr/lib/systemd/system/graphical.target' '/etc/systemd/system/default.target'
```

> 也可以在開機時，修改開機設定，在 `linux16` 那一行最後面，加上 `systemd.unit=graphical.target`，也可以進入圖形模式

-------------------------------------------------------

13.2 Repairing Common Boot Issues
=================================

##  13.2.1 Recovering the root password

> 透過修改開機設定，加上 `rd.break` 進入可救援的模式來修改 root 密碼，但這樣會破壞 SELinux security context

1. 修改 linux16 開頭的設定，從最後面移除設定直到 `ro` 前，並在該行最後加上 `rd.break`，按下 `Ctrl+x` 使用此設定開機 (開機完成後會位於 initramfs 的 **/**，並包含了 **real-only** 的 **/sysroot**)

2. 重新掛載 /sysroot 目錄為可讀寫：`mount -o remount,rw /sysroot`

3. 切換根目錄到 /sysroot 上：`chroot /sysroot`

4. 修改 root 密碼：`passwd root`

5. 產生 `/.autorelabel` 檔案，讓開機過程可以重新 relabel：`touch /.autorelabel`

6. 連續輸入兩次 `exit`，回到開機程序

`/.autorelabel`：此檔案用途會讓 SELinux 進行 relabel 的動作，將 security context 恢復到正確的狀態

### Using journalctl

預設 log 儲存在記憶體中，只要 `sudo mkdir -p -m2775 /var/log/journal` 就可以將 systemd journal 放到硬碟中

顯示上一次開機的過程中，Level ERROR 的 log：`sudo journalctl -b -1 -p err`

### Diagnose and repair systemd boot issues

`sudo systemctl enable debug-shell.service`：重開機後可以透過 Ctrl + Alt + F9 得到一個 root shell

開機流程中，得到 shell 的順序是：

1. rd.break

2. emergency.target

3. rescue.target

> 所以其實使用 rd.break 就可以解決所有開機的相關問題

`systemctl list-job`：可用來觀察開機時的 stuck job

-------------------------------------------------------

13.3 Repairing File System Issues at Boot
=========================================

- 每次開機 fsck 都會嘗試自動修復，若無法自動處理則會進入 emergency shell

- 設備不存在，systemd 會等一段時間，若依然沒有就會進入 emergency shell 給管理者除錯

- mount point 不存在，systemd 會嘗試自動建立，否則就進入 emergency shell

- /etc/fstab 錯誤，直接進入 emergency shell

-------------------------------------------------------

13.4 Repairing Boot Loader Issues
=================================

- GRUB2 同時支援 BIOS & UEFI

- 主要設定檔位於 `/boot/grub2/grub.cfg`；但若是要設定 UEFI 時，grub.conf 就要到 `/boot/efi` 找

- 在 linux16 那一行，再最後加上 `net.ifnames=0` 後，網卡就會以傳統的方式命名(eth0, eth1 ... etc)

- `set root` 是描述開機相關的檔案(vmlinuz-xxx / initramfs-xxx ... etc)存在於哪個 partition

- linux16 設定中，`root=UUID=xxxx` 則是用來指定 root partition
