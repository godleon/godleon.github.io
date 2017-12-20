---
layout: post
title:  "[RHCE7] RH254 Chapter 1 Controlling Services and Daemons Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 1 Controlling Services and Daemons Learning Notes 留下的內容"
date: 2016-05-21 16:25:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH254]
---

1.1 Controlling Services with systemctl
=======================================

## 1.1.1 Introduction to systemd

RHEL7 已經使用 `systemd` 取代原有的 `init` & `xinetd`，因此目前 ID=1 的 process 為 systemd，主要加入了下列新的特性：

1. 平行處理，可加速系統開機

2. on-demand 啟動服務的能力

3. service 相依性的自動處理，避免造成系統長時間等待特定服務(例如：網路不通時不會把 network service 啟動)

4. 可透過 Linux control group 追蹤相關的 process 的狀態

> 現在大多 service 的相關設定都被移到 `/etc/sysconfig` 目錄中了，只有某些 legacy services 才依然是以 shell-based 的形式存在

現在 RHEL7 中使用 `systemctl` 管理各式各樣的 systemd 物件，這些物件統稱為 `units`，常看到的 units 會有：

1. `Service units`(**.service**)：這就是系統服務了，通常會以 daemon 的形式存在

2. `Socket Units`(**.socket**)：指的是 IPC socket，通常在有新的連線與 service 產生時，socket unit 會被傳給 daemon 處理

3. `Path units`(**.path**)：用來 delay 特定 service，直到特定的檔案有所變化，例如：列印服務

> 若 system unit 狀態為 `static`，表示無法被 enable，但可以被 `enabled unit` 自動啟用


## 1.1.2 Starting and stopping daemons on a running system

`restart` 會改變 service 的 process ID；但 `reload` 則會維持 service 原有的 process ID


## 其他：systemctl 特別的指令

1. `sudo systemctl --failed --type=service`：列出狀態為 failed 的 service

2. `sudo systemctl list-dependencies [UNIT]`：列出指定 system unit 的相依關係

3. `sudo systemctl list-dependencies --reverse [UNIT]`：列出指定 system unit 啟動所需要預先啟動的其他 system unit

----------------------------------------------------

1.2 Controlling the Boot Process
================================

## 1.2.1 Selecting a systemd target

以下列出幾個重要的 systemd target：

- `graphical.target`：就是一般的圖形模式，也包含了文字介面

- `multi-user.target`：純文字模式

- `rescue.target`：要求用 root 身分登入，基本系統服務已經初始化完成

- `emergency.target`：要求用 root 身分登入，initramfs 已經執行完畢，並把 system root 以 read-only 的方式掛載

每個 target 之間會有某種程度的相依關係，例如：

```bash
# 可看出 graphical.target unit 與那些 target unit 有相依
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

# 列出所有 target unit
$ systemctl list-units --type=target --all
  UNIT                   LOAD   ACTIVE   SUB    DESCRIPTION
  basic.target           loaded active   active Basic System
  cryptsetup.target      loaded active   active Encrypted Volumes
  emergency.target       loaded inactive dead   Emergency Mode
  final.target           loaded inactive dead   Final Step
  getty.target           loaded active   active Login Prompts
  graphical.target       loaded active   active Graphical Interface
  local-fs-pre.target    loaded active   active Local File Systems (Pre)
  local-fs.target        loaded active   active Local File Systems
  multi-user.target      loaded active   active Multi-User System
  network-online.target  loaded inactive dead   Network is Online
  ......
```

透過 `systemctl isolate [TARGET_UNIT]` 可以馬上變換不同的 target unit，假設目前在圖形模式下，執行指令 `sudo systemctl isolate multi-user.target` 可以切換到文字模式

### Setting a default target

`default.target` 為開機時所啟動的 target unit，這是個 symbolic link，指向真正的 target unit

```bash
# 目前指向 graphical.target，因此開機會是圖形介面
$ ls -al /etc/systemd/system/default.target
lrwxrwxrwx. 1 root root 40 Jul 11  2014 /etc/systemd/system/default.target -> /usr/lib/systemd/system/graphical.target

# 也可以用另外一個指令取得 default target unit
$ systemctl get-default
graphical.target
```

當然也有指令可以設定 default target unit：

```bash
$ sudo systemctl set-default multi-user.target
rm '/etc/systemd/system/default.target'
ln -s '/usr/lib/systemd/system/multi-user.target' '/etc/systemd/system/default.target'

$ systemctl get-default
multi-user.target
``

> 開機時編輯開機設定，在 `linux16` 那一行的最後面加上 `systemd.unit=[TARGET_UNIT]` 也可以直接進入所指定的 target unit

----------------------------------------------------

Lab: Controlling Services and Daemons
=====================================

## 目標

- 設定開機時同時支援文字介面登入 & 圖形介面登入

- 設定 `rsyslog` service 開機時啟動

## 實作步驟

```bash
# 檢查目前的 target unit
$ systemctl get-default
multi-user.target

# 設定 target unit 為 graphical.target (同時支援文字介面登入 & 圖形介面登入)
[student@server0 ~]$ sudo systemctl set-default graphical.target
rm '/etc/systemd/system/default.target'
ln -s '/usr/lib/systemd/system/graphical.target' '/etc/systemd/system/default.target'

[student@server0 ~]$ systemctl get-default
graphical.target

[student@server0 ~]$ systemctl status rsyslog
rsyslog.service - System Logging Service
   Loaded: loaded (/usr/lib/systemd/system/rsyslog.service; disabled)
   Active: inactive (dead)

# 設定 rsyslog.service 開機啟動
[student@server0 ~]$ sudo systemctl enable rsyslog.service
ln -s '/usr/lib/systemd/system/rsyslog.service' '/etc/systemd/system/multi-user.target.wants/rsyslog.service'
[student@server0 ~]$ sudo systemctl restart rsyslog.service
[student@server0 ~]$ systemctl status rsyslog
rsyslog.service - System Logging Service
   Loaded: loaded (/usr/lib/systemd/system/rsyslog.service; enabled)
   Active: active (running) since Sat 2016-05-21 17:21:24 JST; 2s ago
 Main PID: 31841 (rsyslogd)
   CGroup: /system.slice/rsyslog.service
           └─31841 /usr/sbin/rsyslogd -n
```
