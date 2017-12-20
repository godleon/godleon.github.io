---
layout: post
title:  "[RHCE] RH124 Chapter 1~3 學習筆記"
description: "此文章記錄學習 RHCE RH124 所留下的內容，此篇包含 RH124 chapter 1~3，分別是 'ACCESSING THE COMMAND LINE', 'MANAGING FILES FROM THE COMMAND LINE', 以及 'GETTING HELP IN RED HAT ENTERPRISE LINUX'"
date: 2016-02-17 23:20:00
published: true
comments: true
categories: [Linux]
tags: [Linux, RHCE, RH124]
---

Introduction
============

### Internationalization

若想要讓 desktop & console 環境的語系一致，可以加入以下的 script 到 ~/.bashrc 中

```bash
i=$(grep 'Language' /var/lib/AccountsService/users/${USER} | \
sed 's/Language=//')
if [ "$i" != "" ]; then
    export LANG=$i
fi
```

透過 <font color='red'>`locale`</font> 可以查詢目前語系相關的設定 & 環境變數：

```bash
$ locale
LANG=zh_TW.UTF-8
.....(LANG-related environment variables)
LC_ALL=
```

若要修改整個系統的預設語系，可以透過以下指令：

```bash
$ localectl set-locale LANG=zh_TW.UTF-8
```

或是修改 <font color='red'>`/etc/locale.conf`</font> 檔案的內容：

```bash
$ cat /etc/locale.conf
LANG=zh_TW.UTF-8
```


-------------------------------------------------

<a name="ch1"/>
Chapter 1. ACCESSING THE COMMAND LINE
=====================================

1.1 Accessing the Command Line Using the Local Console
--------------------------------------------------

### Virtual Console

在 RHEL 7 中，若有 GUI 環境，則會預設執行在第一個 virtual console，另外還會包含 5 個文字模式的 virtual console，可使用 `Ctrl + Alt + F[1-6]` 在不同的 virtual console 間切換。

若沒有 GUI 環境，則 6 個 virtual console 都會是純文字模式。

要調整 virtual console 的數量，可修改 `/etc/systemd/login.conf` 中的 **<font color='red'>NAutoVTs</font>** 的選項，


1.2 Accrssing the Command Line Using the Desktop
--------------------------------------------

### Windows 連線至 Linux GUI

要從 Windows 連線到 Linux GUI，可使用 [pietty](http://ntu.csie.org/~piaip/pietty/) + [Xming](http://sourceforge.net/projects/xming/)

使用說明可參考 => [八克里: 使用 xming 從windows 系統登入 Linux 系統](http://blog.jangmt.com/2009/11/xming.html)

### Auto Login

要在 RHEL 7 作到 Auto Login，要修改 `/etc/gdm/custom.conf`，並調整 <font color='blue'>**daemon</font>** section 中的 **<font color='red'>AutomaticLoginEnable</font>** & **<font color='red'>AutomaticLogin</font>** 兩個參數


1.3 Executing Commands Using the Bash Shell
---------------------------------------

### Examples of simple commands

**<font color='red'>file</font>** 可用來檢查檔案的型態 & 格式 (也可以用 **[stat](http://www.computerhope.com/unix/stat.htm)**)

```bash
$ file /etc/passwd
/etc/passwd: ASCII text

$ file /bin/passwd
/bin/passwd: setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=0x91a7160a019b7f5f754264d920e257522c5bce67, stripped
```

``` bash
$ stat /etc/passwd
  File: ‘/etc/passwd’
  Size: 961             Blocks: 8          IO Block: 4096   regular file
Device: fd01h/64769d    Inode: 1573453     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Context: system_u:object_r:passwd_file_t:s0
Access: 2016-01-20 07:38:22.557000000 -0500
Modify: 2015-10-02 10:38:00.710867846 -0400
Change: 2015-10-02 10:38:00.710867846 -0400
 Birth: -

$ stat /bin/passwd
  File: ‘/bin/passwd’
  Size: 27832           Blocks: 56         IO Block: 4096   regular file
Device: fd01h/64769d    Inode: 2234382     Links: 1
Access: (4755/-rwsr-xr-x)  Uid: (    0/    root)   Gid: (    0/    root)
Context: system_u:object_r:passwd_exec_t:s0
Access: 2014-06-10 02:27:56.000000000 -0400
Modify: 2014-06-10 02:27:56.000000000 -0400
Change: 2015-10-02 10:30:36.743867846 -0400
 Birth: -

# 甚至可以直接針對目錄來檢查
$ file /home
/home: directory
```

### Editing the command Line

使用 command line 的實用快速鍵：

| 快速鍵 | 說明 |
|-------|------|
| `Ctrl + a`  | 游標跳至最前 |
| `Ctrl + e`  | 游標跳至最後  |
| `Ctrl + u` | 清除整個命令中，從游標到最前面的內容 |
| `Ctrl + k` | 清除整個命令中，從游標到最後面的內容 |
| `ESC + . | 複製上一個命令中的最後一個參數到目前的命令中 |
| `Alt + . | 同上 |
| `Ctrl + r` | 可用 keyword 來尋找最近使用過的命令 |
| `Ctrl + l(小寫 L)` | 清除螢幕內容(效果等同 `clear`) |


-------------------------------------------------

<a name="ch2"/>
Chapter 2. MANAGING FILES FROM THE COMMAND LINE
===============================================

2.1 The file system hierarchy
-----------------------------

RHEL 中的重要目錄：

| 路徑 | 目的 |
|------|-----|
| `/usr`<br />(Unix Software Resource)  | 安裝的軟體、shared library ... 等資料都會放在此處，其中幾個重要目錄：<br />`/usr/bin`：使用者用指令<br />`/usr/sbin`：系統管理者用指令<br />`/usr/local`：使用者自行安裝的軟體 |
| `/etc` | 設定檔存放路徑 |
| `/var` | 持續不斷變動的資料，例如 log、print spool、資料庫檔案 ... 等等 |
| `/run` | 從上次開機以來的 runtime 資訊(**<font color='red'>此目錄的資料在每次重開機都會清空</font>**) |
| `/tmp` | 所有人都有權限存取的站存資料目錄(**<font color='red'>此目錄中日期大於 10 天的資料會被自動清除</font>**)，若是在目錄 `/var/tmp` 中的資料，則是超過 30 天的資料會被清除 |
| `/boot` | 系統開機所需要的檔案 |
| `/dev` | 存放系統用來存取硬體裝置所需要的檔案 |

> 原本在 `/` 下的某些目錄，在 RHEL 7 後都被移到 `/usr` 下了，包含 `/bin`(=> `/usr/bin`)、`/sbin`(=> `/usr/sbin`)、`/lib`(=> `/usr/lib`)、`/lib64`(=> `/usr/lib64`)
> 但原本在 `/` 的以上四個目錄都還存在，只是改成用 symbolic link 的方式連到 `/usr` 中的子目錄

詳細資料可查詢 <font color='blue'>**hier(7)</font>** man page

```bash
$ man 7 hier
```

2.2 Locating Files by name
--------------------------

檔名限制 & 特性：

1. 完整檔案路徑長度不能超過 4095 bytes (含 `/`)

2. 兩個 `/` 之間的長度不能超過 255 bytes

3. 檔名可以是任意的 UTF-8 字元，但不能是 `/` & `NUL`

4. Case-Sensative

### Navigating paths

- **touch**
> touch 會更新檔案的 timestamp 到目前的時間，而不會改變檔案內容
> 若是不存在的檔案，則會建立一個空白檔案

```bash
[vagrant@server ~]$ touch /tmp/test{1,2}.txt
[vagrant@server ~]$ ls -l /tmp/
total 0
-rw-rw-r--. 1 vagrant vagrant 0 Jan 30 00:23 test1.txt
-rw-rw-r--. 1 vagrant vagrant 0 Jan 30 00:23 test2.txt
```

2.3 Managing Files Using command-Line Tools
-------------------------------------------

```bash
# 保留複製檔案的屬性
$ cp -a
```

2.4 Matching File Names Using Path Name Expansion
-------------------------------------------------

### File globbing: path name Expansion

| Pattern | Matches |
|---------|---------|
| `~+` | 目前工作目錄 |
| `~-` | 上一個工作目錄 |
| `[abc...]` | 任何在中括號中的字母都符合 |
| `[!abc...]` | 不包含中括號中的任何一個字母 |
| `[^abc...]` | 同上 |
| `[[:punct:]]` | 任何可印出來的字元(但不包含空白 or 英文字母) |

#### Brace Expansion

```bash
[vagrant@server ~]$ echo {Sunday,Month,Tuesday}.lo
Sunday.log Month.log Tuesday.log

[vagrant@server ~]$ echo file{a..c}.txt
filea.txt fileb.txt filec.txt

[vagrant@server ~]$ echo file{a,b}{1,2}.txt
filea1.txt filea2.txt fileb1.txt fileb2.txt

[vagrant@server ~]$ echo file{a{1,2},b,c}.txt
filea1.txt filea2.txt fileb.txt filec.txt
```

#### Command Substitution

- 單引號中的變數 $ 無效

- 雙引號中的變數 $ 有效

- 較推薦加大括號確定變數名稱的方式，比較不容易跟單引號混淆

```bash
[vagrant@server ~]$ echo Today is `date +%A`
Today is Saturday

[vagrant@server ~]$ echo The time is $(date +%M) minutes past $(date +%l%p)
The time is 33 minutes past 8AM
```

2.5 Lab: Managing Files with Shell Expansion
--------------------------------------------

```bash
[vagrant@server lab]$ touch tv_season{1,2}_episode{1..6}.ogg
[vagrant@server lab]$ ls
tv_season1_episode1.ogg  tv_season1_episode3.ogg  tv_season1_episode5.ogg  tv_season2_episode1.ogg  tv_season2_episode3.ogg  tv_season2_episode5.ogg
tv_season1_episode2.ogg  tv_season1_episode4.ogg  tv_season1_episode6.ogg  tv_season2_episode2.ogg  tv_season2_episode4.ogg  tv_season2_episode6.ogg

[vagrant@server lab]$ touch mystery_chapter{1..8}.odf
[vagrant@server lab]$ ls
mystery_chapter1.odf  mystery_chapter4.odf  mystery_chapter7.odf     tv_season1_episode2.ogg  tv_season1_episode5.ogg  tv_season2_episode2.ogg  tv_season2_episode5.ogg
mystery_chapter2.odf  mystery_chapter5.odf  mystery_chapter8.odf     tv_season1_episode3.ogg  tv_season1_episode6.ogg  tv_season2_episode3.ogg  tv_season2_episode6.ogg
mystery_chapter3.odf  mystery_chapter6.odf  tv_season1_episode1.ogg  tv_season1_episode4.ogg  tv_season2_episode1.ogg  tv_season2_episode4.ogg

[vagrant@server lab]$ mkdir -p Videos/season{1,2}
[vagrant@server lab]$ ls Videos/
season1  season2

[vagrant@server lab]$ mv tv_season1* Videos/season1/
[vagrant@server lab]$ mv tv_season2* Videos/season2/
[vagrant@server lab]$ ls -R Videos/
Videos/:
season1  season2

Videos/season1:
tv_season1_episode1.ogg  tv_season1_episode2.ogg  tv_season1_episode3.ogg  tv_season1_episode4.ogg  tv_season1_episode5.ogg  tv_season1_episode6.ogg

Videos/season2:
tv_season2_episode1.ogg  tv_season2_episode2.ogg  tv_season2_episode3.ogg  tv_season2_episode4.ogg  tv_season2_episode5.ogg  tv_season2_episode6.ogg

[vagrant@server lab]$ mkdir -p ./my_bestseller ./chapters

[vagrant@server lab]$ mkdir ./my_bestseller/{editor,plot_change,vacation}
[vagrant@server lab]$ ls ./my_bestseller/
editor  plot_change  vacation

[vagrant@server lab]$ cd chapters/
[vagrant@server chapters]$ mv ../*chapter*.odf ./

[vagrant@server chapters]$ mv mystery_chapter{1,2}.odf ../my_bestseller/editor/

[vagrant@server chapters]$ mv mystery_chapter{7,8}.odf ../my_bestseller/vacation/

[vagrant@server chapters]$ cd ../Videos/season2/
[vagrant@server season2]$ cp tv_season2_episode1.ogg ../../my_bestseller/vacation/

[vagrant@server season2]$ cd /tmp/lab/my_bestseller/vacation/
[vagrant@server vacation]$ ls
mystery_chapter7.odf  mystery_chapter8.odf  tv_season2_episode1.ogg
[vagrant@server vacation]$ cd ~-
[vagrant@server season2]$ cp tv_season2_episode2.ogg ~-
[vagrant@server season2]$ cd ~-
[vagrant@server vacation]$ ls
mystery_chapter7.odf  mystery_chapter8.odf  tv_season2_episode1.ogg  tv_season2_episode2.ogg

[vagrant@server vacation]$ cp /tmp/lab/chapters/mystery_chapter5.odf /tmp/lab/my_bestseller/plot_change/mystery_chapter5_$(date +%F).odf
[vagrant@server vacation]$ cp /tmp/lab/chapters/mystery_chapter5.odf /tmp/lab/my_bestseller/plot_change/mystery_chapter5_$(date +%s).odf
[vagrant@server vacation]$ cp /tmp/lab/chapters/mystery_chapter5.odf /tmp/lab/my_bestseller/plot_change/mystery_chapter5_$USER.odf
[vagrant@server vacation]$ ls /tmp/lab/my_bestseller/plot_change/
mystery_chapter5_1455368579.odf  mystery_chapter5_2016-02-13.odf  mystery_chapter5_vagrant.odf
```


-------------------------------------------------

<a name="ch3"/>
Chapter 3. GETTING HELP IN RED HAT ENTERPRISE LINUX
===================================================

## 3.1 Reading Documentation Using man Command

### 3.1.1 Introducing the man command

Linux manual 包含多個 section：

| Section | Content Type |
|---------|--------------|
| **<font color='red'>1</font>** | 一般使用者命令(包含可執行程式 & shell script) |
| `2` | System calls(kernal routines invoked from user space) |
| `3` | Library functions (程式函式庫提供) |
| `4` | 特殊檔案(例如：設備檔 /dev 目錄中的檔案) |
| **<font color='red'>5</font>** | <font color='blue'>檔案格式(設定檔 & 內容結構說明)</font> |
| `6` | Games |
| **<font color='red'>7</font>** | 慣例、標準、其他...等等(協定、檔案系統) |
| **<font color='red'>8</font>** | <font color='blue'>系統管理員以及特殊指令(用於維護工作)</font> |
| `9` | Linux kernal API (internal kernel calls) |

`man 1 passwd` or `man passwd`(未指定 section 則預設帶 1) 可以知道使用指令的方式 & 相關參數

`man 5 passwd` 則是說明 **/etc/passwd** 的檔案結構，組成內容....等資訊

> 以上資訊要安裝 **<font color='red'>man-pages</font>** 套件才會有

### 3.1.3 Searching for man pages by keywords

小寫 k 僅針對 title & description 搜尋關鍵字：

```bash
[vagrant@server ~]$ man -k passwd
grub2-mkpasswd-pbkdf2 (1) - Generate a PBKDF2 password hash.
sslpasswd (1ssl)     - compute password hashes
```

> 大寫 K 會進行全文搜尋.....內容很多...

透過 `mandb` 指令可以立即強制 man page 資料庫更新，但系統其實已經將更新資料庫的工作放在 **/etc/cron.daily/man-db.cron** 中。


## 3.2 Reading Documentation Using pinfo Command

info 文件是以類似超連結網頁的方式進行編排，透過 pinfo 指令來啟動 **lynx** 文字網頁瀏覽器來瀏覽。

`--` 在指令中代表 command option 的結束，表示後面接的是 command argument，例如：`touch -- -r` 會產生名稱為 **-r** 的檔案。
