---
layout: post
title:  "[RHCE] RH124 Chapter 14 Accessing Linux File Systems 學習筆記"
description: "此文章記錄學習 RHCE7 RH124 Chapter 14 Accessing Linux File System 留下的內容"
date: 2016-04-28 15:25:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---


<a name="ch14.1" />
14.1 Identifying File Systems and Devices
=========================================

常用指令：

- `sudo du -h / --max-depth=1 2>/dev/null | sort -h`：檢查 root directory 每個目錄所使用的容量

--------------------------------------------

<a name="ch14.2" />
14.2 Mounting and Unmounting File Systems
=========================================

常用指令：

- `blkid`：顯示所有 block device 資訊

- `mount source_device destination_dir`：透過 device name 掛載

- `mount UUID destination_dir`：透過 UUID 掛載

```bash
# 查詢機器上的 block device
$ sudo blkid
/dev/vda1: UUID="9bf6b9f7-92ad-441b-848e-0257cbb883d1" TYPE="xfs" 
/dev/vdb1: UUID="bffdaa4a-34f2-4a74-8455-a11aca40a6e1" TYPE="xfs" 

# 指定 UUID 掛載 block device
$ sudo mkdir /mnt/newspace && sudo mount UUID="bffdaa4a-34f2-4a74-8455-a11aca40a6e1" /mnt/newspace

# 卸離 block device
$ sudo umount /mnt/newspace
```

--------------------------------------------

<a name="ch14.3" />
14.3 Making Links Between Files
===============================

## Partition UUID & inode

> UUID 存於 super block 中

inode block -> inode table 結構：
1. inode number
2. Permission
3. Hard Link Subdirectory 數量
4. UID
5. GID
6. Size
7. Timestamp
8. Filename
9. Pointer

## Hard Link

**<font color='red'>inode 在 Linux 中是真正指向檔案實際內容的指標</font>**

透過 `ln` 可建立 Hard Link，這是個指向 inode 的連結

```bash
# 產生檔案
[student@server0 ~]$ echo "Hello World" > newfile.txt
[student@server0 ~]$ ls -l
total 4
-rw-rw-r--. 1 student student 12  4月 28 15:05 newfile.txt

# 建立 hard link (注意數字從 1 變成 2, inode number 相同)
[student@server0 ~]$ ln newfile.txt ~/newfile-hlink.txt
[student@server0 ~]$ ls -li
total 8
12889 -rw-rw-r--. 2 student student 12  4月 28 15:05 newfile-hlink.txt
12889 -rw-rw-r--. 2 student student 12  4月 28 15:05 newfile.txt
```

Hard Link 特性 & 說明：
- 上面建立 Hard Link 的示範，可看出指向同一個 inode 的連結，從一個變成兩個(可防止檔案誤刪)
- 增加 hard link 不會增加磁碟空間
- 不能跨 File System
- 不能 link 目錄，只能建立在檔案上

## Symbolic Link 

```bash
# 建立 symbolic link
$ ln -s newfile.txt ~/newfile-symlink.txt
$ ls -l
total 8
-rw-rw-r--. 2 student student 12  4月 28 15:05 newfile-hlink.txt
lrwxrwxrwx. 1 student student 11  4月 28 15:19 newfile-symlink.txt -> newfile.txt
-rw-rw-r--. 2 student student 12  4月 28 15:05 newfile.txt

# 移除 symbolic link 指向的檔案(系統會標示連結失效)
$ rm newfile.txt 
$ ls -l
total 4
-rw-rw-r--. 1 student student 12  4月 28 15:05 newfile-hlink.txt
lrwxrwxrwx. 1 student student 11  4月 28 15:19 newfile-symlink.txt -> newfile.txt (這裡會有底色標記連結失效)

# link 目錄
$ ln -s /etc ~/config_files
```

特色：
1. 類似捷徑
2. 不能防止檔案誤刪

--------------------------------------------

<a name="ch14.4" />
14.4 Locating Files on the System
=================================

## locate

要使用 locate 之前必須先執行 `sudo updatedb`，才會有檔案資料庫可用，若要搜尋最新的檔案，也必須要執行 updatedb

- `sudo locate passwd`：尋找檔名為 passwd 的檔案

- `sudo locate -n 5 passwd`：同上，但只列出 5 筆資料

- `sudo locate -i messages`：以 case-insensitive 的方式搜尋

## find

即時搜尋，可找到剛新增的檔案

- `sudo find / -name sshd_config`：搜尋檔名為 sshd_config 的檔案

- `sudo find / -name '*.txt'`：在 / 目錄下尋找副檔名為 txt 的檔案

- `sudo find / -iname '*messages*'`：在 / 目錄下以 case-insensitive 的方式檔名尋找 *messages* 的檔案

- `sudo find -user student`：尋找 /home 目錄中，user 為 student 的檔案

- `sudo find -group student`：尋找 /home 目錄中，group 為 student 的檔案

- `sudo find / -user root -group mail`：在 / 目錄中尋找 user=root, group=mail 的檔案

- `sudo find /home -perm 764`：尋找 /home 中 permission=764 檔案

- `sudo find /home -perm -324`：尋找 /home 中，**<font color='red'>至少</font>**有指定權限的檔案

- `sudo find /home -perm /442`：尋找 /home 中，user(read)/group(read)/others(write) 至少其中一個符合指定權限的檔案

- `sudo find / -perm /7000`：搜尋檔案當中含有 SGID 或 SUID 或 SBIT 的屬性

- `sudo find /run -type s`：找出 /run 目錄中，檔案類型為 Socket 的檔名有哪些
> type 選項可以有 f(一般檔案) / d(目錄) / l(symbolic link) / b(block device)

- `sudo find -size -10M`：尋找小於 10MB 的檔案

- `sudo find / -type f -links +1`：尋找擁有超過 1 個 hard link 的一般檔案

### find 的特別功能

find 還可以針對搜尋結果加上 action：

`sudo find /etc/yum.repos.d/ -type f -exec mv {} {}1 \;`

以上指令表示：

1. 搜尋 /etc/yum.repos.d/ 目錄中的一般檔案

2. 將每個檔案進行改名，在檔名後面多加一個 1

--------------------------------------------

參考資料
=======
- [鳥哥的 Linux 私房菜 -- 第七章、Linux 磁碟與檔案系統管理 >> 7.2.2 實體連結與符號連結： ln](http://linux.vbird.org/linux_basic/0230filesystem.php#link)
- [鳥哥的 Linux 私房菜 -- 第六章、Linux 檔案與目錄管理 >> 6.5 指令與檔案的搜尋](http://linux.vbird.org/linux_basic/0220filemanager.php#file_find)
