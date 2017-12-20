---
layout: post
title:  "[RHCE7] RH134 Chapter 4 Scheduling Future Linux Tasks 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 4 Scheduling Future Linux Tasks 留下的內容"
date: 2016-05-05 04:25:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

<a name="ch4.1" />
4.1 Scheduling One-Time Tasks with at
=====================================

```bash
[vagrant@server ~]$ at now+5minutes
at> cp /etc/yum.conf /tmp
at> <EOT>
job 1 at Mon Feb 22 08:23:00 2016

# 看目前的排程
$ atq

# 看指定排程的內容
$ at -c 1

# 刪除
$ atrm 1
```

```bash
[vagrant@server tmp]$ echo "cp /etc/passwd /tmp" > file.txt

[vagrant@server tmp]$ at now+10minutes < file.txt
job 2 at Mon Feb 22 08:31:00 2016

[vagrant@server tmp]$ atq
1	Mon Feb 22 08:23:00 2016 a vagrant
2	Mon Feb 22 08:31:00 2016 a vagrant
```

### Practice

```bash
[vagrant@server tmp]$ echo "date > ~/myjob" | at now+3min
job 3 at Mon Feb 22 08:33:00 2016

[vagrant@server tmp]$ atq
3	Mon Feb 22 08:33:00 2016 a vagrant

# 注意 while 的條件式中左右各要帶一個 space
[vagrant@server tmp]$ while [ $(atq | wc -l) -gt 0 ]; do sleep 1s; done
[vagrant@server tmp]$ cat ~/myjob
Mon Feb 22 08:33:00 UTC 2016

# 指定 queue
[vagrant@server tmp]$ at -q g teatime tomorrow
at> touch ~/cookies
at> <EOT>
job 4 at Tue Feb 23 16:00:00 2016

[vagrant@server tmp]$ at -q b 16:05 tomorrow
at> touch ~/cookies
at> <EOT>
job 5 at Tue Feb 23 16:05:00 2016

[vagrant@server tmp]$ atq
4	Tue Feb 23 16:00:00 2016 g vagrant
5	Tue Feb 23 16:05:00 2016 b vagrant
```

-------------------------------------------------------------

<a name="ch4.2" />
4.2 Scheduling Recurring Jobs with cron (User cron)
===================================================

```bash
# 透過 crontab -e 來設定 user cron jobs
[vagrant@server tmp]$ crontab -e
01 * * * *  ~/test.sh
30 2 * * *  run-parts   ~/cron.d

[vagrant@server tmp]$ sudo ls /var/spool/cron/
vagrant

# 使用 root 身份檢視 vagrant 使用者的 cron jobs
[vagrant@server tmp]$ sudo crontab -u vagrant -l
01 * * * *  ~/test.sh
30 2 * * *  run-parts   ~/cron.d

# 移除 vagrant 的 user cron jobs
[vagrant@server tmp]$ sudo crontab -u vagrant -r
[vagrant@server tmp]$ sudo crontab -u vagrant -l
no crontab for vagrant
```

### Practice

``` bash
[vagrant@server ~]$ crontab -e
*/2 9-16 * * 1-5 date >> /home/vagrant/my_first_cron_job

[vagrant@server ~]$ crontab -l
*/2 9-16 * * 1-5 date >> /home/vagrant/my_first_cron_job

[vagrant@server ~]$ cat ~/my_first_cron_job
Mon Feb 22 09:12:01 UTC 2016

[vagrant@server ~]$ crontab -r
[vagrant@server ~]$ crontab -l
no crontab for vagrant
```

-------------------------------------------------------------

<a name="ch4.3" />
4.3 Scheduling System cron Jobs
===============================

cron job 設定存在於 `/etc/crontab` 中，可透過 `man 5 crontab` 查詢格式 & 設定方式

```bash
# 每個小時的第 1 分鐘
01 * * * * *    root  /tmp/aa.sh

# 星期天 02:01
01 2 * * * 0    vagrant   /tmp/bb.sh

# 每個小時的第 2 分鐘，執行指定目錄中的所有檔案
02 * * * * *    root  run-parts   /root/cron.d
```

> 指定的檔案都必須要有 **execute**(chmod +x) 的權限才會正確執行


`/etc/anacrontab` & `/var/spool/anacron`(目錄)：用來處理未執行的 cron job

```bash
[vagrant@server tmp]$ sudo cat /etc/anacrontab
# /etc/anacrontab: configuration file for anacron

# See anacron(8) and anacrontab(5) for details.

SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45
# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22

#period in days   delay in minutes   job-identifier   command
1	5	cron.daily		nice run-parts /etc/cron.daily
7	25	cron.weekly		nice run-parts /etc/cron.weekly
@monthly 45	cron.monthly		nice run-parts /etc/cron.monthly


# 透過比較日期的方式，判斷有哪些該執行的 cron job 未執行
[vagrant@server tmp]$ ls /var/spool/anacron/
cron.daily  cron.monthly  cron.weekly
[vagrant@server tmp]$ sudo cat /var/spool/anacron/cron.daily
20160222
[vagrant@server tmp]$ sudo cat /var/spool/anacron/cron.monthly
20160222
[vagrant@server tmp]$ sudo cat /var/spool/anacron/cron.weekly
20160222
```

-------------------------------------------------------------

<a name="ch4.4" />
4.4 Managing Temporary Files
============================

從 RHEL 7 開始，`init` 已經被 `systemd` 所取代

在 RHEL 7 之前，暫存檔的監控 & 移除由 `tmpwatch` 套件來管理；但在 RHEL 7，systemd 提供了一個稱為 `systemd-tmpfiles` 的服務來監控 & 管理所指定的目錄

透過 `stat filename` 可以檢視 inode 的內容，與 systemd-tmpfiles 相關的為 `Access Time`(atime)、`Modify Time`(mtime)、以及 `Change Time`(ctime)

### 4.4.1 Managing temporary files with systemd-tmpfiles

`systemd-tmpfiles` 的功能在於定期(並非依賴 system cron，而是透過自身的 Timer 機制)的清除指定的目錄內容，或是恢復指定監控的目錄下被務刪的檔案，以下是設定範例(`/usr/lib/systemd/system/systemd-tmpfiles-clean.timer`)：

```init
[Timer]
OnBootSec=15min   # 開機後的 15 分鐘執行一次
OnUnitActiveSec=1d  # 之後每天執行一次
```

`systemd-tmpfiles` 的設定檔有 3 個地方：

- /etc/tmpfiles.d/\*.conf

- /run/tmpfiles.d/\*.conf

- /usr/lib/tmpfiles.d/\*.conf

> 下面兩個是屬於預設的設定檔，建議從下面兩個複製到第一個目錄後再修改，因為系統讀取到第一個目錄中有設定後，就不會執行下面兩個目錄的設定檔

以上設定表示 `systemd-tmpfiles-clean.service` 會在 systemd 啟動後的 15 分鐘後啟動執行，並在每 24 小時後重新執行一次，並根據上面三個目錄中的 `*.conf` 的設定，執行 `systemd-tmpfiles --clean` 來清除不需要的檔案(藉由比對檔案的 atime/mtime/ctime)。

以下是設定檔範例說明：

```bash
# 若 /run/tuned 目錄不存在，則建立，user/group 皆為 root，權限為 0755
d /run/tuned 0755 root root -

# 清除 /var/run/lsm 目錄中的內容
D /var/run/lsm 0755 libstoragemgmt libstoragemgmt -
```

> tmpfiles.d 詳細的設定檔撰寫方式可參考 tmpfiles.d(5), systemd-tmpfiles(8) 等文件


### 補充

RHEL 7 之前：standalone(daemon) service + xinetd(短暫式服務)

RHEL 7 之後：systemd service unit = service(對應到原先的 daemon) + socket(對應到原先的 xinetd) + path(監控目錄中檔案的變化，來決定執行的程式，例如：cups.path)

-------------------------------------------------------------

Practice: Managing Temporary Files
==================================

RHEL7 中，暫存檔由 `systemd-tmpfiles` 這個服務來進行管理。

將 /tmp 中的自動清除設定由 10 天改為 5 天：

```bash
# 複製 template
$ sudo cp /usr/lib/tmpfiles.d/tmp.conf /etc/tmpfiles.d/
$ cat /etc/tmpfiles.d/tmp.conf
.......
d /tmp 1777 root root 5d
d /var/tmp 1777 root root 30d
....

# 修改 /etc/tmpfiles.d/tmp.conf 檔案，將 10d 改為 5d
$ sudo sed -i '/^d .tmp /s/10d/5d/' /etc/tmpfiles.d/tmp.conf

# 測試設定是否成功
$ sudo systemd-tmpfiles --clean tmp.conf
```

設定 **/run/gallifrey** 每 30 秒清空一次：

```bash
# 編輯設定檔
$ echo "d /run/gallifrey 0700 root root 30s" | sudo tee /etc/tmpfiles.d/gallifrey.conf

# 建立規則
$ sudo systemd-tmpfiles create gallifrey.conf

# 測試是否有效
$ sudo touch /run/gallifrey/helloworld
$ sudo systemd-tmpfiles --clean gallifrey.conf
$ sudo ls -l /run/gallifrey/
```
