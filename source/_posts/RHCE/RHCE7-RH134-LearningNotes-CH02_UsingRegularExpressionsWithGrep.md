---
layout: post
title:  "[RHCE7] RH134 Chapter 2 Using Regular Expressions with grep 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 2 Using Regular Expressions with grep 留下的內容"
date: 2016-04-29 11:50:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

<a name="ch2.2" />
2.2 Matching Text with grep
===========================

`.`(單一任何字元) & `\`(跳脫字元) 的用法：

```
# 尋找 rcx.d
[vagrant@server etc]$ ls | grep 'rc.\.d'
rc0.d
rc1.d
....
```

`[]`(中括號，符合其中一個字元) & `[^]` 的用法：

```bash
[vagrant@server etc]$ ls | grep 'rc[134]\.d'
rc1.d
rc3.d
rc4.d

[vagrant@server etc]$ ls | grep 'rc[1-3]\.d'
rc1.d
rc2.d
rc3.d

[vagrant@server etc]$ ls | grep 'rc[^1-5]\.d'
rc0.d
rc6.d
```

`*`(零個或多個前面的字元) & `\+`(一個或多個前面的字元) & `\?`(零個或一個前面的字元) 的用法：

```bash
[vagrant@server tmp]$ cat file.txt
ac
abc
abbc
abbbc
abbbbc
abbbbbc
abbbbbbc

# 找到全部
[vagrant@server tmp]$ cat file.txt | grep 'ab*c'
ac
abc
abbc
abbbc
abbbbc
abbbbbc
abbbbbbc

[vagrant@server tmp]$ cat file.txt | grep 'ab\+c'
abc
abbc
abbbc
abbbbc
abbbbbc
abbbbbbc

[vagrant@server tmp]$ cat file.txt | grep 'ab\?c'
ac
abc
```

`\{i\}`(i 個前面的字元) & `\{i,\}`(大於等於 i 個前面的字元) & `\{i,j\}`(i 到 j 的前面的字元) 的用法：

```bash
[vagrant@server tmp]$ cat file.txt | grep 'ab\{3\}c'
abbbc

[vagrant@server tmp]$ cat file.txt | grep 'ab\{3,\}c'
abbbc
abbbbc
abbbbbc
abbbbbbc

[vagrant@server tmp]$ cat file.txt | grep 'ab\{3,4\}c'
abbbc
abbbbc
```

### 練習：尋找包含 IP address 的行

```bash
# 尋找 IP address
[vagrant@server ~]$ ip addr | grep '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}'
    inet 127.0.0.1/8 scope host lo
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
    inet 172.25.25.11/24 brd 172.25.25.255 scope global enp0s8
```

`^`(一行的開頭) & `$`(一行的結尾)的用法：

```bash
[vagrant@server tmp]$ cat passwd | grep 'root'
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin

# 以 root 為開頭
[vagrant@server tmp]$ cat passwd | grep '^root'
root:x:0:0:root:/root:/bin/bash

# 以 bsah 為結尾
[vagrant@server tmp]$ cat passwd | grep 'bash$'
root:x:0:0:root:/root:/bin/bash
vagrant:x:1000:1000:vagrant:/home/vagrant:/bin/bash
```

```bash
[vagrant@server tmp]$ cat ff
aaaaa cataaa aaaa
aaaaa cat aaaaa
aaaaa aaaacat aaaa

[vagrant@server tmp]$ cat ff | grep '\<cat\>'
aaaaa cat aaaaa
```

`-v`：反向(**顯示沒符合的**)

```bash
# 搜尋沒有 root 的行
[vagrant@server tmp]$ grep -v 'root' passwd
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
```

`-n`：搜尋結果加上行號

```bash
[vagrant@server tmp]$ grep -n 'nologin' passwd
2:bin:x:1:1:bin:/bin:/sbin/nologin
3:daemon:x:2:2:daemon:/sbin:/sbin/nologin
4:adm:x:3:4:adm:/var/adm:/sbin/nologin
```

`-c`：列出符合條件的數量

```bash
# 搜尋 /etc 有多少個目錄
[vagrant@server tmp]$ sudo ls -lR /etc | grep -c '^d'
179
```

`-l`：只列出符合條件的檔名

```bash
[vagrant@server etc]$ grep 'password' /etc/* 2>/dev/null
/etc/dnsmasq.conf:#dhcp-option=encap:175, 191, pass     # iSCSI password
/etc/login.defs:#	PASS_MAX_DAYS	Maximum number of days a password may be used.
/etc/login.defs:#	PASS_MIN_DAYS	Minimum number of days allowed between password changes.
....
/etc/login.defs:# Use SHA512 to encrypt password.
/etc/services:shell           514/tcp         cmd             # no passwords used
...

[vagrant@server etc]$ grep -l 'password' /etc/* 2>/dev/null
/etc/dnsmasq.conf
/etc/login.defs
/etc/services
```

`-r`：搜尋整個路徑下的檔案

`-i`：不區分大小寫

`-e`：可同時給多個搜尋條件

### sed

1. `sed 's/cat/dog/' file.txt`：將檔案中每一行左邊的第一個 cat 換成 dog

2. `sed 's/cat/dog/gi' file.txt`：同上，但全部一起置換，且不區分大小寫

3. `sed 's/[Cc]at/dog/gi' file.txt`：Cat or cat 都會置換

4. `sed 's/\<[Cc]at\>/dog/gi' file.txt`：只有精準的 Cat or cat 會被置換

5. `sed '1,30s/cat/dog/gi' file.txt`：同 2，但僅處理 1~30 行

6. `sed '/begin/,/end/s/cat/dog/gi' file.txt`：同 2，但僅處理 begin 開頭的行到 end 開頭的行

7. `sed -e 's/cat/dog/g' -e 's/Cat/dog/g' file.txt`：同時給多個條件

8. `set '/^root/d' file.txt`：開頭為 root 的行刪除

8. `set '/^root/!d' file.txt`：開頭為 root 的行不刪除

> 加上 **<font color='red'>-i</font>** 參數會將實際的改變反應到檔案中(原本預設是不會變更檔案內容)
