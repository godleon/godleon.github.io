---
layout: post
title:  "[RHCE7] RH134 Chapter 7. Managing SELinux Security 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 07. Managing SELinux Security 留下的內容"
date: 2016-05-09 05:10:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

<a name="ch7.1" />
7.1 Enabling and Monitoring Security Enhanced Linux (SELinux)
=============================================================

## 7.1.1 Basic SELinux security concepts

SELinux 為系統安全提供了多一層的保護，目的在於當系統服務被攻陷後，還有另外一層機制(object-based)可以保護使用者的資料。

![SELinux](https://lh3.googleusercontent.com/bR6HlmRdHb9TosMXahqB3BxjdVdlFnfgpHh3bYbHeOmr2xGY_tvnAd7YBE4S4UWVzO_DX56XWwh6ytRqH1V27HhfcgRVlSHI5il3oCrJDaq0CRj_SBKElh8LLtCi4oDh3X1YXTK0gRX3p39i0soYp_ILkDBUku85ao5bYgFwmADGGN7S4Hd2FCQaREgbk9SFKgXQRJgMsvpfwmCg3KBKR6yEhG9T9iiQZg2cxZdXxELuzOFLx0jjrbgn84jxtXNnkNJJlWaOKQKtq8qGReElZkvZy2F1ZaJTcJtO208xPgEtoHHbI4N9GJyGUIcQzDhQ3rlXBuPbcuo0TT_W1yULLmLxA-uGR_JnA4cm9wv16tJZs3zMdeUKsFNxALBFZptxtOkOcpi1jopCJUEvCw4NAHhExPVHVHkVdFS2a-KUjRMBRyrFX-sa-xC5Ey3l7dbgKgjHjYta11VozADz7WIUV1FUVkR2tUmdXv6_XJVwVpclaD2PXnd7tHbOmM8Mnf4mwMP4E6yjjWiuTk9YVpoGx_2_8Nyw3xzzJ9CHYJy-WVPgSI1Q-m0Co6ZcR2bK_6p02bE8h9nft0354PiugpLIqV6dKQ0SW6g=w888-h225-no)

在傳統架構下，當 web server 被攻陷後，駭客取得 **apache** 使用者的權限，就可以自由地存取 **/var/www/html**, **/tmp**, **/var/tmp** 等目錄；但若是在 SELinux 的限制下，就僅有 **/var/www/html** 可以存取。

從上面就不難看出，SELinux 定義了特定服務可以存取的特定 file、directory、port .... 等資訊，用以限縮 service owner 可以存取檔案的範圍；而方法就在於每一個 file, directory, process, port 都有所謂的 `SELinux context`。

而 context 大概長這樣 => `system_u:object_r:httpd_sys_content_t:s0`

以 **:** 作為分隔，分別是 user(system_u), role(object_r), type(httpd_sys_content_t), sensitvity(s0)；而 SELinux rule 的制定則是以 type 為主來設計的。

舉例來說，與 Apache 相關的檔案位於 **/var/www/html** 中，而這裡檔案的 context type 則為 `httpd_sys_content_t` or `httpd_t`，與 Apache 服務運行相關的檔案則為 `http_port_t`，這些 context type　的檔案都可以讓 Apache service 存取；但若是 **/tmp** 與 **/var/tmp** 中的檔案，其 context type 則為 `tmp_t`，而 Apache service 要嘗試存取時就會被拒絕。

許多指令都可以透過 `-Z` 參數取得 SELinux context 資訊，例如：`ps axZ`, `ps -ZC httpd`, `ls -Z /var/www` ... 等等。

## 7.1.2 SELinux modes

預設為 **enforcing** mode，但若是基於臨時性的需求而需要關掉 SELinux，可轉換成 **permissive** mode，此時只會有警告 & Log，但不會被安全機制阻擋，且可以 online 切換，不須 reboot；但如果要完全 disable SELinux ，就需要重開機。

> 使用 `getenforce` 指令就可以知道目前的 SELinux mode

## 7.1.3 SELinux Booleans

**SELinux Booleans** 是用來設定 SELinux Rules 是否啟用，可以用來調整 SELinux 的原始設定；若要檢視 SELinux Booleans 目前的設定值，可使用 `getsebool -a` 取得。


-------------------------------------------------------------------------------

<a name="ch7.2" />
7.2 Changing SELinux Modes
==========================

## 7.2.1 Changing the current SELinux mode

檢視目前 SELinux 狀態：

```bash
[vagrant@server tmp]$ getenforce
Permissive

# 1(enforcing), 0(permissive)
[vagrant@server tmp]$ sudo setenforce 1
[vagrant@server tmp]$ getenforce
Enforcing
```

## 7.2.2 Setting the default SELinux mode

SELinux 的設定檔位於 `/etc/selinux/config`

```bash
[vagrant@server tmp]$ cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

`permissive`：僅會 log 不會限制存取

`enforcing`：會 log & 限制存取

> SELINUXTYPE 預設為 targeted, 也許在少數狀況下會使用另外兩個

> 變更完設定後必須重新啟動，設定才會生效


-------------------------------------------------------------------------------

<a name="ch7.3" />
## 7.3 Chaging SELinux Contexts

在開始這個部分之前，需要先安裝兩個必要套件，分別是 `policycoreutils` & `policycoreutils-python`(semanage)

### 7.3.1 Initial SELinux context

在 RHEL7 中，檔案預設的 SELinux context 會由其所在的目錄所決定，新產生的檔案都會繼承目錄的 context(`vim`, `cp`, `touch` 適用)，但若是非新建的或是特別情況則不會(`mv`, `cp -a`)

```bash
# context => httpd_sys_content_t
[student@server0 ~]$ ls -Zd /var/www/html
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 /var/www/html

# 產生兩個檔案
[student@server0 ~]$ sudo touch /var/www/html/index.html
[student@server0 ~]$ sudo cp -a /tmp/rht /var/www/html/

# 新建的繼承目錄的 context，透過 cp -a 的則保留原有的 context
[student@server0 ~]$ ls -laZ /var/www/html/
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 .
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 ..
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
-rw-r--r--. root root system_u:object_r:init_tmp_t:s0  rht
```

### 7.3.2 Changing the SELinux context of a file

有兩種方式可以改變 SELinux context：

1. `chcon`：搭配 `-t` 參數指定所要變更的 context

2. `restorecon`：直接將 context 改為預設 context (根據檔案 or 目錄所在的位置而定)

```bash
# 顯示目錄(/var/www/html)的預設 SELinux context
[student@server0 ~]$ ls -laZ /var/www/html/
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 .
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 ..
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
-rw-r--r--. root root system_u:object_r:init_tmp_t:s0  rht

# 使用 chcon 變更檔案的 context
[student@server0 ~]$ sudo chcon -t tmp_t /var/www/html/index.html
[student@server0 ~]$ ls -laZ /var/www/html/
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 .
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 ..
-rw-r--r--. root root unconfined_u:object_r:tmp_t:s0   index.html
-rw-r--r--. root root system_u:object_r:init_tmp_t:s0  rht

# 使用 restorecon 直接將檔案變成為預設值
[student@server0 ~]$ sudo restorecon -vR /var/www/html
restorecon reset /var/www/html/index.html context unconfined_u:object_r:tmp_t:s0->unconfined_u:object_r:httpd_sys_content_t:s0
restorecon reset /var/www/html/rht context system_u:object_r:init_tmp_t:s0->system_u:object_r:httpd_sys_content_t:s0
[student@server0 ~]$ ls -laZ /var/www/html/
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 .
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 ..
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
-rw-r--r--. root root system_u:object_r:httpd_sys_content_t:s0 rht
```

### 7.3.3 Defining SELinux default file context rules

使用 `semanage fcontext` 可以顯示(`-l`) or 修改(`-a`) 目錄的預設 SELinux context：

```bash
# 在 / 建立新的目錄，其預設 SELinux context 為 default_t
[student@server0 ~]$ sudo mkdir /virtual
# 在此目錄建立的檔案，context 也會變成 default_t
[student@server0 ~]$ sudo touch /virtual/index.html
[student@server0 ~]$ ls -alZ /virtual
drwxr-xr-x. root root unconfined_u:object_r:default_t:s0 .
drwxr-xr-x. root root system_u:object_r:root_t:s0      ..
-rw-r--r--. root root unconfined_u:object_r:default_t:s0 index.html

# 使用 semanage fcontext 修改 /virtual 目錄的預設 SELinux context
[student@server0 ~]$ sudo semanage fcontext -a -t httpd_sys_content_t '/virtual(/.*)?'

# 透過 restorecon 將 /virtual 目錄下的檔案還原為預設值
[student@server0 ~]$ sudo restorecon -Rv /virtual
restorecon reset /virtual context unconfined_u:object_r:default_t:s0->unconfined_u:object_r:httpd_sys_content_t:s0
restorecon reset /virtual/index.html context unconfined_u:object_r:default_t:s0->unconfined_u:object_r:httpd_sys_content_t:s0
[student@server0 ~]$ ls -alZ /virtual
drwxr-xr-x. root root unconfined_u:object_r:httpd_sys_content_t:s0 .
drwxr-xr-x. root root system_u:object_r:root_t:s0      ..
-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html
```


-------------------------------------------------------------------------------

<a name="ch7.3" />
7.4 Chaging SELinux Booleans
============================

安裝 `selinux-policy-devel` 套件可取得與 SELinux Booleans 相關的說明資訊，位於 **selinux(8)**，可使用 `man -k _selinux` 來查詢目前系統中存在的文件。

SELinux Booleans 是用來決定 rule 是否啟用的設定值，可透過 `getsebool` & `setsebool` 兩個指令來設定：

```bash
# 取得 SELinux Booleans 資訊('-a' 參數表示顯示全木)
[student@server0 ~]$ getsebool -a | grep httpd_enable_homedir
httpd_enable_homedirs --> off
[student@server0 ~]$ getsebool httpd_enable_homedirs
httpd_enable_homedirs --> off

# 修改 SELinux Boolean
[student@server0 ~]$ sudo setsebool httpd_enable_homedirs on

# 透過 'semanage boolean -l' 可以查詢 SELinux Boolean 是否永久被更改(第二個結果)
[student@server0 ~]$ sudo semanage boolean -l | grep httpd_enable_homedirs
httpd_enable_homedirs          (on   ,  off)  Allow httpd to read home directories
[student@server0 ~]$ getsebool -a | grep httpd_enable_homedir
httpd_enable_homedirs --> on

# 透過 '-P' 參數永久變更 SELinux Boolean 的設定
[student@server0 ~]$ sudo setsebool -P httpd_enable_homedirs on
[student@server0 ~]$ getsebool -a | grep httpd_enable_homedir
httpd_enable_homedirs --> on
[student@server0 ~]$ sudo semanage boolean -l | grep httpd_enable_homedirs
httpd_enable_homedirs          (on   ,   on)  Allow httpd to read home directories
```


-------------------------------------------------------------------------------

<a name="ch7.3" />
7.5 Troublshooting SELinux
==========================

## 7.5.1 Troubleshooting SELinux issues

關於 SELinux 會造成的 issue，大概會有幾個原因 & 方向可以思考：

1. 服務是否有特定目錄 or 檔案的存取權限

2. 可能是錯誤的 file context 造成的，此時可用 `restorecon` 解決

3. 可能是太嚴格的 SELinux Boolean 設定所造成(例如 `ftpd_anon_write` 限制了匿名使用者對 FTP 服務的存取)

4. 可能是 SELinux policy 的 bug 所造成

## 7.5.2 Monitoring SELinux violations

套件 `setroubleshoot-server` 必須安裝，才有辦法紀錄 SELinux 所產生的相關 log 資訊(存在於 **/var/log/audit/audit.log**)。

- `sealert -l UUID`：檢視指定 UUID 的報告

- `sealert -a /var/log/audit/audit.log`：檢視 log 中所有的稽核報告

```bash
[student@server0 ~]$ sudo systemctl start httpd

# 在 /root 下新增檔案，並移到 /var/www/html 目錄
[student@server0 ~]$ sudo touch /root/file3
[student@server0 ~]$ sudo mv /root/file3 /var/www/html/

# 嘗試透過瀏覽器存取 file3
[student@server0 ~]$ curl http://localhost/file3
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don\'t have permission to access /file3
on this server.</p>
</body></html>

# 從 /var/log/audit/audit.log 中尋找相關的錯誤資訊
[student@server0 ~]$ sudo tail /var/log/audit/audit.log
......
type=AVC msg=audit(1462720907.786:517): avc:  denied  { getattr } for  pid=1754 comm="httpd" path="/var/www/html/file3" dev="vda1" ino=8846767 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file
type=SYSCALL msg=audit(1462720907.786:517): arch=c000003e syscall=4 success=no exit=-13 a0=7fd102869b48 a1=7fff0049e4f0 a2=7fff0049e4f0 a3=7fd0f7202752 items=0 ppid=1751 pid=1754 auid=4294967295 uid=48 gid=48 euid=48 suid=48 fsuid=48 egid=48 sgid=48 fsgid=48 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1462720907.786:518): avc:  denied  { getattr } for  pid=1754 comm="httpd" path="/var/www/html/file3" dev="vda1" ino=8846767 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file
type=SYSCALL msg=audit(1462720907.786:518): arch=c000003e syscall=6 success=no exit=-13 a0=7fd102869c18 a1=7fff0049e4f0 a2=7fff0049e4f0 a3=0 items=0 ppid=1751 pid=1754 auid=4294967295 uid=48 gid=48 euid=48 suid=48 fsuid=48 egid=48 sgid=48 fsgid=48 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
.......

# 找到 SELinux 相關的 UUID
[student@server0 ~]$ sudo tail -50 /var/log/messages
.....
May  8 11:21:48 localhost setroubleshoot: Plugin Exception restorecon_source
May  8 11:21:48 localhost setroubleshoot: SELinux is preventing /usr/sbin/httpd from getattr access on the file . For complete SELinux messages. run sealert -l 26a423d9-3dbd-413b-8048-3e7abec01df1
May  8 11:21:48 localhost python: SELinux is preventing /usr/sbin/httpd from getattr access on the file .
.....

# 透過 sealert -l 檢視詳細的報告內容
[student@server0 ~]$ sudo sealert -l 26a423d9-3dbd-413b-8048-3e7abec01df1
SELinux is preventing /usr/sbin/httpd from getattr access on the file .

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that httpd should be allowed getattr access on the  file by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# grep httpd /var/log/audit/audit.log | audit2allow -M mypol
# semodule -i mypol.pp


Additional Information:
Source Context                system_u:system_r:httpd_t:s0
Target Context                unconfined_u:object_r:admin_home_t:s0
Target Objects                 [ file ]
Source                        httpd
Source Path                   /usr/sbin/httpd
Port                          <Unknown>
Host                          localhost
Source RPM Packages           httpd-2.4.6-17.el7.x86_64
Target RPM Packages
Policy RPM                    selinux-policy-3.12.1-153.el7.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     server0.example.com
Platform                      Linux server0.example.com 3.10.0-123.el7.x86_64 #1
                              SMP Mon May 5 11:16:57 EDT 2014 x86_64 x86_64
Alert Count                   1
First Seen                    2016-05-09 00:21:47 JST
Last Seen                     2016-05-09 00:21:47 JST
Local ID                      26a423d9-3dbd-413b-8048-3e7abec01df1

Raw Audit Messages
type=AVC msg=audit(1462720907.786:518): avc:  denied  { getattr } for  pid=1754 comm="httpd" path="/var/www/html/file3" dev="vda1" ino=8846767 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:admin_home_t:s0 tclass=file


type=SYSCALL msg=audit(1462720907.786:518): arch=x86_64 syscall=lstat success=no exit=EACCES a0=7fd102869c18 a1=7fff0049e4f0 a2=7fff0049e4f0 a3=0 items=0 ppid=1751 pid=1754 auid=4294967295 uid=48 gid=48 euid=48 suid=48 fsuid=48 egid=48 sgid=48 fsgid=48 tty=(none) ses=4294967295 comm=httpd exe=/usr/sbin/httpd subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: httpd,httpd_t,admin_home_t,file,getattr
```

-------------------------------------------------------------------------------

補充教材
========

## 初階篇

```bash
[vagrant@server tmp]$ id -Z
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

[vagrant@server tmp]$ ls
ABC  hosts  passwd  vagrant-shell

[vagrant@server tmp]$ ls -Z
drwxrwxrwx+ vagrant vagrant unconfined_u:object_r:user_tmp_t:s0 ABC
-rw-r--r--. vagrant vagrant unconfined_u:object_r:user_tmp_t:s0 hosts
-rw-r-xr--+ vagrant vagrant unconfined_u:object_r:user_tmp_t:s0 passwd
-rwx--x--x. vagrant vagrant unconfined_u:object_r:user_tmp_t:s0 vagrant-shell
```

以上權限對應 => system_r:object_r:var_t:s0
1. User
2. Role
3. Type
若 SELinux 設定為 enforcing，只要看 type 即可

SELinux 的 check policy 存放於 `/etc/selinux/targeted/policy/policy.29`

安裝 `setools-console` 後就可以查詢 SELinux policy

```bash
[vagrant@server tmp]$ ps auxZ | grep httpd
system_u:system_r:httpd_t:s0    root     14773  0.0  0.2 221904  4968 ?        Ss   05:44   0:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache   14774  0.0  0.1 221904  2972 ?        S    05:44   0:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache   14775  0.0  0.1 221904  2972 ?        S    05:44   0:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache   14776  0.0  0.1 221904  2972 ?        S    05:44   0:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache   14777  0.0  0.1 221904  2972 ?        S    05:44   0:00 /usr/sbin/httpd -DFOREGROUND
system_u:system_r:httpd_t:s0    apache   14778  0.0  0.1 221904  2972 ?        S    05:44   0:00 /usr/sbin/httpd -DFOREGROUND

# 可查詢來源端為 httpd_t 相關的權限設定
[vagrant@server tmp]$ sesearch -A -s httpd_t
....more

# 查詢來源端為 httpd_t，目的端為 lib_t 的權限設定
[vagrant@server tmp]$ sesearch -A -s httpd_t -t lib_t
Found 13 semantic av rules:
   allow domain base_ro_file_type : file \{ ioctl read getattr lock open \} ;
   allow domain base_ro_file_type : dir \{ ioctl read getattr lock search open \} ;
   allow domain base_ro_file_type : lnk_file \{ read getattr \} ;
.... more
```

## 進階篇

### 限制 Process 對檔案目錄的存取

```bash
[vagrant@server tmp]$ ls -ldZ /var/www/html
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 /var/www/html

[vagrant@server tmp]$ ls -lZ passwd
-rw-r--r--. vagrant vagrant unconfined_u:object_r:user_tmp_t:s0 passwd

# 變更 SELinux context user/role/type
[vagrant@server tmp]$ chcon -u system_u passwd
[vagrant@server tmp]$ chcon -r object_r passwd
[vagrant@server tmp]$ chcon -t httpd_sys_content_t passwd
[vagrant@server tmp]$ ls -lZ passwd
-rw-r--r--. vagrant vagrant system_u:object_r:httpd_sys_content_t:s0 passwd

# 同時變更 SELinux context user/role/type
[vagrant@server tmp]$ chcon system_u:object_r:httpd_sys_content_t:s0 passwd
```

`policycoreutils-python` 套件是用來尋找正確的 SELinux Context type 之用

```bash
# 尋找系統中已經存在的檔案 or 目錄的 context type
[vagrant@server tmp]$ sudo semanage fcontext -l | grep /var/www
/var/www(/.*)?                                     all files          system_u:object_r:httpd_sys_content_t:s0
/var/www(/.*)?/logs(/.*)?                          all files          system_u:object_r:httpd_log_t:s0
/var/www/[^/]*/cgi-bin(/.*)?                       all files          system_u:object_r:httpd_sys_script_exec_t:s0
/var/www/svn(/.*)?                                 all files          system_u:object_r:httpd_sys_rw_content_t:s0
/var/www/git(/.*)?                                 all files          system_u:object_r:git_content_t:s0
/var/www/perl(/.*)?                                all files          system_u:object_r:httpd_sys_script_exec_t:s0
.... more

# 參考同性質檔案 or 目錄的 context type
[vagrant@server tmp]$ ls -ldZ /var/www
drwxr-xr-x. root root system_u:object_r:httpd_sys_content_t:s0 /var/www
```

`policycoreutils` 中的 `restorecon` 工具可幫助恢復成原有的標籤：例如：`restorecon -Rv /var/www/html/`

> 前提是資料庫必須要有相關資料

### 修改資料庫

```bash
[vagrant@server tmp]$ sudo mkdir /WWW
[vagrant@server tmp]$ ls -ldZ /WWW
drwxr-xr-x. root root unconfined_u:object_r:default_t:s0 /WWW

# 透過 semanage fcontext 修改目錄預設的 SELinux context
[vagrant@server tmp]$ sudo semanage fcontext -a -f a -t httpd_sys_content_t '/WWW(/.*)?'
[vagrant@server tmp]$ sudo semanage fcontext -l | grep WWW
/WWW(/.*)?                                         all files          system_u:object_r:httpd_sys_content_t:s0

# 恢復原有設定
[vagrant@server tmp]$ sudo restorecon -Rv /WWW
restorecon reset /WWW context unconfined_u:object_r:default_t:s0->unconfined_u:object_r:httpd_sys_content_t:s0
restorecon reset /WWW/aaa context unconfined_u:object_r:default_t:s0->unconfined_u:object_r:httpd_sys_content_t:s0
[vagrant@server tmp]$ ls -ldZ /WWW
drwxr-xr-x. root root unconfined_u:object_r:httpd_sys_content_t:s0 /WWW
```

```bash
[vagrant@server tmp]$ sudo cp /etc/shadow /WWW/
[vagrant@server tmp]$ ls -ldZ /WWW/
drwxr-xr-x. root root unconfined_u:object_r:httpd_sys_content_t:s0 /WWW/

[vagrant@server tmp]$ ls -ldZ /WWW/*
----------. root root unconfined_u:object_r:httpd_sys_content_t:s0 /WWW/shadow

# -a 會保留原檔案的相關 metadata，因此不會被 dest dir 的 SELinux context 複寫
[vagrant@server tmp]$ sudo cp -a /etc/shadow /WWW/shadow_a
[vagrant@server tmp]$ ls -ldZ /WWW/*
----------. root root unconfined_u:object_r:httpd_sys_content_t:s0 /WWW/shadow
----------. root root system_u:object_r:shadow_t:s0    /WWW/shadow_a
```

查詢 SELinux 相關的 log 可到 /var/log/audit/audit.log 查詢


### 限制應用程式的特定功能是否能夠啟用

```bash
[vagrant@server tmp]$ sudo semanage boolean -l | grep ftp
ftp_home_dir                   (off  ,  off)  Allow ftp to home dir
ftpd_use_cifs                  (off  ,  off)  Allow ftpd to use cifs
sftpd_write_ssh_home           (off  ,  off)  Allow sftpd to write ssh home
ftpd_use_fusefs                (off  ,  off)  Allow ftpd to use fusefs
..... more

# 加了 -P 會永久儲存，沒加就只會存到記憶體中
[vagrant@server tmp]$ sudo setsebool -P ftp_home_dir 1
[vagrant@server tmp]$ sudo semanage boolean -l | grep ftp
ftp_home_dir                   (on   ,   on)  Allow ftp to home dir
```


### 限制應用程式所能夠存取的 port

```bash
[vagrant@server tmp]$ sudo semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989

# -a 增加 port, -d 刪除 port
[vagrant@server tmp]$ sudo semanage port -a -t http_port_t -p tcp 5678
[vagrant@server tmp] manage port -l | grep http_port_t
http_port_t                    tcp      5678, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

-------------------------------------------------------------------------------

Practice: Changing SELinux Booleans
===================================

啟用 web server 的使用者家目錄 web 功能

```bash
# 安裝 apache
$ sudo yum -y install httpd

# 啟動 apache server
$ sudo systemctl enable httpd.service
ln -s '/usr/lib/systemd/system/httpd.service' '/etc/systemd/system/multi-user.target.wants/httpd.service'
$ sudo systemctl start httpd.service
$ sudo systemctl status httpd.service
httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
   Active: active (running) since Mon 2016-05-16 17:55:59 JST; 5s ago
.......

# 開啟使用者 web 目錄為 ~/public_html
$ sudo sed -i 's/^\s*#\(UserDir public_html\)/    \1/g' /etc/httpd/conf.d/userdir.conf
$ sudo sed -i 's/^\s*\(UserDir\sdisabled\)/    # \1/g' /etc/httpd/conf.d/userdir.conf
$ sudo systemctl restart httpd.service

# 建立 index.html，但還是沒有權限可以存取
$ mkdir ~/public_html
$ echo "Hello Internet" | sudo tee /home/student/public_html/index.html
Hello Internet
$ chmod 711 /home/student
$ curl http://localhost/~student/index.html
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don\'t have permission to access /~student/index.html
on this server.</p>
</body></html>

# 檢視 SELinux Boolean 設定
$ getsebool -a | grep httpd | grep home
httpd_enable_homedirs --> off

# 開啟 httpd_enable_homedirs
$ sudo setsebool -P httpd_enable_homedirs on
$ getsebool -a | grep httpd | grep home
httpd_enable_homedirs --> on
$ curl http://localhost/~student/index.html
Hello Internet
```
