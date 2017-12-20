---
layout: post
title:  "[RHCE] RH124 Chapter 8 Controlling Services and Daemons 學習筆記"
description: "此文章記錄學習 RHCE RH124 Chapter 8 Controlling Services and Daemons 所留下的內容"
date: 2016-04-19 04:30:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---

8.1 Identifying Automatically Started System Processes
======================================================

## 8.1.1 Introduction to systemd

systemd(是一個小型的 kernel) 跟開機流程很有關係

通常一個 service 是由一個或多個 daemon 提供。

多年前，Linux & UNIX 系統中 PID=1 是屬於一支稱為 `init`(載入 kernel 之後執行) 的 process；當 kernel 載入後執行。

在 RHEL 7 中，PID=1 的 process 已經變成 systemd：
- 有平行處理的能力，可提升系統開機的速度
- 有些先前需要的 daemon 會自動啟動
- 自動管理 service 相依性
- LCG (Linux control group)

> 現在的 `/usr/sbin/init` 已經改由 symbolic link 指到 systemd

systemd 用來管理各式各樣不同的型態的 object，稱為 **systemd unit**，可以用 `systemctl -t help` 查詢目前 systemd 可管理的 unit 有那些：

```bash
[student@server0 ~]$ systemctl -t help
Available unit types:
service
socket
target  # 取代原先 Run Level 的概念
device
mount
automount
snapshot
timer   # 自動排程的功能
.....
```
- `service`：用來表示系統服務
- `socket`：表示 IPC(inter-process communication) socket
- `timer`：自動排程工作是由 timer unit 來處理
- `target`：取代原先 Run Level 的概念
- `path`：通常用來以特定檔案有無變化為前提下，延遲服務發生，例如：列印服務(print pool 的概念)

### Service status

systemd service unit 有以下幾種狀態：

- **loaded**：unit 設定被處理後(套件剛裝好時，會處於此狀態)
- **active**：啟用狀態
- **inactive**：非啟用狀態
- **enabled**：開機時會自動啟動
- **disabled**：開機時不會自動啟動
- **static**：無法控管的狀態，必須由其他 unit 來自動的 enable

## 8.1.2 systemctl 使用方式

`systemctl` 是用來管理不同 systemd unit 的指令，使用方式大概如下：

列表相關的指令：

| Command | Description |
|---------|-------------|
| `sudo systemctl -l` | 檢視所有的 systemd unit 的狀態，但不包含狀態為 inactive 的(加上 `--all` or `-a` 可看到全部) |
| `sudo systemctl status sshd.service` | 檢查 ssh service unit 的狀態<p />(透過 `systemctl status name.type` 可以檢視 unit 狀態，`type` 可以不輸入，則預設為 **service**) |
| `sudo systemctl --type=service` | 檢視 type 屬於 service 的 systemd unit |
| `sudo systemctl is-active sshd.service` | 檢視 sshd service 是否為 active 狀態 |
| `sudo systemctl is-enabled sshd.service` | 檢視 sshd service 是否為 enabled 狀態 |
| `sudo systemctl list-unit-files --type=service` | 檢視 service type 所有的 unit 設定 |

----------------------------------------

8.2 Controlling System Services
===============================

## 8.2.1 Starting and stopping daemons on a running system

安裝 apache2，檢視狀態，並啟動服務：

``` bash
$ sudo yum -y install httpd

# inactive + disabled
$ sudo systemctl status httpd.service

# active(running) + disabled
$ sudo systemctl start httpd.service

# active(running) + enabled
$ sudo systemctl enable httpd.service
```

系統中有 exited & waiting 狀態的 system unit：

``` bash
# 觀察 active(exited) 狀態
$ sudo systemctl status network.service

# 觀察 active(waiting) 狀態
$ sudo systemctl status cups.path
```

``` bash
# 以下兩行指令相同
$ sudo service sshd restart
$ sudo /bin/systemctl restart sshd.service

# 以下兩行指令相同
$ sudo systemctl --type=service
$ sudo systemctl list-unit --type=service
```

> service reload 時，PID 不會換

### Unit dependencies

``` bash
# 檢視該 system unit 被那些 systen unit 所依賴
$ sudo systemctl list-dependencies graphical.target | grep target

# 檢視指定的 system unit 依賴那些其他的 system unit
$ sudo systemctl list-dependencies multi-user.target --reverse
```

### Masking services

- `disabled` 的 service 開機時不會自動啟動，但可以手動啟動

- `mask` 的 service 無法透過手動或自動的方式啟動(會在 `/etc/systemd/system` 目錄中產生指定 system unit 的 symbolic link 並指向 /dev/null)
