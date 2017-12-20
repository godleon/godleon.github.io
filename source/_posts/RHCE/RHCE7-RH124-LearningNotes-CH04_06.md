---
layout: post
title:  "[RHCE] RH124 Chapter 4~6 學習筆記"
description: "此文章記錄學習 RHCE RH124 所留下的內容，此篇包含 RH124 chapter 4~6，分別是 'Creating, Viewing, and Editing Text Files', 'Chapter 5. Managing Local Linux Users and Groups', 以及 'Controlling Access to Files with Linux File System Permissions'"
date: 2016-04-03 16:45:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH124]
---

Chapter 4. Creating, Viewing, and Editing Text Files
====================================================

## 4.1 Redirecting Output to a File or Program

### Standard input, standard output, and standard error

![Process I/O Channel](https://upload.wikimedia.org/wikipedia/commons/7/70/Stdstreams-notitle.svg)

![STDIN、STDOUT & STDERR](http://cs.ucla.edu/classes/fall08/cs111/scribe/4/FDT_diagram.JPG)

### Redirecting output to a file

| Usage | 說明 |
|-------|------|
| `&>file`  | stdout & stderr 各自輸出到相同的檔案中<br />會複寫指定檔案，若檔案不存在則建立新檔 |
| `>>file 2>&1`<p />`&>>file` | stderr 會導向變成 stdout 輸出，並附加內容於指定檔案<br />**<font color='red'>不會再有 stderr 輸出，而是全部皆為 stdout 輸出</font>** |

```bash
[vagrant@server tmp]$ find /etc -name passwd &>/tmp/save-both
[vagrant@server tmp]$ cat /tmp/save-both
....
find: ‘/etc/lvm/backup’: Permission denied
find: ‘/etc/lvm/cache’: Permission denied
/etc/passwd
find: ‘/etc/polkit-1/rules.d’: Permission denied
find: ‘/etc/polkit-1/localauthority’: Permission denied
....
/etc/pam.d/passwd
find: ‘/etc/audisp’: Permission denied
....
```

### Constructing pipelines

pipeline 並沒有對 stderr 進行處理，透過以下兩個指令可以看出差別：(看有顏色的部分)

```bash
# 僅有 stdout 的內容會被篩選
[vagrant@server tmp]$ find /etc -name passwd | grep etc

# 連同 stderr 的內容都會被篩選(因為 stderr 的內容已經導向 stdout 輸出)
[vagrant@server tmp]$ find /etc -name passwd 2>&1 | grep etc
```

![Linux Pipeline](http://civilnet.cn/book/kernel/GNU.Linux.Application.Programming/images/11.1_0.jpg)

#### tee

![tee pipeline](https://upload.wikimedia.org/wikipedia/commons/thumb/2/24/Tee.svg/400px-Tee.svg.png)

tee 的使用範例：

```bash
# -a = append
[vagrant@server ~]$ ps -f | tee -a ps_file.txt
UID        PID  PPID  C STIME TTY          TIME CMD
vagrant   3372  3371  0 14:25 pts/0    00:00:00 -bash
vagrant   3395  3372  0 14:25 pts/0    00:00:00 ps -f
vagrant   3396  3372  0 14:25 pts/0    00:00:00 tee -a ps_file.txt

[vagrant@server ~]$ cat ps_file.txt
UID        PID  PPID  C STIME TTY          TIME CMD
vagrant   3372  3371  0 14:25 pts/0    00:00:00 -bash
vagrant   3395  3372  0 14:25 pts/0    00:00:00 ps -f
vagrant   3396  3372  0 14:25 pts/0    00:00:00 tee -a ps_file.txt
```

以下兩種寄 mail 的方式相同：

```bash
$ cat filename | mail -s subject username

$ mail -s subject username < filename
```


-----------------------------------------------------------


Chapter 5. Managing Local Linux Users and Groups
================================================

## 5.1 Users and Groups

### What is a user?

`id` 用來顯示目前登入的使用者資訊：

```bash
[vagrant@server ~]$ id
uid=1000(vagrant) gid=1000(vagrant) groups=1000(vagrant),10(wheel) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```
> context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 是 SELinux 安全脈絡標籤

**<font color='red'>/etc/passwd</font>** 的格式可以參考 `man 5 passwd`

基本上 password 的部分已經都用 x 取代，並移到 **<font color='red'>/etc/shadow</font>** 存放

### What is a group?

1. 每個使用者只會有一個 primary group，記錄在 `/etc/passwd` 內

2. 每個使用者可以有 0 到多個 supplementary group，資訊會記錄在 `/etc/group` 內


## 5.2 Gaining Superuser Access

### The root user

一般使用者可透過 `su`, `sudo`, `PolicyKit`(GUI 內用，類似 Windows UAC) 來取得 root 權限

### Switching users with su

`su [-] username`：等同於新的 user 重新登入的效果

`su username`：產生新的 shell 並使用目前的環境變數

```bash
# 使用 -c 可達到類似 windows runas 的效果
[vagrant@server ~]$ su -c "ls /root" root
Password:
anaconda-ks.cfg
```

### Running commands as root with sudo

su 的缺點是，一次就拿到完整的 root 權限，且切換成 root 還必須知道 root 的密碼。

```bash
[vagrant@server ~]$ sudo cat /etc/sudoers | grep '^[^#]'
Defaults   !visiblepw
Defaults    always_set_home
Defaults    env_reset
Defaults    env_keep =  "COLORS DISPLAY HOSTNAME HISTSIZE INPUTRC KDEDIR LS_COLORS"
Defaults    env_keep += "MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE"
Defaults    env_keep += "LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES"
Defaults    env_keep += "LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE"
Defaults    env_keep += "LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY"
Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin
root    ALL=(ALL)       ALL
%wheel  ALL=(ALL)       ALL   #表示 wheel 群組內的擁有所有權限(% 開頭表示指定群組)

[vagrant@server ~]$ cat /etc/group | grep wheel
wheel:x:10:vagrant
```

使用 `sudo` 的特點：

1. 執行 root 的系統命令不需要記住 root 密碼

2. 所有 sudo 所執行的紀錄都會留在 `/var/log/secure` 中

```bash
[vagrant@server ~]$ sudo tail /var/log/secure
....
Feb 22 13:45:26 localhost sudo: vagrant : TTY=pts/0 ; PWD=/home/vagrant ; USER=root ; COMMAND=/bin/ls /root/
Feb 22 13:45:45 localhost sudo: vagrant : TTY=pts/0 ; PWD=/home/vagrant ; USER=root ; COMMAND=/bin/cat /var/log/secure
Feb 22 13:46:10 localhost sudo: vagrant : TTY=pts/0 ; PWD=/home/vagrant ; USER=root ; COMMAND=/bin/tail /var/log/secure
```

### 補充

`NIS`：集中驗證

`IPA`：集中授權管理

`sudo su -` or `sudo -s`：用一般的 user 切換成 root，輸入一般 user 的密碼


## 5.3 Managing Local User Accounts

### Managing local users

`/etc/login.defs` 檔案中紀錄了新增使用者時的相關預設設定，例如 UID range, 密碼有效期限 ... 等

```bash
[vagrant@server ~]$ cat /etc/login.defs | grep '^[^#]'
MAIL_DIR        /var/spool/mail
PASS_MAX_DAYS   99999
PASS_MIN_DAYS   0
PASS_MIN_LEN    5
PASS_WARN_AGE   7
UID_MIN                  1000
UID_MAX                 60000
SYS_UID_MIN               201
SYS_UID_MAX               999
GID_MIN                  1000
GID_MAX                 60000
SYS_GID_MIN               201
SYS_GID_MAX               999
CREATE_HOME     yes
UMASK           077
USERGROUPS_ENAB yes
ENCRYPT_METHOD SHA512
```

#### usermode modifies existing users

`usermod` 用來修改使用者資訊，相關參數如下：

| 參數 | 說明 |
|------|------|
| `-g/--gid` GROUP  | 設定 user 的 primary group |
| `-G/--groups` GROUPS | 設定 user 的 supplementary group |
| `-a/--append` | 與 `-G` 搭配，用來增加 supplementary group 設定 |
| `-d/--home` HOME_DIR | 設定 user 家目錄 |
| `-m/--move-home` | 移動 user 家目錄到新的地方，必須與 `-d` 參數同時使用 |
| `-s/--shell` SHELL | 指定登入 shell |
| `-L/--lock` | 鎖定 user |
| `-U/--unlock` | 解鎖 user |

#### userdel deletes users

`userdel -r username`：會將 user 刪除，連同家目錄 & mail 都一併移除

> userdel 命令若沒有加上 -r 參數，可能會有安全疑慮，主要是沒有 owner & owner group 的檔案可能會被新增的 user 取得存取權限

> 此問題可透過 `sudo find / -nouser -o --nogroup 2>/dev/null` 指令來找到if沒有 owner & owner group 的檔案

#### id displays user information

`id` 可用來顯示使用者資訊

```bash
[vagrant@server ~]$ id
uid=1000(vagrant) gid=1000(vagrant) groups=1000(vagrant),10(wheel) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

[vagrant@server ~]$ id root
uid=0(root) gid=0(root) groups=0(root)
```

#### UID ranges

UID 在 RHEL 7 中的定義：(預設值可參考 `/etc/login.defs`)

- `UID 0`：root

- `UID 1-200`：system users，通常用來啟動系統服務之用

- `UID 201-999`：保留的 system users，有其他額外未預先定義的系統服務需要時可使用

- `UID 1000`：一般使用者用的 UID

> 在 RHEL 7 之前，system user 使用 UID 1-499，一般使用者使用  UID 500+


### Practice

使用 variable 新增 user：

```bash
[vagrant@server ~]$ USER=juliet

[vagrant@server ~]$ echo $USER
juliet

[vagrant@server ~]$ sudo useradd ${USER}; echo ${USER} | sudo passwd --stdin ${USER}
Changing password for user juliet.
passwd: all authentication tokens updated successfully.
```


## 5.4 Managing Local Group Accounts

### 5.4.1 Managing supplementary groups

#### groupadd

- `-r`：用來增加 system group (GID > 1000)

```bash
# 使用 -g 選項指定 GID
$ sudo groupadd -g 5000 ateam

# 使用 -r 選項姜群組新增成 system group
$ sudo groupadd -r appusers

# 查詢系統中預設 GID range
[vagrant@server ~]$ cat /etc/login.defs | grep GID
GID_MIN                  1000
GID_MAX                 60000
SYS_GID_MIN               201
SYS_GID_MAX               999
```

> `/etc/login.defs` 檔案中紀錄了許多與使用者 & 群組管理上相關的預設參數

#### groupmod

- `-n`：修改群組名稱

- `-g`：修改 GID (這會衍生非預期的問題)
  - 家目錄不會改
  - 相對應的檔案 & 目錄皆不會改

#### usermod alters group membership

- `-g`：指定 primary group

- `sudo usermod -g NEW_PRIMARY_GROUPNAME USERNAME`：變更指定使用者的 primary group

- `-G`：指定 supplementary group

- `sudo uermod -aG SUPPLYMENTARY_GROUP USERNAME`：新增使用者的 supplementary group

## 5.5 Managing User Passwords

### 5.5.1 Shadow passwords and password policy

現在為了安全性，密碼都已經改存到只有 root 能讀取的 `/etc/shadow` 中，分為幾個欄位：
1. `name`：使用者名稱
2. `password`：密碼
3. `lastchange`：上次修改時間 (為單一數值，從 1970/01/01 作為第1天起算) (改為 0，強制使用者必須在下次登入時改密碼)
4. `minage`：密碼存活的最小生命周期(0 表示馬上可以改回來)
5. `maxage`：密碼存活的最大生命周期(最大為 99999，從 `lastchange` 開始算)
6. `warning`：提醒使用者的時間 (default: 7，七天前提醒)
7. `inactive`：寬限期，最多密碼可以存活 (maxage + inactive) 天；在寬限期登入會被強制要求更改密碼
8. `expire`：密碼失效日期，不受前面欄位影響，過期就表示密碼完全失效
9. `blank`：保留作為未來使用

說明第 2 個欄位 `password`

```bash
[vagrant@desktop ~]$ sudo tail -3 /etc/shadow
juliet:$6$TFn6B61u$sA/maknNkUeghjzgrGkoAiLYzNa/KqDQDR5A0m0PxTZeac4gdXQAleR.sxWcWK5VZnnSIhSAsD/WnIZr51MZA/:16864:0:99999:7:::
romeo:$6$vRE6wD1I$4eh6QNLPzyMw9pVjr.YwOia6s8y1zixFq2LpHSi/n.Q/A45J/jqiBCNXRJan2rl0p6MwuL.91mf3IkRNZQUVv.:16864:0:99999:7:::
hamlet:$6$Nu8r/BWW$PqoSIv9iRx9oQEH2ziw4L2WH0HMXs2YXFvEH8SFHmmKd/WxeAV0qOweqAXUN6TE375W7ShHHLxP6i7jBcHsD/1:16864:0:99999:7:::
```

以 user juliet `$6$TFn6B61u$sA/maknNkUeghjz` 為例：(以 **<font color='red'>$</font>** 作為分隔)

1. `6`：第一個部分，表示加密用的 hash 演算法，1 = MD5，6 = SHA-512 (RHEL 7 預設為 6)

2. `TFn6B61u`：用來在加密密碼時用的 random 字串(salt)，加密時會一起用到，為了避免同樣密碼的使用者會有相同的 hash 結果

3. `sA/maknNkUeghjz....`：hash(password + salt) 的結果

> `/etc/shadow` 的詳細說明，可以參考 [鳥哥的 Linux 私房菜 -- 第十三章、Linux 帳號管理與 ACL 權限設定](http://linux.vbird.org/linux_basic/0410accountmanager.php#shadow_file)

1. `name`：使用者帳號

2. `password`：上述的密碼資訊

3. `lastchange`：從 1970/01/01 起算的日期(0 表示強制使用者下次登入時變更密碼)

4. `min days`：

5. `max days`

6. `warning`

7. `inactive`

8. `expires`

9. `blank`

### 5.5.2 Password aging

![Linux Password Aging](https://lh3.googleusercontent.com/2bDqeWVGlV0vGtV0lm6TE-1KwHmNixd6pUAaDGh30pcFJGF3tI3WUsV6dxUzdRbSPh7dR3mzznf_EdinGaKeWQA1xrA9cdOJFajn-jQFCcgEfni4Sdpn4zd-6R4T-Rj0Cezry17a0m_hI61F1nD8TjQFomv6NltRNseZgPgHVM1JU0or0xr8Y322N7MfhzoQIXeLQgQMLTsKtLAr19avk7A3yUQiZn5m6RI6T0rVqmW2FsXYwGEiQiDQlmg9POKDIndNH9LknDoIGHK5PAL_03cmbuY5EdouDrBMQu1PCESLO5GvZgzG-UIWNXMYk692Rai3Cpw6zNbyMUdwqpzvUaCgsPFHYcRfA9C3U1VJ4p-O2Q9iIKq5-8DWzEzGb7HELfYNwAHny8QGnXBGnHa7qtzVUJseFngyW3I2T-rCD7rriDbNYep4-V_fSqMSCNFSpVcs6Xnaj9TmnmXoegWDt3p43VTQxqPPb5C1LpyOK8mwwzFJdrYEjRJNmRzz6o4tUTqfbeay_UdKTzARA4MrsP6k2X5wHPWlDOao9eh7cMtRsyQpk9KE3Mp2l44mIHs_tuIE=w657-h235-no)

RHEL7 用來加密密碼的 hash algorithm 已經預設改為 `SHA-512`(<font color='red'>**6**</font>)

```bash
# 強制使用者下次登入修改密碼
$ chage -d 0 USER_NAME

# 以 human readable 的方式顯示 /etc/shadow 的內容
$ chage -l USER_NAME

# 指定密碼失效日期 (也可以改用數字)
$ chage -E YYYY-MM-DD USER_NAME
```

```bash
# 檢視 juliet 目前的密碼期限相關設定
[vagrant@desktop ~]$ sudo chage -l juliet
Last password change                                    : Mar 04, 2016
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7

# 強制 juliet 在下次登入時變更密碼
[vagrant@desktop ~]$ sudo chage -d 0 juliet

# 使用 -E YYYY-MM-dd 的設定方式將帳號過期時間設定為 45 天後
[vagrant@desktop ~]$ sudo chage -E $(date -d "+45 days" +%F) juliet
```

### 5.5.3 Restricting access

限制使用者存取的方式：

1. Lock USER_NAME
2. 給 Expiration Date
3. 把 shell 給成 /sbin/nologin

``` bash
# 以下兩個功能相同
$ sudo usermod -L USER_NAME
$ sudo passwd -l USER_NAME
```

```bash
# lock user
$ sudo usermod -L juliet

# unlock user
$ sudo usermod -U juliet

# 設定 nologin shell 給使用者，使用者就沒有 shell 可用(例如：mail service)
$ sudo usermod -s /sbin/nologin juliet
```

### 5.5.4 Lab: Managing Local Linux Users and Groups

修改 `/etc/login.defs`(<font color='red'>**PASS_MAX_DAYS**</font>) 將密碼預設過期日改為 30

``` bash
# 新增一個 GID 40000，名稱為 consultants 的群組
$ groupadd -g 40000 consultants

# 新增使用者，指定 GID 為 40000，並設定密碼為 default
$ USR=sspade; useradd -G 40000 ${USR}; echo "default" | passwd --stdin ${USR}
$ USR=bboop; useradd -G 40000 ${USR}; echo "default" | passwd --stdin ${USR}
$ USR=dtracy; useradd -G 40000 ${USR}; echo "default" | passwd --stdin ${USR}

# 設定 sspade, bboop, dreacy 的密碼過期日(expiration date)為 90 天後
$ chage -E $(date -d "+90 days" +%Y-%m-%d) sspade
$ chage -E $(date -d "+90 days" +%Y-%m-%d) bboop
$ chage -E $(date -d "+90 days" +%Y-%m-%d) dtracy

# 限制 bboop 每 15 天要改一次密碼
$ chage -M 15 bboop

# 強制 sspade, bboop, dreacy 下次登入時要改密碼
$ chage -d 0 sspade
$ chage -d 0 bboop
$ chage -d 0 dtracy
```

## 5.6 Practice: Managing User Password Aging

Lock User：

```bash
# lock 使用者 romeo
[vagrant@desktop ~]$ sudo usermod -L romeo
# 無法切換使用者，因為已經 lock
[vagrant@desktop ~]$ su - romeo
Password:
su: Authentication failure

# unlock 之後便可登入
[vagrant@desktop ~]$ sudo usermod -U romeo
[vagrant@desktop ~]$ su - romeo
Password:
Last login: Wed Mar  9 14:39:24 UTC 2016 on pts/0
Last failed login: Wed Mar  9 14:40:51 UTC 2016 on pts/0
There were 2 failed login attempts since the last successful login.
[romeo@desktop ~]$
```

設定密碼過期時間 90 天 & 下次登入時修改密碼：

```bash
[vagrant@desktop ~]$ sudo chage -l romeo
Last password change                                    : Mar 09, 2016
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7

# 設定密碼過期時間 90 天
[vagrant@desktop ~]$ sudo chage -M 90 romeo
[vagrant@desktop ~]$ sudo chage -l romeo
Last password change                                    : Mar 09, 2016
Password expires                                        : Jun 07, 2016
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 90
Number of days of warning before password expires       : 7

# 強制使用者下次登入時變更密碼
[vagrant@desktop ~]$ sudo chage -l romeo
Last password change                                    : password must be changed
Password expires                                        : password must be changed
Password inactive                                       : password must be changed
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 90
Number of days of warning before password expires       : 7
```

設定使用者密碼過期時間為 180 天後：

```bash
[vagrant@desktop ~]$ sudo chage -E $(date -d "+180 days" +%F) romeo
[vagrant@desktop ~]$ sudo chage -l romeo
Last password change                                    : Mar 09, 2016
Password expires                                        : Jun 07, 2016
Password inactive                                       : never
Account expires                                         : Sep 05, 2016
Minimum number of days between password change          : 0
Maximum number of days between password change          : 90
Number of days of warning before password expires       : 7
```

-----------------------------------------------------------


Chapter 6. Controlling Access to Files with Linux File System Permissions
=========================================================================

## 6.1 Linux File System Permissions

### 6.1.1 Linux file system permissions

> 新增/刪除檔案不是看檔案本身權限，而是看上層目錄的權限

當使用者擁有目錄的 `w(write)` & `x(execute)` 權限時，可以刪除該目錄中自己也沒有權限的檔案，但這問題可以透過 `sticky bit` 來解決!

正常對目錄有存取權限的使用者，會同時有 `r(read)` & `w(write)` 兩個權限：

- 若沒有目錄的 `r(read)` 權限，使用者無法列出目錄中的檔案，但若知道明確檔名還是可以存取

- 若沒有目錄的 `w(write)` 權限，就只能列出目錄中的檔案內容，但都無法存取目錄中的任何檔案(連 timestamp 資訊都看不到)

- 但如果進不了目錄(**<font color='red'>沒有 execute 權限</font>**)，還是無法刪除檔案


### 6.1.2 Viewing file/directory permissions and ownership

```bash
# 列出目錄中所有子目錄 & 檔案的相關權限資訊
[vagrant@server ~]$ ls -l /home
total 4
drwx------. 6 vagrant vagrant 4096 Mar 30 20:23 vagrant

# 透過 '-d' 參數，僅列出指定目錄的權限 (權限中的最後一個 . 是作為 ACL 控制用)
[vagrant@server ~]$ ls -ld /home
drwxr-xr-x. 3 root root 20 Jan  3 04:22 /home
```

## 6.2 Managing File System Permissions from the Command Line

### 6.2.1 Changing file/directory permissions

透過 chmod -R 以遞迴的方式指定檔案 & 目錄的權限時，若包含了 x(execute) 權限，會讓檔案 & 目錄同時都有 x(execute) 的權限，但這通常不是我們需要的結果；一般我們只會希望只有目錄才需要 x(execute) 權限，此時只要加上大寫 `W` 參數即可：

例如：`chmod -R g+rwX somedir` 表示 somedir 目錄下所有的檔案都給 rw 權限，目錄則是給 rwx 權限。

### 6.2.2 Changing file/directory user or group ownership

```bash
# 以下兩個指令功能相同，都是修改檔案的 group ownership
[vagrant@server ch06]$ sudo chown :vboxsf 1st/aa
[vagrant@server ch06]$ sudo chgrp vboxsf 1st/aa
```
> 只有 root 可以修改檔案的 ownership，一般使用者儘可以針對自己所屬的群組設定 ownership

一般 web 網站的目錄，只會開啟目錄的 execute 權限，並且讓目錄內的檔案有 others read 的權限，讓使用者可以進入目錄，可以存取目錄中檔案的內容，但卻無法列出目錄中所有的檔案

``` bash
# 檔案 file1 拿掉 group & others 的 read & write 權限
$ chmod go-rw file1

# 針對多層目錄 & 檔案透過遞迴的方式設定權限(group 給予所有權限)
$ chmod -R g+rwx multi_layer_dir

# 針對多層目錄(只有目錄，沒有檔案)，使用遞迴的方式指定所有人有 execute 的權限
# 透過大寫 X，指定套用權限時僅會套用在目錄上，不會在檔案上
$ chmod -R a+X multi_layer_dir
```

只有 root 可以變更檔案的 ownership
> 例外：檔案擁有者可以改檔案所屬群組，但只可以改成屬於自己群組

## 6.3 Managing Default Permissions and File Access

### 6.3.1 Special permissions

若檔案的 execute 權限標示為 `setuid` or `setgid` 時(以 `s` 表示權限)，表示此檔案不論是哪個使用者執行，會以檔案 owner 的身分(`setgid` 會以群組的身分)執行，而不是執行檔案的使用者。 例如：**/etc/passwd**

```bash
[vagrant@server ~]$ ls -al /usr/bin/passwd
-rwsr-xr-x. 1 root root 27832 Jun 10  2014 /usr/bin/passwd
```

若 sticky bit(以 `t` 表示權限) 位於 directory 時，以 **<font color='red'>/tmp</font>** 為例，**只有檔案的擁有者才可以刪除檔案**：

```bash
[vagrant@server ~]$ ls -ld /tmp/
drwxrwxrwt. 7 root root 88 Apr  3 01:00 /tmp/
```

| 設定方式 | 在檔案上的效果 | 在目錄上的效果 |
|---------|---------------| ------------- |
| `u+s`(4) | 檔案執行時會以 owner user 的權限執行 | N/A |
| `g+s`(2) | 檔案執行時會以 owner group 的權限執行 | 在此目錄中新建立的檔案會擁有與目錄相同的 group 權限 |
| `o+t`(1) | N/A | 在此目錄中，使用者僅能移除他們自己所建立的檔案 |

```bash
# setuid
$ chmod u+s some_file

# setgid
$ chmod g+s,o-rx some_dir
$ chmod 2770 some_dir
```

#### SetUID
- 檔案
> -rwsr-xr-x 1 root root 47032 Jul 16  2015 /usr/bin/passwd -> /etc/shadow
> passwd 指令的 owner 為 root，因此執行此指令時是以 root 的權限執行

#### SetGID
- 檔案
> /usr/bin/locate -> /var/lib/mlocate/mlocate.db

- 目錄
> SetGID 設定在目錄上，則表示在該目錄中建立的檔案 or 目錄的擁有群組都會被強制設定為該目錄的擁有群組

#### Sticky Bit
- 目錄
> `o+t` 表示該目錄中的檔案只有擁有者可以移除

> **<font color='red'>T</font>**：表示 others 原本沒有 execute 權限

> **<font color='red'>t</font>**：表示 others 原本有 execute 權限

### 6.3.2 Default file permissions


Default permission (system)
- File: `666`
- Directory: `777`

Default Umask:
- root：`022`
- regular user：`002`

umask 使用 3 個數字進行修改，若少於 3 個數字，前面會被自動補 0；且更改的效果僅限於該 terminal session 中，重新登入後就會無效。

umask 的設定，若是要設定 global 的，可以到 **<font color='red'>/etc/profile</font>** & **<font color='red'>/etc/bashrc</font>** 中進行調整

如果要進行個人化設定，則可以到 **<font color='red'>~/.bash_profile</font>** & **<font color='red'>~/.bashrc</font>** 中進行調整
