---
layout: post
title:  "[RHCE] RH124 Chapter 13 Installing and Updating Software Packages 學習筆記"
description: "此文章記錄學習 RHCE RH124 Chapter 13 Installing and Updating Software Packages 留下的內容"
date: 2016-04-28 04:50:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---

<a name="ch13.2" />
13.2 RPM Software Packages and YUM
==================================

| 功能 | yum | rpm |
|-----|-----|-----|
| 查詢 | yum list &#124; grep <font color='blue'>*KEYWORD*</font> <br />yum search <font color='blue'>*KEYWORD*</font> <br />yum info <font color='blue'>*PACKAGE_NAME*</font> <br />yum provides <font color='blue'>*FILE_PATH_NAME*</font> | rpm -qa &#124; grep <font color='blue'>*KEYWORD*</font> <br />rpm [ -qi &#124; -ql &#124; -qc &#124; -gd &#124; -q --scripts &#124; -q --changelog ] <font color='blue'>*PACKAGE_NAME*</font> <br /> rpm -qf <font color='blue'>*FILE_PATH*</font> |
| 查詢(Group) | yum groups [ list &#124; info ] | rpm [ -qpi &#124; -qpl &#124; -qpc &#124; -qpd &#124; -qp --scripts &#124; -qp --changelog] <font color='blue'>*PACKAGE_NAME*</font> |
| 安裝 | yum -y [group] install <font color='blue'>*PACKAGE_NAME*</font> | rpm -ivh <font color='blue'>*PACKAGE_NAME*</font> <br />yum -y localinstall <font color='blue'>*PACKAGE_NAME*</font> |
| 更新 | yum -y update <font color='blue'>*PACKAGE_NAME*</font> | rpm -Uvh <font color='blue'>*PACKAGE_NAME*</font> |
| 移除 | yum -y [group] remove <font color='blue'>*PACKAGE_NAME*</font> | rpm -e <font color='blue'>*PACKAGE_NAME*</font> |

---------------------------------------------------------------

<a name="ch13.3" />
13.3 Managing Software Updates with yum
=======================================

- `sudo yum group install "Development Tools"`：安裝整包 Development Tools

- `yum list kernel`：列出 kernel 清單 (包含已經安裝 & 可安裝的)

- `uname -r`：列出 kenal 版本

- `uname -a`：列出 kernel 詳細資訊

- `sudo yum history`：檢視 yum 歷程記錄

- `sudo yum undo 5`：取消 ID=5 所紀錄的 yum 工作

---------------------------------------------------------------

<a name="ch13.4" />
13.4 Enabling yum Software Repositories
=======================================

`/etc/yum.repos.d/*.repo`：此目錄內的附檔名必須都是 **repo**

```ini
[ID]
name=
baseurl= YUM server 上的容器
enabled=
gpgcheck=
```

常用指令：

- `yum repolist all`：列出目前所有 repository

- `sudo yum-config-manager --disable rhel_dvd`：停用 "rhel_dvd" repository

- `sudo yum-config-manager --add-repo="http://content.example.com/rhel7.0/x86_64/rht/"`：直接指定路徑增加 repository

- `sudo rpm --import http://dl.fedoraproject.org/pub/epel/RPM-GPG-KEY-EPEL-7`：加入 GPG Key

- `sudo yum -y install http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm`：透過 rpm 安裝方式加入 repository

其他：

- `EPEL`：Extra Package for Enterprise Linux

- 使用 yum-config-manager 搭配 `--nogpgcheck` 表示忽略 GPG key 的檢查，可能會有安全性上的風險

### (**<font color='red'>非常重要</font>**) ===== Practice ====== (**<font color='red'>非常重要</font>**)

```bash
$ sudo yum-config-manager --add-repo="http://content.example.com/rhel7.0/x86_64/rht"
[content.example.com_rhel7.0_x86_64_rht]
name=added from: http://content.example.com/rhel7.0/x86_64/rht
baseurl=http://content.example.com/rhel7.0/x86_64/rht
enabled=1
```

編輯 **<font color='red'>/etc/yum.repo.d/errata.repo</font>** 內容如下：

```bash
[updates]
name=RedHat updates
baseurl=http://content.example.com/rhel7.0/x86_64/errata
enabled=1
gpgcheck=0
```

```bash
# 檢查詢所有的 repository (包含 enabled & disabled)
$ yum repolist all

# 停用指定 repository
$ sudo yum-config-manager --disable content.example.com_rhel7.0_x86_64_rht

# 再次確認 repository 狀態
$ yum repolist all

# 套件升級(會發現有個 kernel 的 update 來自剛剛的 errata repo)
$ sudo yum -y update
# 檢視目前系統中所有的 kernel 清單
$ yum list kernel
```

---------------------------------------------------------------

<a name="ch13.5" />
13.5 Examining RPM Package Files
================================

rpm 常用參數：

```bash
# 查詢指定套件
$ rpm -q yum
yum-3.4.3-118.el7.noarch

# 查詢指定檔案(or 目錄)屬於哪個套件
$ rpm -q -f /etc/yum.repos.d
yum-3.4.3-118.el7.noarch

# 查詢套件資訊，類似 "yum info" 的功能
$ rpm -q -i yum
Name        : yum
Version     : 3.4.3
Release     : 118.el7
Architecture: noarch
......

# 列出安裝指定套件所產生的檔案列表
$ rpm -q -l yum
.....
/etc/yum.conf
/etc/yum.repos.d

# 列出指定套件相關的文件資訊
$ rpm -q -d yum
/usr/share/doc/yum-3.4.3/AUTHORS
/usr/share/doc/yum-3.4.3/COPYING
/usr/share/doc/yum-3.4.3/ChangeLog
/usr/share/doc/yum-3.4.3/INSTALL
.......

# 列出安裝指定套件所會執行的相關 script 內容
$ rpm -q --scripts openssh-server
preinstall scriptlet (using /bin/sh):
getent group sshd >/dev/null || groupadd -g 74 -r sshd || :
.......
postinstall scriptlet (using /bin/sh):
.......
.....

# 查詢套件所包含的設定檔
$ rpm -q -c yum
/etc/logrotate.d/yum
/etc/yum.conf
/etc/yum/version-groups.conf

# 查詢指定套件的 changelog 資訊
$ rpm -q --changelog yum
```
