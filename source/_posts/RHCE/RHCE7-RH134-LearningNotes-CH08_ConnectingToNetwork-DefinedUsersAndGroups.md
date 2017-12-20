---
layout: post
title:  "[RHCE7] RH134 Chapter 08. Connecting to Network-defined Users and Groups 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 08. Connecting to Network-defined Users and Groups 留下的內容"
date: 2016-05-10 04:45:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

老師補充
========

```bash
# 取得使用者資訊(跟 /etc/passwd 沒有絕對關係)
[vagrant@server tmp]$ getent passwd user1
user1:x:1001:1001::/home/user1:/bin/bash

# 取得群組資訊
[vagrant@server tmp]$ getent group user1
user1:x:1001:
```

驗證是否通過：

1. 帳號 & 密碼正確

2. 可以正確取得使用者資訊


NIS & LDAP 單獨可做到：

1. 可提供使用者資訊

2. 可提供帳號密碼的驗證方法

> 但 Kerberos 僅能提供帳號密碼驗證，無法提供使用者資訊，因此一般會與 LDAP 搭配來解決提供使用者資訊的問題，Windows AD 目前就是依照這種方式達成，安全性提高，且提供了 SSO 的能力)

> 目前使用 LDAP + Kerberos 架構的產品有 Windows AD, SAMBA 4.x, RedHat IDM(IPA)

-------------------------------------------------------------------------

8.1 Using Identity Management Services
======================================

## 8.1.1 User information and authentication services

中央認證系統會包含兩個部分：

### 1. Account information

此部分用來存放使用者帳號的資訊，以及使用者權限等相關資訊。(例如：**/etc/passwd**)

### 2. Authentication information

此部分的目的則是驗證使用者是否為其所宣稱的使用者，因此會包含密碼資訊，但這些密碼資訊通常會透過密碼學的技術輔助來加密，避免以明碼的方式呈現。(例如：**/etc/shadow**)

### 其他補充

Kerberos 僅能提供安全的驗證功能，無法提供使用者資訊；若需要使用者資訊，需要搭配 LDAP/NIS .... 等服務來提供。

目前 LDAP + Kerberos 的選擇有 Windows AD / SAMBA 4.x / RedHat IDM ... 等。

透過調整 `/etc/nsswitch.conf`，可以調整驗證的順序

```bash
[vagrant@localhost ~]$ cat /etc/nsswitch.conf | grep '^[^#]'
passwd:     files sss
shadow:     files sss
group:      files sss
hosts:      files dns
bootparams: nisplus [NOTFOUND=return] files
ethers:     files
netmasks:   files
networks:   files
protocols:  files
rpc:        files
services:   files sss
netgroup:   files sss
publickey:  nisplus
automount:  files
aliases:    files nisplus
```

> NSS 設定中的 sss 表示指向系統中的 sssd(security system daemon, for cache)，即使在沒網路的狀態下也還是可以進行認証

RedHat 設計的工具大多以 `system-config` 開頭：

```bash
[vagrant@localhost ~]$ system-config-
system-config-authentication  system-config-firewall-tui    system-config-kickstart       system-config-printer-applet  
system-config-date            system-config-kdump           system-config-language        system-config-users           
system-config-firewall        system-config-keyboard        system-config-printer
```

安裝 `system-config-authentication` 相關的工具：`sudo yum -y install authconfig-gtk sssd krb5-workstation`

若 LDAP 過程中要加密，LDAP server 設定時就不能使用 IP，必須使用 domain name，否則 CA 憑證檢查不會過

> 設定 Kerneros 要把 `Use DNS to resolve hosts to realm` & `Use DNS to locate KDCs for realm` 兩個選項拿掉

## 8.1.2 Attaching a system to centralized LDAP and Kerberos servers

設定時建議安裝套件：`authconfig-gtk`, `sssd`, `krb5-workstation`

### Authconfig

RHCE 中的示範以 LDAP + Kerberos 為組合，有一些設定資訊是必須了解的：

- **/etc/ldap.conf**：提供 LDAP 服務的 server 的相關設定

- **/etc/krb5.conf**：Kerberos 的相關設定

- **/etc/sssd/sssd.conf**：system security services daemon(sssd) 的設定，負責用來取得 & 快取使用者的認證資訊

- **/etc/nsswitch.conf**：用來設定認證使用者所要使用的服務 or 系統

- **/etc/pam.d**：此目錄包含了提供了不同驗證機制的模組

- /etc/openldap/cacerts：儲存 certificate 用

> 認證設定的過程建議使用 `authconfig-gtk`(提供 **system-config-authentication** 指令) or `authconfig-tui` 套件來協助，有圖形介面較為容易設定；不建議使用 `authconfig` 套件

### LDAP

LDAP 帳號描述範例：`cn=kevin,ou=sales,ou=tp,ou=tw,dc=example,dc=com` (DN, Distinguish Name)

LDAP 認証可用 SSL/TLS 加密

**要使用 LDAP 服務，要提供以下必要資訊**：

1. LDAP server hostname

2. base DN(即是上面的 `ou=tp,ou=tw,dc=example,dc=com`，依據要認證到的層級而定)

3. 若要加密則必須 CA

### Kerberos

兩大訴求：

1. SSO (Single Sign On)

2. 安全性高(驗證過程中，帳號密碼不會在網路上傳遞)

密碼儲存 & 認証方式：

1. Server 儲存 hash 過後的密碼

2. client 認証時，會將輸入的密碼 hash 後的結果作為加密帳號資訊的對稱式金鑰，將資訊(user_account,time,....)加密後傳到 server 驗證

3. server 收到加密結果後，會拿資料庫中的 hash value 並進行解密，若是有符合則通過

4. 為防止重送攻擊，被加密的資料有包含時間資訊(因此系統時間很重要)

5. 假設驗證通過，server 會發送一個 ticket 給 client，之後 client 要存取其他 resource 只要提供 account & ticket 即可，resource owner 會自動去找 KDC(Kerberos Key Distribution Center) 驗證 ticket 是否有效

**要使用 Kerberos 服務，要提供以下必要資訊**：

1. Kerberos realm，類似 domain name 的概念

2. 至少一個 key distribution center(KDC)，即為 Kerberos hostname

3. Admin server hostname(用來協助使用者密碼及其他資訊變更之用)，通常與 KDC 會是相同一台機器


## 8.1.3 Attaching a System to an IPA Server

在 RHEL 7 中提供了方便工具設定，安裝 `ipa-client` 套件即可。

以 non-interactive 的方式設定 IPA server 認証：`sudo ipa-client-install --domain=serverX.example.com --no-ntp --mkhomedir -p admin -w redhat123 -U`

-----------------------------------------------------------

Practice: Connecting to a Central LDAP and Kerberos Server
==========================================================

設定 LDAP + Kerberos 驗證的步驟：

1、安裝相關套件：`sudo yum -y install sssd authconfig-gtk krb5-workstation`

2、使用圖形工具進行設定(設定如下)

![system-config-authentication](https://lh3.googleusercontent.com/yrIOq6ZotETj8rEMxH3hnoBqJzUmaF4A_aYQQlvMTuqU6u2-xBCA7aUMIkVBUi_uwjiSOcEr5V_TXrlv0jYDX_ikqhPjZVWPiXo5XKIoWUovuYNtbyPu4nibuVU1sZ150Y43kME_1dExiLtYF3cowhoHA_ex4CbHBDGULMGkfPJQnJkoWM_PVlb-xk1ENpKjElCw0If4VUMnyk6QsSjWTeKrUt6gl8A9Szi-DEWgh2FuhnXFFfL2BOlUmcvLeUDDDu7cT1v09QRqQI1usK3bDKIjrmNUaSAYQ8hPoxDqZaT0-KMycPnDwc8SWJXmm4En8uQMHSGMqFKlx1LmJqufLKZMaFZgF3rM1BN8pCYFxkUkws2nNuG-tmzHi0yxVq7T0vYFerWX66XTvhrOHiReMjLKp3IEm7ue4-AUNY9bcwlPgyjoOBJXId1vT4AgyHeR7sivsMFvwfw2jTwZdEEp2723scmSxiYoO0baKp7rrSnls11b9SI_ZIPd7gAD_42AqnstDCEWWsKzXCpGLXuAXFFjk14lO5g0MHtCd73J2aJmsz8QN_rMZK_Hd9YW44VJ6AEQF3rB4Sxq09bT6q8HPF9ht-ak_qg=w474-h710-no)

3、驗證設定結果：`sudo getent passwd ldapuser1` or `ssh ldapuser1@server1`(密碼 `kerberos`)

-----------------------------------------------------------

Lab: Connecting to Network-defined Users and Groups
===================================================

Lab 設計為連線到 IPA server 進行驗證，步驟如下：

1、安裝相關套件：`sudo yum -y install ipa-client`

2、設定連線：`sudo ipa-client-install --domain=server1.example.com --no-ntp --mkhomedir`

```bash
$ sudo ipa-client-install --domain=server0.example.com --no-ntp --mkhomedir

Discovery was successful!
Hostname: desktop1.example.com
Realm: SERVER0.EXAMPLE.COM
DNS Domain: server1.example.com
IPA Server: server1.example.com
BaseDN: dc=server1,dc=example,dc=com

Continue to configure the system with these values? [no]: yes
User authorized to enroll computers: admin
Synchronizing time with KDC...
Unable to sync time with IPA NTP server, assuming the time is in sync. Please check that 123 UDP port is opened.
Password for admin@SERVER1.EXAMPLE.COM:
Successfully retrieved CA cert
    Subject:     CN=Certificate Authority,O=SERVER1.EXAMPLE.COM
    Issuer:      CN=Certificate Authority,O=SERVER1.EXAMPLE.COM
    Valid From:  Tue May 17 20:31:07 2016 UTC
    Valid Until: Sat May 17 20:31:07 2036 UTC

Enrolled in IPA realm SERVER1.EXAMPLE.COM
Created /etc/ipa/default.conf
New SSSD config will be created
Configured /etc/sssd/sssd.conf
Configured /etc/krb5.conf for IPA realm SERVER1.EXAMPLE.COM
trying https://server1.example.com/ipa/xml
Forwarding 'ping' to server 'https://server1.example.com/ipa/xml'
Forwarding 'env' to server 'https://server1.example.com/ipa/xml'
Adding SSH public key from /etc/ssh/ssh_host_rsa_key.pub
Adding SSH public key from /etc/ssh/ssh_host_ecdsa_key.pub
Forwarding 'host_mod' to server 'https://server1.example.com/ipa/xml'
Could not update DNS SSHFP records.
SSSD enabled
Configured /etc/openldap/ldap.conf
Configured /etc/ssh/ssh_config
Configured /etc/ssh/sshd_config
Client configuration complete.
```

3、驗證設定結果：`sudo getent passwd ipauser` & `ssh ipauser@localhost`(原本密碼為 `password`，改為 `redhat123`)

4、程式驗證：`lab ipaclient grade` (@desktop)
