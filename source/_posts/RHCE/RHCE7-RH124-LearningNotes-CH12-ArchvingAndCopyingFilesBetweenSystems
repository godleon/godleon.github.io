---
layout: post
title:  "[RHCE] RH124 Chapter 11 Managing Red Hat Enterprise Linux Networking 學習筆記"
description: "此文章記錄學習 RHCE RH124 Chapter 12 Archiving And Copying Files Between Systems 留下的內容"
date: 2016-04-27 15:30:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---

<a name="ch12.1" />
12.1 Managing Compressed tar Archives
=====================================

## 12.1.2 Archive files and directories with tar

tar 指令要先輸入目的地檔案，後面才是來源檔案 (跟 cp & mv 等指令相反)

tar 打包檔案中若包含絕對路徑檔案，則會把最開頭的 `/` 拿掉 (安全因素)

tar 打包檔案時會包含檔案的修改時間、權限 ... 等資訊，但預設不儲存檔案 SELinux conext & ACL 屬性(若要一起打包則要在指令最後方加上 `--xattrs` 參數)

指令參考：

- `tar cf archive.tar file1 file2 file3`：將三個檔案打包為 archive.tar

- `tar tf archive.tar`：列出 archive.tar 中的內容

- `sudo tar cf /root/etc.tar /etc`：打包整個 /etc 目錄成 /root/etc.tar

## 12.1.3 Extract an archive created with tar

`mkdir test && sudo sudo tar xf /root/etc.tar -C test/`：將 /root/etc.tar 解開後放到 test 目錄中

> 用 tar 解開打包檔，若要保留原有檔案的權限資訊，要加上 `-p` 參數 (這是 root 預設就會包含的選項)


## 12.1.4 Create a compressed tar archive

- 'z'：gzip (archive.tgz / archive.tar.gz)

- 'j'：bz2 (archive.tar.bz2)

- 'J'：xz (archive.tar.xz) (壓縮效率最好，但速度最慢)

參考指令：

- `sudo tar czf /root/etc.tar.gz /etc`：以 /etc 為資料來源，建立 gzip 打包壓縮檔

- `sudo tar cjf /root/etc.tar.bz2 /etc`：以 /etc 為資料來源，建立 bzip2 打包壓縮檔

- `sudo tar cJf /root/etc.tar.xz /etc`：以 /etc 為資料來源，建立 xz 打包壓縮檔

## 12.1.5 Extract a compressed tar archive

系統會清楚知道檔案壓縮的格式，因此解壓縮時不用加上 `z` or `j` or `J` 也沒關係

參考指令：

- `sudo tar xJf /root/etc.tar.xz -C test/`：將上述 xz 壓縮檔解壓縮到 test 目錄下

--------------------------------------------------------

<a name="ch12.2" />
12.2 Copying Files Between Systems Securely
===========================================

`scp`：適合用於單一檔案

`rsync`：可用於單一檔案，但特色是目錄的同步與屬性的保留

`sftp`：互動功能，Windows 作業系統上較常用

參考指令：

- `scp /etc/yum.conf /etc/hosts student@172.25.0.11:/home/student`：透過 scp 直接傳兩個檔案到遠端主機

--------------------------------------------------------

<a name="ch12.3" />
12.3 Synchronizing Files Between Systems Securely
=================================================

`-n`：Dry run，不會真的執行
`-a`：`-r` + `-l` + `-p` + `-t` + `-g` + `-o` + `-D`
`-H`：保留 Hard Link
`-A`：保留 ACLs 設定
`-X`：保留 SELinux context 設定

```bash
$ sudo rm -rf /tmp/*

# 同步目錄下所有檔案(只會有一個 log 目錄)
$ sudo rsync -av /var/log /tmp

# 同步目錄下所有檔案(會跑出很多檔案 & 目錄)
$ sudo rsync -av /var/log/ /tmp
```
