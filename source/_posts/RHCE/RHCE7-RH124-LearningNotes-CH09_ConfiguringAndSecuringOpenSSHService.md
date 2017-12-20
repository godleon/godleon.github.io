---
layout: post
title:  "[RHCE] RH124 Chapter 9 Controlling and Securing OpenSSH Service 學習筆記"
description: "此文章記錄學習 RHCE RH124 Chapter 9 Controlling and Securing OpenSSH Service 留下的內容"
date: 2016-04-20 04:50:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---

9.1 Accessing the Remote Command Line with SSH
==============================================

## 9.1.1 SSH Host Keys

server 會將 public key copy 送到 client 端，有兩個功能：
1. 用來加密 ssh connection 用
2. 用來驗證 server

- server public key 會存在於 client 端的 `~/.ssh/known_hosts` 檔案中

- server 端會把 key pair 儲存在 `/etc/ssh/ssh_host_key*` 目錄下

- 當 client 透過 ssh 連到 server 時，會把 server 的 public 儲存在 **<font color='red'>~/.ssh/known_hosts</font>** 內，且每次連線都會檢查，若發現內容不會就會警告且中斷連線!

-------------------------------------------------

9.2 Conguring SSH Key-based Authentication
==========================================

`ssh-copy-id`：上傳 <font color='red'>**~/.ssh/id_rsa.pub**</font> 到 remote server 的 <font color='red'>**~user/.ssh/authorized_keys**</font> 檔案中：

```bash
# 將指定的 public key 加入到 remote server 的 student 帳號下
[student@server0 ~]$ ssh-copy-id -i ~/.ssh/id_rsa.pub student@172.25.0.10
# 將指定的 public key 加入到 remote server 的 root 帳號下
[student@server0 ~]$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.25.0.10
```

> 若沒有使用 `-i` 指定 public key 位置，則就預設為 `~/.ssh/id_rsa.pub`


若要使用 key-based 認證但又希望在 private key 上加密碼，並達成 password-less 的效果時：

``` bash
# 產生一個新的 ssh agent 並將 private key 驗證加入
$ ssh-agent bash
$ ssh-add
```

-------------------------------------------------

9.3 Customize SSH Service Configuration
=======================================

修改 `/etc/ssh/sshd_config` 中，調整使用者登入方式：

1. `PermitRootLogin no`：禁止 root 使用 ssh 登入

2. `PermitRootLogin without-password`：root 只能透過 key-based 的方式登入

3. `PasswordAuthentication no`：關閉密碼登入功能(只能透過 key-based 的方式登入)

> 要重新 reload sshd.service 讓設定變更生效 (`sudo systemctl restart sshd.service`)
