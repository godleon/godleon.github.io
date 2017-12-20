---
layout: post
title:  "[RHCE7] RH134 Chapter 11. Accessing Network Storage with Network File System (NFS) 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 11. Accessing Network Storage with Network File System (NFS) 留下的內容"
date: 2016-05-13 04:45:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

11.1 Mounting Network Storage with NFS
======================================

## 11.1.1 Manualling mouting and unmouting NFS shares

### NFS 4.0 之前：

- NFS 啟動時會向 rpcbind 註冊

- NFS 的 port 是 rpcbind(TCP 111) 所配發的

- rpcbind restart，NFS 也要跟著 restart

- 查詢遠端主機開放的目錄：`sudo showmount -e [remote_host_name or IP]`

- 掛載：`sudo mount -t nfs [remote_host_name or IP]:/content /mnt`


### NFS 4.0：

- 固定使用 TCP port 2049

- 無法使用 **showmount**，因此必須預先知道開放的目錄

- 同時掛載所有開放的目錄：`mount -t nfs [remote_host_name or IP]:/ /mnt` (確定路徑也可以 mount 特定目錄)

> NFSv2, NFSv3, NFSv4 預設是同時開啟的

RHEL 7 預設會使用 NFSv4，若不支援才會往下降；NFSv4 使用 TCP，舊版本則會用 TCP or UDP

掛載 NFS 方式：

- 手動下命令

- 加到 `/etc/fstab` 中：`[remote_host_name]:/content  /mnt  nfs   default,sync,sec=xxx   0 0`

> 網路磁碟機建議使用 sync 選項，確保資料完整性 (另外了解 soft & hard 的差異)

> 使用 fsck 檢查的設備，必須是 umount or readonly 的狀態所，所出來的結果才是正確的

## 11.1.2 Security methods

- `none`：client 匿名存取(送來的身份會被忽略)，身份全部都會轉成 **nfsnobody**

- `sys`：client 存取時會把身份送進來(例如：UID 1000)，NFS server 會找到對應的身份 & 權限 (**<font color='red'>預設值</font>**，但其實不安全)

- `krb5`：使用 kerberos ticket，但通訊時以明碼傳遞

- `krb5i`：同上，但加上完整性驗證，可判斷傳輸的資料是否被修改

- `krb5p`：同上，但傳輸的資料會被加密

> 若要讓 NFS 可以向 Kerberos 進行驗證，也同時需要有 `nfs-secure` 服務(在 `nfs-utils` 套件中)

> 使用 `klist` 可以查詢目前所擁有的 kerberos ticket

-------------------------------------------------------------------------

Practice: Mounting and Unmounting NFS
=====================================

##  目前已存在設定

1. server1 分享 `/shares/manual` & `/shares/public` 兩個目錄

2. desktop1 掛載點為 `/mnt/manual` & `/mnt/public`

3. `public` 分享目錄需要使用 `krb5p` 認證，`manual` 分享目錄則是使用 `sys` 認證

4. `krb5.keytab` 位置位於 `http://classroom.example.com/pub/keytabs/desktop1.keytab`

## 實作過程

1、取得 krb5.keytab

```bash
$ sudo wget -O /etc/krb5.keytab http://classroom.example.com/pub/keytabs/desktop1.keytab
$ ls -lZ /etc/krb5.keytab
-rw-r--r--. root root unconfined_u:object_r:krb5_keytab_t:s0 /etc/krb5.keytab
```

2、啟動 `nfs-secure.service`

```bash
$ sudo systemctl enable nfs-secure.service
ln -s '/usr/lib/systemd/system/nfs-secure.service' '/etc/systemd/system/nfs.target.wants/nfs-secure.service'
[student@desktop0 ~]$ sudo systemctl restart nfs-secure.service
```

3、建立目錄並設定掛載資訊

```bash
$ echo "server1:/shares/public /mnt/public nfs sec=krb5p,sync 0 0" | sudo tee --append /etc/fstab

$ echo "server1:/shares/manual /mnt/manual nfs sec=sys,sync 0 0" | sudo tee --append /etc/fstab

$ sudo mount -a
$ $ sudo df -hT
Filesystem             Type      Size  Used Avail Use% Mounted on
/dev/vda1              xfs        10G  3.1G  7.0G  31% /
devtmpfs               devtmpfs  906M     0  906M   0% /dev
tmpfs                  tmpfs     921M   80K  921M   1% /dev/shm
tmpfs                  tmpfs     921M   17M  904M   2% /run
tmpfs                  tmpfs     921M     0  921M   0% /sys/fs/cgroup
server1:/shares/public nfs4       10G  3.1G  7.0G  31% /mnt/public
server1:/shares/manual nfs4       10G  3.1G  7.0G  31% /mnt/manual
```

-------------------------------------------------------------------------

11.2 Automounting Network Storage with NFS
==========================================

需要安裝套件 `autofs`，會在系統中提供一個 autofs system service

**優點：**

- 使用者不需要 root 權限來執行 mount & unmount 指令

- automounter 的設定對所有使用者皆有效，主要是著重在存取權限的調整

- 不會與 NFS server 一直持續保持連線，節省網路 & 系統資源

- 與原本 mount 所使用的設定參數相同

- 提供 direct & indirect 兩種目錄 mapping 模式，以提供 mount point 一定程度的彈性

- indirect 模式下，目錄會被自動生成，降低手動的需求

- automounter 其實可以用在 NFS 之外的其他檔案系統上

## direct-map

目錄 `/etc/auto.master.d`，檔名不拘，副檔名必須是 `.autofs`

`/etc/sysconfig/autofs` 中有 `TIMEOUT` 參數可以用，表示離開目錄多久會自動 unmount

自動掛載光碟機：`/mnt/cdrom   -ro,iso9660   :/dev/sr0`

## indirect-map

掛載的目錄都會在同一個目錄下

可用萬用字元

## 掛載方式

### 1、安裝套件

首先要安裝 `autofs` 套件

### 2、編輯 master-map 檔案

master-map 檔案名稱必須是 `.autofs` 結尾，並存放於 `/etc/auto.master.d` 目錄中，以下舉例：

```bash
$ sudo cat /etc/auto.master.d/demo.autofs
# indirect-map，並以 "/share" 目錄作為掛載目錄的基底，詳細設定檔內容位於 /etc/auto.demo 檔案中
/shares   /etc/auto.demo
# direct-map，以 "/-" 作為掛載基底，詳細設定檔內容位於 /etc/auto.direct 檔案中
/-        /etc/auto.direct
```

### 3、設定 mapping 檔案

mapping 檔案則是要撰寫詳細的設定內容：

`/etc/auto.demo` indirect-map 設定檔內容：

```bash
# 使用本地目錄 /shares/work，掛載遠端目錄 serverX:/shares/work
work  -rw,sync  serverX:/shares/work
# 使用本地目錄 /shares，掛載遠端目錄 serverX:/shares 下的所有目錄(會一一取得對應)
*     -rw,sync  serverX:/shares/&
```

> indirect-map 設定使用的是相對路徑的設計概念

`/etc/auto.direct` direct-map 設定檔內容：

```bash
# 使用本地目錄 /mnt/docs，掛載遠端目錄 serverX:/shares/docs
/mnt/docs   -rw,sync  serverX:/shares/docs
```

> 由此可看出 direct-map 就是很明確的指定完整路徑

### 4、啟動 autofs 服務

```bash
$ sudo systemctl enable autofs

$ sudo systemctl start autofs
```

------------------------------------------------------------------------

Practice: Automounting NFS
==========================

## 環境設定

1. 遠端主機 `server1` 使用 NFS 分享了 `/shares/{docs,work,public}` 三個目錄

2. 存取遠端主機必須使用 Kerberos 協定，使用的是 `krb5p`

3. 本地主機 `desktop1` 使用 `/shares/{docs,work}` & `/mnt/public` 目錄進行掛載

4. `krb5.keytab` 檔案可到 `http://classroom.example.com/pub/keytabs/desktop1.keytab` 下載

5. 必須要設定成永久性掛載

## 設定方式

```bash
# 安裝 autofs 套件
$ sudo yum -y install autofs

# 建立掛載用目錄
$ sudo mkdir /shares

# 取得 krb5.keytab
$ sudo wget -O /etc/krb5.keytab http://classroom.example.com/pub/keytabs/desktop0.keytab
# 驗證 SElinux context 是否正確
$ ls -lZ /etc/krb5.keytab
-rw-r--r--. root root unconfined_u:object_r:krb5_keytab_t:s0 /etc/krb5.keytab

# 編輯 master-map 檔案
$ echo "/shares   /etc/autofs.indirect" | sudo tee --append /etc/auto.master.d/practice.autofs
$ echo "/-   /etc/autofs.direct" | sudo tee --append /etc/auto.master.d/practice.autofs

# 編輯 indrect-map 檔案
$ echo "docs  -rx,sync,sec=krb5p  server1:/shares/docs" | sudo tee --append /etc/autofs.indirect
$ echo "work  -rx,sync,sec=krb5p  server1:/shares/work" | sudo tee --append /etc/autofs.indirect

# 編輯 direct-map 檔案
$ echo "/mnt/public  -rx,sync,sec=krb5p  server1:/shares/public" | sudo tee --append /etc/autofs.direct

# 啟動 nfs-secure 服務(為了進行 Kerberos 認證)
$ sudo systemctl start nfs-secure.service
$ sudo systemctl restart autofs.service

# 啟動 autofs 服務
$ sudo systemctl enable autofs.service
$ sudo systemctl start autofs.service
```

------------------------------------------------------------------------

Lab: Accessing Network Storage with Network File System(NFS)
============================================================

## 目標

1. `desktop1` 已經設定好 LDAP & Kerberos 認証，遠端主機為 `classroom.example.com`

2. 遠端主機分享了 `ldapuserX` 所有的家目錄，分享路徑為 `/home/guests` 底下

3. 掛載於本地端的 `/home/guests` 下

4. ldapuserX 登入後會位於 `/home/guests/ldapuserX` 家目錄中

## 實作過程

```bash
$ sudo mkdir /home/guests
$ sudo yum -y install autofs

# autofs 設定
$ echo "/home/guests    /etc/autofs.indirect" | sudo tee --append /etc/auto.master.d/lab.autofs
$ echo "*   -rw,sync    classroom.example.com:/home/guests/&" | sudo tee --append /etc/autofs.indirect

# 啟用服務
$ sudo systemctl enable autofs.service
$ sudo systemctl restart autofs.service

# 登入驗證
$ ssh ldapuser1@localhost
[ldapuser1@desktop1 ~]$ pwd
/home/guests/ldapuser1
```
