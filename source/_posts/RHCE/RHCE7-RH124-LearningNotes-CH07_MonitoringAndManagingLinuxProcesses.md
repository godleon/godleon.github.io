---
layout: post
title:  "[RHCE] RH124 Chapter 7 Monitoring and Managing Linux Processes 學習筆記"
description: "此文章記錄學習 RHCE RH124 Chapter 7 Monitoring and Managing Linux Processes 所留下的內容"
date: 2016-04-14 04:30:00
published: false
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---

7.1 Process
===========

## 7.1.1 What is a process?

每個 process 都會有個獨一無二的 PID，而 parent process ID(PPID) 則為 PID 的父程序所擁有的 ID；process 可透過 `fork` 的方式產生 child process，而這些 child process 則會繼承 parent process 的 security identifiers, file descriptors, port, resource privileges, 環境變數....等等，整個 process lifeycle 可以參考下圖：

![Process Lifecycle](https://www.freebsd.org/doc/en_US.ISO8859-1/books/design-44bsd/fig1.png)

在 RHEL7 中，所有的 process 都是 **<font color='red'>systemd(1)</font>** 的 child process。

## 7.1.2 Process States

![Linux Process States](http://4.bp.blogspot.com/-5jYQDgc6Z4M/UyV1ni478uI/AAAAAAAADZw/rpBf813wpbg/s1600/ProcessStates.JPG)

| 狀態 | Flag | kernel-defined state and description |
|------|------|--------------------------------------|
| Running | R | **<font color='red'>TASK_RUNNING</font>**：正在等待執行 or 正在執行的 process，正在執行的又包含執行 user routine & kernel routine(system calls)。例如：Data -> Memory |
| Sleeping | S | **<font color='red'>TASK_INTERRUPTIBLE</font>**：等待特定的情況(硬體要求、系統資源存取、信號....等)發生，當條件滿足時就會回到 Running 的狀態 |
| Sleeping | D | **<font color='red'>TASK_UNINTERRUPTIBLE</font>**：與 **<font color='red'>S</font>** 類似，但不會回應從其他地方送來的信號。例如：寫資料到外接儲存裝置時(Memory -> USB) |
| Sleeping | K | **<font color='red'>TASK_KILLABLE</font>**：與 **<font color='red'>D</font>** 相反，可以接收來自其他地方的信號，例如：掛載網路磁碟機 |
| Stopped | T | **<font color='red'>TASK_STOPPED</font>**：可能因為 user 或是其他 process 送來訊號而進入 Stopped 狀態，也可能因為特定訊號而返回 Running 狀態 |
| Stopped | T | **<font color='red'>TASK_TRACED</font>**：同上，但可 debug |
| Zombie | Z | **<font color='red'>EXIT_ZOMBIE</font>**：已通知 parent process 準備離開後的狀態，除了 process identity 之外的資源都會被釋放 |
| Zombie | X | **<font color='red'>EXIT_DEAD</font>**：所有資源都被釋放，ps 也看不見了 |

## 7.1.2 Listing processes

ps 指定常用的參數：
- `aux`
- `las`
- `afx` (含階層)
- `-O` (指定要顯示的欄位)
- `--sort`：排序

`ps` 可用來觀察目前 process 的狀態，有以下幾種格式

- UNIX(POSIX)：參數加上一個 `-`

- BSD：參數不加上 `-`

- GNU long：參數加上兩個 `-`

```bash
# BSD
[student@server0 ~]$ ps f
  PID TTY      STAT   TIME COMMAND
 1986 pts/0    Ss     0:00 -bash
 2072 pts/0    R+     0:00  \_ ps f

 # 以階層顯示(顯示往上五階)
[student@server0 ~]$ ps afx | grep -B5 ssh-agent
  1130 ?        Ss     0:00 /usr/sbin/sshd -D
  2823 ?        Ss     0:00  \_ sshd: student [priv]
  2827 ?        S      0:00      \_ sshd: student@pts/0
  2829 pts/0    Ss     0:00          \_ -bash
  3110 pts/0    R+     0:00              \_ ps afx
  3111 pts/0    S+     0:00              \_ grep --color=auto -B5 ssh-agent

 # UNIX(POSIX)
[student@server0 ~]$ ps -f
UID        PID  PPID  C STIME TTY          TIME CMD
student   1986  1983  0 05:28 pts/0    00:00:00 -bash
student   2073  1986  0 05:30 pts/0    00:00:00 ps -f
[student@server0 ~]$ ps --f
```

檢視全部的 process 常用 `aux` 選項 or `alx`(較為詳細)，若想要檢視 process 之間的父子關係可使用 `afx`(關鍵是 `f` 參數)

關於 ps 還有幾個重點：

- 若 ps 不加上任何參數，就僅會顯示與目前這個 user terminal 有關係的 process：

```bash
[student@server0 ~]$ ps
  PID TTY          TIME CMD
 1986 pts/0    00:00:00 bash
 2326 pts/0    00:00:00 ps
```

- 顯示結果中，若是以 **<font color='red'>[ ]</font>** 包覆的，表示為 kernel thread

```bash
[student@server0 ~]$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.2  0.3  52328  6484 ?        Ss   04:55   0:06 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root         2  0.0  0.0      0     0 ?        S    04:55   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S    04:55   0:00 [ksoftirqd/0]
root         5  0.0  0.0      0     0 ?        S<   04:55   0:00 [kworker/0:0H]
root         6  0.0  0.0      0     0 ?        S    04:55   0:00 [kworker/u2:0]
```

- 透過 `ps afx` or `pstree` 可看到 process 之間的 parent/child 關係

- 若要排序 ps 出來的結果，可透過 `--sort` 選項 ([參考網址(Linux process memory usage - how to sort the ps command)](http://alvinalexander.com/linux/unix-linux-process-memory-sort-ps-command-cpu))

```bash
# 根據 CPU 遞增排序
$ ps au --sort=%cpu

# 根據 CPU 遞減排序
$ ps au --sort=-%cpu

# PID 從小到大排序
[student@server0 ~]$ ps aux --sort pid
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.5  0.3  52328  6472 ?        Ss   19:53   0:09 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root         2  0.0  0.0      0     0 ?        S    19:53   0:00 [kthreadd]
root         3  0.0  0.0      0     0 ?        S    19:53   0:00 [ksoftirqd/0]

# PID 從大到小排序
[student@server0 ~]$ ps aux --sort -pid
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
student   2065  0.0  0.0 123516  1448 pts/0    R+   20:24   0:00 ps aux --sort -pid
root      2064  0.0  0.0 107892   364 ?        S    20:24   0:00 sleep 60
root      2001  0.0  0.0      0     0 ?        S    20:19   0:00 [kworker/0:0]
```

-------------------------------------------------------

7.2 Controlling Jobs
====================

`jobs` 指令只會顯示目前 session 中的 job

```bash
$ gedit &
$ jobs
$ nautilus &
$ jobs
$ sleep 1000 &
$ jobs
$ fg
CTRL + C (終止目前前景執行的程式)
$ jobs
$ fg %1
CTRL + C (終止目前前景執行的程式)
```

## 7.2.1 Jobs and sessions

當 terminal or console 被開啟時，會產生一個 process session，所以透過同一個 terminal or console 產生出來的 process，都會使用相同的 session ID，而每個 session 中，一次只能有一個 process 在前景執行。

service daemon 或是 kernel process thread 在 `ps` 出來的結果中，`TTY` 欄位會以 `?` 來呈現，因為此類的 background process 並沒有 controlling terminal。

## 7.2.2 Running jobs in the background

`Ctrl + z`：可送出 suspend 要求，用來讓 process 進入 Stopped 狀態(T)

`Ctrl + c`：中斷 process 執行

`bg %JOB_ID`：可讓 process 恢復執行。

``` bash
# 一個執行中的範例
$ dd < /dev/zero > /dev/null

# R(Running) 狀態
$ ps au | grep dd

Ctrl + Z

# 變成 S(Sleeping, TASK_INTERRUPTIBLE) 狀態
$ ps au | grep dd

$ sleep 1000

# S(Sleep) 狀態
$ ps au | grep sleep

# 用來驗證 D(Sleeping, TASK_UNINTERRUPTIBLE) 狀態
$ dd if=/dev/zero bs=1M count=50 of=/run/media/...../test.txt
```

-------------------------------------------------------

7.3 Killing Processes
=====================

## 7.3.1 Process control using signals

signal 是送到 process 的一種軟體型式的中斷，有以下幾種：

| Signal number | Short name | Definition | Purpose |
|---------------|------------|------------|---------|
| `1` | `HUP` | Hangup |  |
| `2` | `INT` | Keyboard interrupt | 讓程式中止，等同按下 `Ctrl + c` |
| `3` | `QUIT` | Keyboard quit | 讓 process 中止，但會執行 core dump 的動作，等同按下 `Ctrl + \` |
| `9` | `KILL` | Kill, unblockable | 強行中止 |
| `15`(default) | `TERM` | Terminate | 通知程式中止，允許 process 完成 self-cleanup 後才中止 |
| `20` | `TSTP` | Keyboard stop | 讓 process 中斷執行(但可恢復)，效果等同按下 `Ctrl + z` |

> 詳細的 signal 列表可使用 `kill -l` 查詢

> 基本上砍掉 process 時先嘗試用 `SIGTERM`，不行的話才改用 `SIGKILL`，會是比較穩妥的作法‧

> 沒指定 SIG 則預設給 <font color='red'>**SIGTERM(15)**</font>

### Logging users out administratively

`w` 指令是用來檢視目前登入到系統的使用者以及累積到現在的資源使用狀況：

```bash
[kiosk@foundation0 ~]$ w -f
 06:19:29 up  1:27,  3 users,  load average: 0.04, 0.07, 0.06
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
kiosk    :0       :0               04:52   ?xdm?  11:36   0.20s gdm-session-worker [pam/gdm-autologin]
kiosk    pts/3    :0               05:27   52:14   0.03s  0.03s bash
kiosk    pts/4    192.168.1.190    06:11    1.00s  0.03s  0.01s w -f
```

`pkill` 是個強大的關閉 process 的指令，可透過以下方式過濾：
- command
- UID
- GID
- parent
- terminal

`pgrep` 指定 user 作 grep

``` bash
# 刪除使用者 bboop 相關的 process
$ pkill -u bboop

# 刪除使用者 bboop 相關的 process 並強制登出
$ pkill -SIGKILL -u bboop
```

```bash
[student@server0 ~]$ sleep 1000 &
[1] 2893
[student@server0 ~]$ sleep 1000 &
[2] 2894
[student@server0 ~]$ sleep 1000 &
[3] 2895
# pgrep 視特定使用者
[student@server0 ~]$ pgrep -l -u student
2827 sshd
2829 bash
2893 sleep
2894 sleep
2895 sleep
# pstree 檢視特定使用者，使用 PID
[student@server0 ~]$ pstree -p student
sshd(2827)───bash(2829)─┬─pstree(2896)
                        ├─sleep(2893)
                        ├─sleep(2894)
                        └─sleep(2895)

# 指定 PPID(parent ID) 刪除 process
[student@server0 ~]$ pkill -SIGKILL -P 2829
[1]   Killed                  sleep 1000
[2]-  Killed                  sleep 1000
[3]+  Killed                  sleep 1000
[student@server0 ~]$ pstree -p student
sshd(2827)───bash(2829)───pstree(2945)
```

-------------------------------------------------------

7.4 Monitoring Process Activity
===============================

## 7.4.1 Load average

靜態呈現，使用 `w -f` or `uptime`：

``` bash
# 查詢目前系統的 load average
$ w -f
 16:37:45 up  7:29,  3 users,  load average: 0.20, 0.17, 0.14
USER     TTY        LOGIN@   IDLE   JCPU   PCPU WHAT
godleon  tty8      09:08    7:29m  9:30   0.37s mate-session

# $ uptime
 16:43:07 up  7:35,  3 users,  load average: 0.18, 0.15, 0.14
```

> load average 超過 1 就表示系統負擔過重(要先除以 CPU 執行緒數量)

## 7.4.2 Real-time process monitoring

動態呈現，使用 `top`：

- `VIRT`：process 消耗所有記憶體大小(包含實體 & 虛擬)，等同 ps 指令中的 `VSZ`

- `RES`：process 所使用的實體記憶體大小，等同 ps 指令中的 `RSS`

在 top 中常用的按鍵：

| Key | Purpose |
|-----|---------|
| `l`, `t`, `m` | 開啟 or 關閉 load, thread, memory 資訊 |
| `M` | 記憶體使用量從大排到小 |
| `P` | CPU 使用量從大排到小 |
| `k` | 指定 PID 並 TERM 該 process |
| `r` | renice 指定 process |
| `W` | 記錄目前的 top 觀察設定，並可作為下次使用 top 時的預設設定 |
| `B` | header & running process 會以粗體顯示 |
