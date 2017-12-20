---
layout: post
title:  "[RHCE7] RH134 Chapter 06. Controlling Access to Files with Access Control Lists(ACLs) 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 06. Controlling Access to Files with Access Control Lists(ACLs) 留下的內容"
date: 2016-05-07 21:15:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---


透過 ACL 的機制，可以讓一個檔案同時有多個 Owner or Group

<a name="ch6.1" />
6.1 POSIX Access Control Lists (ACLs)
=====================================

若 partition 格式為 ext4，mount 的時候必須加上 `-o acl` 參數：(以下為範例)

```bash
$ mount -o acl /dev/vdb1 /mnt
```

> 或是在 /etc/fstab 上的參數設定加上 acl 也行

也可以透過指令 `sudo tune2fs -o user_xattr,acl /dev/vdb1` 直接把屬性加入到 partition 的 superblock 中

```bash
# 取得 ACL 資訊
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::rw-
group::rw-
other::r--

# 加入 User(user1) 權限 (使用 -m)
[vagrant@server tmp]$ setfacl -m u:user1:r-x acl_test.txt
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::rw-
user:user1:r-x
group::rw-
mask::rwx
other::r--

# 加入 User(user2 & user3) 權限 (使用 -m)
[vagrant@server tmp]$ setfacl -m u:user2:r-- acl_test.txt
[vagrant@server tmp]$ setfacl -m u:user3:rwx acl_test.txt
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::rw-
user:user1:r-x
user:user2:r--
user:user3:rwx
group::rw-
mask::rwx
other::r--

# 加入 Group(user4) 權限 (使用 -m)
[vagrant@server tmp]$ setfacl -m g:user4:r-x acl_test.txt
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::rw-
user:user1:r-x
user:user2:r--
user:user3:rwx
group::rw-
group:user4:r-x
mask::rwx
other::r--

# 移除 User(user1 & user2) 權限 (使用 -x)
[vagrant@server tmp]$ setfacl -x u:user1 acl_test.txt
[vagrant@server tmp]$ setfacl -x u:user2 acl_test.txt
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::rw-
user:user3:rwx
group::rw-
group:user4:r-x
mask::rwx
other::r--

# 移除 User(user3) & Group(user4) 權限 (使用 -x)
[vagrant@server tmp]$ setfacl -x u:user3 acl_test.txt
[vagrant@server tmp]$ setfacl -x g:user4 acl_test.txt
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::rw-
group::rw-
mask::rw-
other::r--

# 複製指定檔案的 ACL 權限到另外一個檔案
$ getfacl acl_test.txt | setfacl --set-file=- acl_clone.txt
```

## 補充 (使用 setfacl 變更檔案的傳統權限)

```bash
[vagrant@server tmp]$ ll
total 8
-rw-rw-r--+ 1 vagrant vagrant  0 Feb 23 03:09 acl_test.txt
-rwx--x--x. 1 vagrant vagrant 22 Feb 23 01:45 vagrant-shell

[vagrant@server tmp]$ setfacl -m u::rwx acl_test.txt
[vagrant@server tmp]$ setfacl -m g::rwx acl_test.txt
[vagrant@server tmp]$ setfacl -m o::rwx acl_test.txt
[vagrant@server tmp]$ ll
total 8
-rwxrwxrwx+ 1 vagrant vagrant  0 Feb 23 03:09 acl_test.txt
-rwx--x--x. 1 vagrant vagrant 22 Feb 23 01:45 vagrant-shell
```


`flags` 表示 SUID, SGID, StickyBit：(下面的範例表示檔案有 SUID 的屬性)

```bash
[vagrant@server tmp]$ getfacl /bin/passwd
getfacl: Removing leading '/' from absolute path names
# file: bin/passwd
# owner: root
# group: root
# flags: s--
user::rwx
group::r-x
other::r-x
```


## 補充(使用 setfacl 修改 mask 設定)

```bash
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::rwx
group::rwx
mask::rwx
other::rwx

# 透過 mask 可以限制 group 僅剩下 rw 的權限
[vagrant@server tmp]$ setfacl -m m::r-x acl_test.txt
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::rwx
group::rwx			#effective:r-x
mask::r-x
other::rwx
```

## 補充(已經有 ACL 的設定就不要再用 chmod)

```bash
[vagrant@server tmp]$ ll
total 8
-rwxr-xrwx+ 1 vagrant vagrant  0 Feb 23 03:09 acl_test.txt
-rwx--x--x. 1 vagrant vagrant 22 Feb 23 01:45 vagrant-shell
[vagrant@server tmp]$ chmod 123 acl_test.txt
[vagrant@server tmp]$ ll
total 8
---x-w--wx+ 1 vagrant vagrant  0 Feb 23 03:09 acl_test.txt
-rwx--x--x. 1 vagrant vagrant 22 Feb 23 01:45 vagrant-shell
[vagrant@server tmp]$ getfacl acl_test.txt
# file: acl_test.txt
# owner: vagrant
# group: vagrant
user::--x
group::rwx			#effective:-w-
mask::-w-
other::-wx
```

## 補充(若要稽核使用者使用檔案的狀況)

可使用 kernel 中的 Audit 功能，設定可參考 `/etc/audit` 目錄中的設定

## 補充(其他)

tar 打包時要包含 ACL & SELinux 的權限，要加入 `-xattr` 參數


---------------------------------------------------------------

<a name="ch6.2" />
6.2 Securing Files with ACLs
============================

透過 `-b` 參數可回復沒有 ACL 權限設定的狀態，例如 `setfacl -b filename`

## Setting an explicit ACL mask

```bash
[vagrant@server tmp]$ setfacl -m u:user1:rwx passwd
[vagrant@server tmp]$ setfacl -m u:user2:r-x passwd
[vagrant@server tmp]$ setfacl -m g:user3:rwx passwd
[vagrant@server tmp]$ setfacl -m g:user4:r-x passwd
[vagrant@server tmp]$ getfacl passwd
# file: passwd
# owner: vagrant
# group: vagrant
user::rw-
user:user1:rwx
user:user2:r-x
group::r--
group:user3:rwx
group:user4:r-x
mask::rwx
other::r--

# 使用 mask，限定所能設定的最大權限(注意 rwx 實際只有 r-x 可用)
[vagrant@server tmp]$ setfacl -m m::r-x passwd
[vagrant@server tmp]$ getfacl passwd
# file: passwd
# owner: vagrant
# group: vagrant
user::rw-
user:user1:rwx			#effective:r-x
user:user2:r-x
group::r--
group:user3:rwx			#effective:r-x
group:user4:r-x
mask::r-x
other::r--
```

## 設定檔案建立時的預設 ACL 權限(`d`)

```bash
[vagrant@server tmp]$ mkdir ABC
[vagrant@server tmp]$ chmod 777 ABC/
[vagrant@server tmp]$ touch ABC/file.txt
[vagrant@server tmp]$ getfacl ABC/file.txt
# file: ABC/file.txt
# owner: vagrant
# group: vagrant
user::rw-
group::rw-
other::r--

[vagrant@server tmp]$ setfacl -m d:u:user1:rwx ABC/
[vagrant@server tmp]$ touch ABC/file2.txt
[vagrant@server tmp]$ getfacl ABC/file2.txt
# file: ABC/file2.txt
# owner: vagrant
# group: vagrant
user::rw-
user:user1:rwx			#effective:rw-
group::rwx			#effective:rw-
mask::rw-
other::rw-

[vagrant@server tmp]$ ls -l ABC/
total 4
-rw-rw-rw-+ 1 vagrant vagrant 0 Feb 23 03:50 file2.txt
-rw-rw-r--. 1 vagrant vagrant 0 Feb 23 03:50 file.txt
```

## 遞迴設定 ACL 權限(-R + X)

```bash
# 大寫 X => 表示 user1 對於檔案沒有 exec 的權限，但目錄則有 exec 的權限(才可瀏覽)
# -R => recusive
$ setfacl -R -m u:user1:rX /dir
```

## 移除 ACL 權限(`-x`)

```bash
$ getfacl acl_clone.txt
# file: acl_clone.txt
# owner: student
# group: student
user::rw-
user:user1:r-x
group::rw-
group:group1:rw-
group:group2:r-x
mask::rwx
other::---

# 透過 -x 參數，指定移除 user1 & group2 的權限
$ setfacl -x u:user1,g:group2 acl_clone.txt
$ getfacl acl_clone.txt
# file: acl_clone.txt
# owner: student
# group: student
user::rw-
group::rw-
group:group1:rw-
mask::rw-
other::---
```

---------------------------------------------------------------

Practice: Using ACLs to Grant and Limit Access
==============================================

實作結果：

1. 讓 **sodor** group 與 **controller** group 在 **/shares/steamies** 目錄有相同的權限，但 user **james** 則是例外沒有任何權限

```bash
# 確認 controller 的 ACL 權限
[student@server0 ~]$ sudo getfacl /shares/steamies/
getfacl: Removing leading '/' from absolute path names
# file: shares/steamies/
# owner: root
# group: controller
# flags: -s-
user::rwx
group::rwx
other::---
default:user::rwx
default:group::rwx
default:other::---

# 讓 sodor group 與 controller group 有相同的權限
[student@server0 ~]$ sudo setfacl -Rm g:sodor:rwX /shares/steamies
[student@server0 ~]$ sudo getfacl /shares/steamies/
getfacl: Removing leading '/' from absolute path names
# file: shares/steamies/
# owner: root
# group: controller
# flags: -s-
user::rwx
group::rwx
group:sodor:rwx
mask::rwx
other::---
default:user::rwx
default:group::rwx
default:other::---

# 讓 user james 沒有任何權限
[student@server0 ~]$ sudo setfacl -Rm u:james:- /shares/steamies
[student@server0 ~]$ sudo getfacl /shares/steamies/
getfacl: Removing leading '/' from absolute path names
# file: shares/steamies/
# owner: root
# group: controller
# flags: -s-
user::rwx
user:james:---
group::rwx
group:sodor:rwx
mask::rwx
other::---
default:user::rwx
default:group::rwx
default:other::---
```

2. **/shares/steamies** 目錄底下的現有檔案都需要設定成上面的 ACL 權限

> 因為上面已經使用 `-R` 參數，因此這個部分已經完成

3. 新增的目錄 & 檔案也會有相同的 ACL 權限

表示 **sodor** group 還是會有 rwx 權限，**james** user 也是同樣沒權限：

```bash
[student@server0 ~]$ sudo setfacl -m d:g:sodor:rwx /shares/steamies
[student@server0 ~]$ sudo setfacl -m d:u:james:- /shares/steamies
[student@server0 ~]$ sudo getfacl /shares/steamies
getfacl: Removing leading '/' from absolute path names
# file: shares/steamies
# owner: root
# group: controller
# flags: -s-
user::rwx
user:james:---
group::rwx
group:sodor:rwx
mask::rwx
other::---
default:user::rwx
default:user:james:---
default:group::rwx
default:group:sodor:rwx
default:mask::rwx
default:other::---
```
