---
layout: post
title:  "[RHCE7] RH134 Chapter 12. Accessing Network Storage with SMB 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 12. Accessing Network Storage with SMB 留下的內容"
date: 2016-05-13 04:45:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---


12.1 Accessing Network Storage with SMB
=======================================

## Manually mounting and unmouting SMB file systems

要裝 `cifs-utils` & `samba-client` 兩個套件

### 查詢

查詢 server 分享了什麼資源(匿名)：`smbclient -L //server0`

查詢 server 分享了什麼資源：`smbclient -L //server0 -U [UserName] -W [DomainName]` (`-U` 指定使用者，`-W` 指定網域)

### 掛載

匿名掛載：`sudo mount -t cifs -o guest //server0/public /mnt`

指定使用者掛載：`sudo mount -t cifs -o username=[UserName],password=[Password],workgroup=[DomainName] //server0/public /mnt` (若不加 `password` 會被要求手動輸入密碼)

## 透過 /etc/fstab 掛載

加入 `/etc/fstab` 中：`//server0/pubilic  /mnt  cifs  defaults,username=[UserName],password=[Password],workgroup=[DomainName] 0 0`

## 指定 credential 掛載

以 credential 的方式設定：`//server0/pubilic  /mnt  cifs  defaults,credentials=/root/smb.txt 0 0`

`/root/smb.txt` 的內容如下：(**only root access, chmod 600**)

```ini
username=[UserName]
password=[Password]
domain=[DomainName]
```

## Mounting SMB file systems with the automounter

除了 NFS 之外，SMB 也可以使用 automounter 來協助自動掛載，差別只有掛載參數上的不同，以下是設定範例：

- 本地目錄：`/bakerst/cases`

- 遠端目錄：`//serverX/cases`

1、安裝 `autofs` 套件

2、新增檔案 **/etc/auto.master.d/bakerst.autofs**，內容如下：

```bash
/bakerst  /etc/auto.bakerst
```

3、新增檔案 **/etc/auto.bakerst**，內容如下：

```bash
cases   -fstype=cifs,credentials=/secure/sherlock   ://serverX/cases
```

4、新增檔案 **/secure/sherlock**，內容如下：(**only root access**, permission **600**)

```ini
username=[UserName]
password=[Password]
domain=[DomainName]
```

5、啟動 autofs：`sudo systemctl enable autofs && sudo systemctl restart autofs`

---------------------------------------------------------

Practice: Mounting a SMB File System
====================================

## 目標

1. 掛載遠端目錄 `//server1/student` 到本地端 `~/work` 中

2. 連線帳號/密碼/Domain = student/student/MYGROUP

3. 永久性掛載

## 實作過程

```bash
$ sudo yum -y install cifs-utils

$ sudo bash -c 'cat << EOF > /root/student.smb
username=student
password=student
domain=MYGROUP
EOF'

$ mkdir ~/work
$ echo "//server0/student  /home/student/work  cifs  credentials=/root/student.smb  0 0" | sudo tee --append /etc/fstab
$ sudo mount -a

# 驗證連線結果
[student@desktop0 ~]$ df -hT | grep work
//server0/student cifs       10G  3.1G  7.0G  31% /home/student/work
```

----------------------------------------------------------

Lab: Accessing Network Storage with SMB
=======================================

## 目標

### 環境

1. 遠端主機：`server1`

2. DOMAIN：`MYGROUP`

3. 使用者帳號密碼：`student` / `student`

### 需求

1. 自動掛載遠端主機的 `student` 到本地端的家目錄 `/shares/work`

2. 自動掛載遠端主機的 `public` 到本地端目錄 `/shares/docs` 公開分享的目錄，允許任何人存取，權限為 `read-only`

3. 自動掛載遠端主機的 `/shares/cases` 到本地端目錄 `/shares/cases`，並限制只有 `bakerst`(GID=10221) 群組可以存取，權限為 `read-write`

4. 要設定為永久性掛載(重開機之後要依然生效)

## 實作過程

```bash
# 安裝套件
$ sudo yum -y install cifs-utils autofs

$ sudo bash -c 'cat << EOF > /root/student.smb
username=student
password=student
domain=MYGROUP
EOF'

# 建立本地掛載目錄
[student@desktop0 ~]$ sudo mkdir -p /shares/{work,docs,cases}

# 設定 autofs
[student@desktop0 ~]$ echo "/shares /etc/autofs.indirect" | sudo tee --append /etc/auto.master.d/smb.autofs
[student@desktop0 ~]$ echo "work -fstype=cifs,credentials=/root/student.smb ://server0/student" | sudo tee --append /etc/autofs.indirect
[student@desktop0 ~]$ echo "docs -fstype=cifs,guest ://server0/public" | sudo tee --append /etc/autofs.indirect
[student@desktop0 ~]$ echo "cases -fstype=cifs,credentials=/root/student.smb ://server0/bakerst" | sudo tee --append /etc/autofs.indirect

# 啟動 autofs 服務
[student@desktop0 ~]$ sudo systemctl enable autofs.service
[student@desktop0 ~]$ sudo systemctl start autofs.service

# 建立 backerst 群組，並將 student 帳號加入
[student@desktop0 ~]$ sudo groupadd -g 10221 bakerst
[student@desktop0 ~]$ sudo usermod -aG bakerst student
```
