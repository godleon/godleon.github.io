---
layout: post
title:  "[RHCE] RH124 Chapter 10 Analyzing and Storing Logs 學習筆記"
description: "此文章記錄學習 RHCE RH124 Chapter 10 Analyzing and Storing Logs 留下的內容"
date: 2016-04-24 21:30:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---


10.1 System Log Architecture
============================

## 10.1.1 System logging
--------------

RHEL 7 有兩支 daemon 管理 log：
- systemd-journald
- rsyslog
  - 紀錄 message type(or facility) & priority 在 `/var/log` 內
  - facility.severity (facility 很多種，severity 則是標準)

### systemd-journald

systemd-journald daemon 蒐集以下的訊息:(並且會把資料寫進 structured database)

- kernel 相關的訊息

- 開機流程中早期的訊息

- daemon 啟動時的標準輸出 & 錯誤訊息

- syslog (會被 forward 給 rsyslog 處理)

### rsyslog

rsyslog 會把從 syslog 來的資料放到 **/var/log** 中，除了 **/var/log/secure**、**/var/log/maillog**、**/var/log/cron**、**/var/log/boot.log** 四類訊息，其他訊息都會放在 **/var/log/messages** 中

--------------------------------------

10.2 Reviewing Syslog Files
===========================

rsyslogd 使用 `facility`(type) & `priority`(severity) 來決定如何處理 log message，透過設定檔 `/etc/rsyslog.config` & `/etc/rsyslog.d/\*.conf` 定義處理方式

## 10.2.2 Sample rule section of rsyslog.config

左邊的部份指定哪些 facility.severity 要被記錄，右邊的部份則是指定 log message 要存到哪個檔案

``` bash
# 所有 severity INFO 以上的訊息都送往 /var/log/message
# 但 facility mail / authpriv / cron 除外
# boot log message 也會送
*.info;mail.none;authpriv.none;cron.none    /var/log/messages

# kernel facility 的 log message 會往 /dev/console 送
kernel.*    /dev/console

# authpriv facility 相關的 log message 都會往 /var/log/secure 送
authpriv.*  /var/log/secure

# mail facility 的訊息會先存在於記憶體中，一段時間後詞才會存到檔案中
mail.*      -/var/log/maillog

# severity Emergency 的 log message 會送到目前所有登入使用者的 terminal 上
*.emerg     :omusrmsg:*

# 開機相關的訊息送往 /var/log/boot.log
local7.*    /var/log/boot.log
```

設定完成後，可透過 `logger -p [facility].[severity] "log message"` 來測試，例如：

``` bash
$ logger -p local7.emerg "boot emergency log message test"
```

> 可安裝 **<font color='red'>rsyslog-doc</font>** 套件取得 rsyslogd 相關的 man page，很有幫助

### 10.2.3 Log file rotation

透過 `/etc/logrotate.conf` 進行設定的修改

`/etc/cron.daily` 目錄中有一支 logrotate 的 shell script 作為每天執行的工作

### 10.2.6 Send a syslog message with logger

若是有修改過 rsyslog 的設定後，可用 `logger` 程式手動發送 log 進行驗證，是否 log 有正確的被記錄：

```bash
[student@server0 ~]$ sudo tail -3 /var/log/boot.log
[  OK  ] Started GNOME Display Manager.
[  OK  ] Started LSB: Start the ipr dump daemon.
[  OK  ] Started Dynamic System Tuning Daemon.

# 手動產生一個 local7.notice 的 log message
[student@server0 ~]$ logger -p local7.notice "Log entry created on server0"

[student@server0 ~]$ tail -3 /var/log/boot.log
[  OK  ] Started LSB: Start the ipr dump daemon.
[  OK  ] Started Dynamic System Tuning Daemon.
Apr 23 23:26:04 server0 student: Log entry created on server0
```

--------------------------------------

10.3 Reviewing systemd Journal Entries
======================================

特點：

- 存放在 `/run/log` 目錄中 (表示重開機之後就會消失)

- 最多使用系統 10% 的空間，超過的話，舊的 journal 就會被砍掉

- 使用 `journalctl` 命令來查詢 (<font color='red'>**只有 root 可用**</font>)

- severity `notice` or `warning` 會以粗體顯示，`error` 以上會以紅色表示

## 10.3.1 Finding events with journalctl

**<font color='red'>journalctl</font>** 的使用方式：

- `sudo journalctl`：顯示所有的 system journal

- `sudo journalctl -n 5`：使用 `-n` 參數，顯示最新五筆的 system journal

- `sudo journalctl -p err`：使用 `-p` 參數，指定要顯示 priority 為 error 的 system journal

- `sudo journalctl -f`：類似 `tail -f`，但持續顯示最新十筆

- `sudo journalctl --since today`：顯示今天發生的 system journal

- `sudo journalctl --since '2016-04-24 00:00:00' --until '2016-04-24 01:00:00'`：顯示特定時段內的 system journal

- `sudo journalctl --since="$(date -d "-30 minutes" +%F' '%H:%M:%S)"`：尋找 30 分鐘前的 system journal

- `sudo journalctl -o verbose`：顯示完整 system journal 訊息

- `sudo journalctl _PID=1`：顯示 pid=1 的 system journal

- `sudo journalctl -b`：顯示上一次開機到目前所存在的 system journal


--------------------------------------

10.4 Preserving the systemd Journal
===================================

保留 systemd journal 的方式：

1. `/var/log/journal` 目錄存在
2. 目錄的 owner 必須為 `root`，owner_group 必須為 `systemd-journal`，並設定權限為 `2755`

--------------------------------------

10.5 Maintain Accurate Time
===========================

## 10.5.1 Set local clocks and time zone

- `timedatectl`：顯示目前時區設定

- `timedatectl list-timezones`：顯示所有時區

- `timedatectl set-timezone Asia/Taipei`：更改時區

- `sudo timedatectl set-ntp true`：設定自動 NTP 教時

``` bash
# 看目前機器上的時間
$ timedatectl
      Local time: Thu 2016-01-21 06:35:27 UTC
  Universal time: Thu 2016-01-21 06:35:27 UTC
        RTC time: Thu 2016-01-21 06:35:26
       Time zone: UTC (UTC, +0000)
     NTP enabled: n/a
NTP synchronized: no
 RTC in local TZ: yes
      DST active: n/a

# 設定時區
$ sudo timedatectl set-timezone Asis/Taipei

# 啟用 NTP 校時
$ sudo timedatectl set-ntp true

# 透過互動的方式選擇時區(Time Zone)
$ tzselect
```

## 10.5.2 Configuring and monitoring chronyd

這個是用在沒有網路環境時，會由一台主要的 NTP server 去取得正確的資料，其他主機再透過 chronyd.service 進行時間同步。

加上 `iburst` 選項會讓網路校時更快，且更正確

`sudo hwclock -w`：強制將時間資訊寫入硬體

## 補充：設定 NTP 校時的完整步驟 (**很重要**)

- NTP server：`classroom.example.com`

```bash
$ sudo timedatectl set-ntp true

$ echo "server classroom.example.com iburst" | sudo tee --append /etc/chrony.conf
$ sudo systemctl restart chronyd.service

# 驗證設定結果
$ chronyc sources -v
210 Number of sources = 1

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
| /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||                                                /   xxxx = adjusted offset,
||         Log2(Polling interval) -.             |    yyyy = measured offset,
||                                  \            |    zzzz = estimated error.
||                                   |           |                         
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* classroom.example.com         8   6    17    18  +2830us[+3050us] +/- 3509us
```
