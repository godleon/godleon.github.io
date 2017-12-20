---
layout: post
title:  "[RHCE7] RH254 Chapter 4 Network Port Security Learning Notes"
description: "此文章記錄學習 RHCE7 RH254 Chapter 4 Network Port Security 留下的內容"
date: 2016-05-29 09:00:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH254]
---

4.1 Managing Firewalld
======================

## 4.1.1 Firewalld overview

`firewalld.service` 在 RHEL7 中目前是預設的防火牆管理用服務，因此也管理了 Linux kernel netfilter。

但因為 `firewalld.service` 與舊有的 `iptables.service`/`ip6tables.service`/`ebtables.service` 會衝突，因此兩個陣營建議選一個使用就好，假設若要使用 firewalld，那就使用 `systemctl mask` 指令把其他的 system unit 停止掉：

```bash
$ sudo systemctl mask iptables.service
ln -s '/dev/null' '/etc/systemd/system/iptables.service'

$ sudo systemctl mask ip6tables.service
ln -s '/dev/null' '/etc/systemd/system/ip6tables.service'

$ sudo systemctl mask ebtables.service
ln -s '/dev/null' '/etc/systemd/system/ebtables.service'
```

firewalld 將所有進入的流量分為不同的 `zone` 來看待，每個 zone 都會擁有相對應的一組規則，處理原則就是**先比對到的規則就先處理**

預設的 firewalld zone 如下：

| Zone name | Default configuration |
|-----------|-----------------------|
| `trusted` | 允許全部進入的流量 |
| `home` | 1. 允許 `ssh`, `mdns`, `ipp-client`, `samba-client`, `dhcpv6-client` 等服務的流量<br />2. 允許與連外相關的進入流量<br/>3. 其他所有進入的流量皆拒絕 |
| `internal` | 與 zone `home` 相同 |
| `work` | 1. 允許 `ssh`, `ipp-client`, `dhcpv6-client` 等服務的流量<br />2. 允許與連外相關的進入流量<br/>3. 其他所有進入的流量皆拒絕 |
| `public` | 1. 允許 `ssh`, `dhcpv6-client` 等服務的流量<br />2. 允許與連外相關的進入流量<br/>3. 其他所有進入的流量皆拒絕<br />**4. 是新增 network interface 的 default zone** |
| `external` | 1. 僅允許 `ssh` 服務的流量<br />2. 允許與連外相關的進入流量<br/>3. 其他所有進入的流量皆拒絕<br />4. 所有連外的流量都會偽裝來源(**NAT**) |
| `dmz` |  1. 僅允許 `ssh` 服務的流量<br />2. 允許與連外相關的進入流量<br/>3. 其他所有進入的流量皆拒絕 |
| `block` | 僅允許與連外相關的進入流量 |
| `drop` | 允許與連外相關的進入流量(**連 ICMP error 都不會回應**) |

## 4.1.2 Managing firewalld

### Configure firewall settings with firewall-cmd

RHEL7 提供 `firewall-cmd`(console) & `firewall-config`(GUI) 作為管理防火牆之用。

透過 firewall-cmd 設定規則，有幾個重點需要注意：

- 若沒特別指定都只是會只有在 runtime 才有效，重開機就無效了，除非加上 `--permanent` 選項。

- 指令中都會加上 `--zone=<ZONE>`，若沒加就會預設為 default zone

- 設定規則一般都會加上 `--permanent` 選項，並使用 `firewall-cmd --reload` 來進行永久性的變更套用

- 若只是暫時的測試，可以使用 `timeout=<TIMEINSECONDS>` 來讓規則短暫在 runtime 時有效

| firewall-cmd commands | Explanation |
|-----------------------|-------------|
| `--get-default-zone` | 查詢 default zone |
| `--set-default-zone=<ZONE>` | 設定 dfault zone |
| `--get-zones` | 列出所有支援的 zone |
| `--get-services` | 列出所有支援的 service |
| `--add-source=<CIDR>` [--zone=<ZONE>] | 允許來自指定 source 的進入流量 |
| `--remove-source=<CIDR>` [--zone=<ZONE>] | 禁止來自特定 source 的進入流量 |
| `--add-interface=<INTERFACE> [--zone=<ZONE>]` | 允許從特定裝置進入的流量 |
| `--change-interface=<INTERFACE> [--zone=<ZONE>]` | 設定指定的 interface 與 zone 關聯 |
| `--list-all [--zone=<ZONE>]` | 列出指定 zone 的所有 interface, source, service, port 等規則 |
| `--list-all-zones` | 取得所有 zone 的設定規則資訊 |
| `--add-service=<SERVICE> [--zone=<ZONE>]` | 允許進入流量到指定的 service |
| `--add-port=<PORT/PROTOCOL> [--zone=<ZONE>]` | 允許進入流量到指定的 port or protocol |
| `--remove-service=<SERVICE> [--zone=<ZONE>]` | 禁止進入流量到指定的 service |
| `--remove-port=<PORT/PROTOCOL>` [--zone=<ZONE>] | 禁止進入流量到指定的 port or protocol |
| `--reload` | 將 runtime 設定變成永久設定<br />(**沒有加上 --permanent 選項的規則會失效**) |

### firewall-cmd example

以下用幾個簡單範例，示範使用 firewall-cmd 達成以下設定：(使用 `dmz` zone)

1. 將 default zone 設定為 `dmz`

2. zone `internal` 允許來自 192.168.0.0/24 的流量

3. zone `internal` 允許存取 mysql 服務

```bash
$ sudo firewall-cmd --set-default-zone=dmz

$ sudo firewall-cmd --permanent --zone=internal --add-source=192.168.0.0/24

$ sudo firewall-cmd --perminant --zone=internal --add-service=mysql

$ sudo firewall-cmd --reload
```

### Firewall configuration files

firewalld 的設定檔放在 `/etc/firewalld` & `/usr/lib/firewalld` 目錄下，同樣的設定，放在 **/etc/firewalld** 目錄下的會優先被採用，這是讓管理者可以用來修改原有的預設值之用

-------------------------------------------------------

Practice: Configuring a Firewall
================================

## 目標

1. 在 server 上安裝 `httpd` & `mod_ssl` 套件，並確認 `httpd.service` 有啟用且運行中

2. web 首頁顯示 `COFFEE` 字眼

3. server 上必須啟動 `firewalld.service`

4. 在 server 上進行 firewalld 設定，使用 `dmz` zone 的設定處理未指定的連線

5. 來自 `172.25.X.0/24` 的流量必須導引到 `work` zone 來處理

6. `work` zone 必須開啟 `https` 服務所需要的所有 port，而 `http` 的流量則必須被過濾

## 實作過程

安裝 & 設定 httpd.service，並修改首頁：

```bash
$ sudo yum -y install httpd mod_ssl

# 啟動 httpd.service
$ sudo systemctl enable httpd.service
ln -s '/usr/lib/systemd/system/httpd.service' '/etc/systemd/system/multi-user.target.wants/httpd.service'
$ sudo systemctl start httpd.service

$ sudo systemctl status httpd.service
httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
   Active: active (running) since Thu 2016-05-26 10:25:01 JST; 5s ago
 Main PID: 29559 (httpd)
   Status: "Processing requests..."
   CGroup: /system.slice/httpd.service
           ├─29559 /usr/sbin/httpd -DFOREGROUND
           ├─29560 /usr/sbin/httpd -DFOREGROUND
           ├─29561 /usr/sbin/httpd -DFOREGROUND
           ├─29562 /usr/sbin/httpd -DFOREGROUND
           ├─29563 /usr/sbin/httpd -DFOREGROUND
           └─29564 /usr/sbin/httpd -DFOREGROUND

May 26 10:25:01 server0.example.com systemd[1]: Started The Apache HTTP Server.

# 修改首頁
$ echo "COFFEE" | sudo tee --append /var/www/html/index.html

# 確認網頁回傳的內容為 COFFEE
$ curl http://localhost
COFFEE
```

確認 firewalld.service 狀態為啟動中：

```bash
# firewalld.service 狀態
$ sudo systemctl status firewalld.service
firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled)
   Active: active (running) since Thu 2016-05-26 09:59:39 JST; 28min ago
 Main PID: 475 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─475 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

May 26 09:59:39 localhost systemd[1]: Started firewalld - dynamic firewall daemon.

# iptables.service 狀態
$ sudo systemctl status iptables.service
iptables.service - IPv4 firewall with iptables
   Loaded: loaded (/usr/lib/systemd/system/iptables.service; disabled)
   Active: inactive (dead)

# ip6tables.service 狀態
$ sudo systemctl status ip6tables.service
ip6tables.service - IPv6 firewall with ip6tables
   Loaded: loaded (/usr/lib/systemd/system/ip6tables.service; disabled)
   Active: inactive (dead)

# ebtables.service 狀態
$ sudo systemctl status ebtables.service
ebtables.service - Ethernet Bridge Filtering tables
   Loaded: loaded (/usr/lib/systemd/system/ebtables.service; disabled)
   Active: inactive (dead)
```

設定 `dmz` zone 處理為指定連線：(即是設定為預設的 zone)

```bash
$ sudo firewall-cmd --set-default-zone=dmz
success
```

來自 `172.25.X.0/24` 的流量必須導引到 `work` zone 來處理

```bash
$ sudo firewall-cmd --permanent --zone=work --add-source=172.25.X.0/24
```

`work` zone 必須開啟 `https` 服務所需要的所有 port，而 `http` 的流量則必須被過濾：

```bash
$ sudo firewall-cmd --permanent --zone=work --add-service=https
```

套用 & 檢查設定：

```bash
$ sudo firewall-cmd --reload
success

$ sudo firewall-cmd --get-default-zone
dmz

$ sudo firewall-cmd --get-active-zones
dmz
  interfaces: eth0
work
  sources: 172.25.X.0/24
ROL
  sources: 172.25.0.252/32

$ sudo firewall-cmd --zone=work --list-all
work
  interfaces:
  sources: 172.25.X.0/24
  services: dhcpv6-client https ipp-client ssh
  ports:
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
```

最後，從 desktop 查詢：`curl -k https://serverX`


-------------------------------------------------------

4.2 Managing Rich Rules
=======================

## 4.2.1 Rich rules concepts

除了 firewalld 原生支援的 zone & service 之外，使用者還可以透過 `direct rules` & `rich rules` 兩種機制來自訂規則

### Direct rules

顧名思義，這種機制就是為了相容 {ip,ip6,eb}tables rule 所產生出來的，若已經有既有的 rules，可透過此機制加入到 firewalld 的管理下，詳細的設定可以參考 `firewall-cmd(1)` & `firewalld.direct(5)`

### Rich rules

這就是全新給 firewalld 使用的客製化 rule 機制，除了規則之外，還可以設定 logging 的機制(使用 `syslog` & `auditd`)，甚至連 port forwards, masquerading, rate limiting 等功能都可以設定。

rich rule 的設定基本語法架構如下：

```bash
rule
    [source]
    [destination]
    service|port|protocol|icmp-block|masquerade|forward-port
    [log]
    [audit]
    [accept|reject|drop]
```

> 完整的 rich rule 設定語法可以參考 `firewalld.richlanguage(5)`

### Rule ordering

當規則多起來時，順序的問題就很重要了，順序如下：

1. 任何與 port forwarding & masquerading 相關的規則

2. 任何 logging 相關的規則

3. 任何放行的規則

4. 任何禁止的規則

基本上，先比對到的規則就會先使用，都沒相符的規則就使用預設的規則，而每個 zone 都會有不同的預設規則。

不過 direct rule 會是例外，direct rule 會在 firewalld 開始過濾前就會先執行。

### Test and debugging

為了測試與 debug 的方便，幾乎所有加入到 runtime 設定的規則都可以額外加上 `timeout` 的設定，時間到就會自動移除，之後確認沒問題就可以改用 `--permanent` 讓設定永久生效

## 4.2.2 Working with rich rules

firewall-cmd 有 4 個選項用來處理 rich rule 之用，這些選項都可以搭配 `--permanent` & `--zone` 使用：

- `--add-rich-rule='<RULE>'`新增規則

- `--remove-rich-rule='<RULE>'`：移除規則

- `--query-rich-rule-'<RULE>'`：查詢規則是否有被加入到指定的 zone(or default zone)，回傳 0 表示有，1 則是沒有

- `--list-rich-rules`：顯示所有規則 (也可以用 `--list-all` or `--list-all-zones` 檢視所有規則)

以下為一些簡單範例：(以下權限以 root 身份執行)

```bash
# default zone = public
[root@server ~]# firewall-cmd --get-default-zone
public

# 在 zone "work" 中設定拒絕來自 192.168.0.11 的流量，永久生效
# family 可以省略
[root@server ~]# firewall-cmd --permanent --zone=work --add-rich-rule='rule family=ipv4 source address=192.168.0.11/32 reject'

# 在 default zone 中，允許每分鐘能有兩個 ftp 連線產生
[root@server ~]# firewall-cmd --add-rich-rule='rule service name=ftp limit value=2/m accept'

# 在 default zone 中，丟棄 IPSec esp 協定的所有封包
# reject 會送 ICMP 回 client，但 drop 則丟棄後完全不回應(因此 drop 會常用在已知的惡意網路段上)
[root@server ~]# firewall-cmd --permanent --add-rich-rule='rule protocol value=esp drop'

# 在 zone "work" 中，設定允許來自 192.168.1.0/24 網段的流量，進入 tcp port 7900-7905
[root@server ~]# firewall-cmd --permanent --zone=work --add-rich-rule='rule family=ipv4 source address=192.168.1.0/24 port port=7900-7905 protocol=tcp accept'
```

## 4.2.3 Loggind with rich rules

logging 的功能很適合用在 debug or monitor 的需求上，firewalld 可以使用 `syslog`，也可以將訊息送給 `auditd`；此外，還可以設定 rate limit，避免 log 爆量產生塞滿硬碟

若要搭配 `syslog`，使用下面語法：(其中 Log Level 有 `emerg`, `alert`, `crit`, `error`, `warning`, `notice`, `info`, `debug` 幾種可用)

> log [prefix="<PREFIX TEXT>"] [level=<LOG LEVEL>] [limit value="<RATE/DURATION>"]

若搭配 `auditd`，則使用下列語法：

> audit [limit value="<RATE/DURATION>"]

以下是則是設定 logging 功能相關的範例：

```bash
# 在 zone "work" 中，設定允許 ssh 的流量
# 搭配 syslog 記錄(prefix="ssh ", level="notice", 一分鐘最多三筆資料)
[root@server ~]# firewall-cmd --permanent --zone=work --add-rich-rule 'rule service name="ssh" log prefix="ssh " level="notice" limit value="3/m" accept'

# 在 default zone 中，拒絕來自 ipv6 "2001:db8::/64" 網段到達 dns 服務的流量
# 搭配 auditd，每個小時最多一筆紀錄
# 此規則在 300 秒後會自動失效
[root@server ~]# firewall-cmd --add-rich-rule='rule family=ipv6 source address="2001:db8::/64" service name="dns" audit limit value="1/h" reject' --timeout=300
```

-------------------------------------------------------

Practice: Writing Custom Rules
==============================

## 目標

1. server 安裝 `web server` 要讓 desktop 可以連

2. 限定`最多每秒三筆 log 記錄`，且 log 記錄開頭必須為 `NEW HTTP `

## 實作過程

```bash
$ sudo yum -y install httpd

# 設定首頁
$ echo "NEW HTTP" | sudo tee /var/www/html/index.html
NEW HTTP

# 啟動 web server
$ sudo systemctl enable httpd.service
ln -s '/usr/lib/systemd/system/httpd.service' '/etc/systemd/system/multi-user.target.wants/httpd.service'
$ sudo systemctl start httpd.service

# 確認本機服務正常 (此時 desktop 無法連入)
$ curl http://localhost
NEW HTTP

[root@server ~]# firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=172.25.0.10/32 service name="http" log level=notice prefix="NEW HTTP " limit value="3/s" accept'
[root@server ~]# firewall-cmd --reload

###### dekstop 連入一次 ######
[root@server ~]# tail /var/log/messages
.......
May 27 16:41:23 localhost kernel: NEW HTTP IN=eth0 OUT= MAC=52:54:00:00:00:0b:52:54:00:00:00:0a:08:00 SRC=172.25.0.10 DST=172.25.0.11 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=55651 DF PROTO=TCP SPT=49712 DPT=80 WINDOW=14600 RES=0x00 SYN URGP=0
```

-------------------------------------------------------

4.3 Masquerading and Port Forwarding
====================================

firewalld 也支援了 `masquerading`(**SNAT**, 稍有差異) & `port forwarding`(**DNAT**) 兩種 NAT 模式

## 4.3.1 Masquerading

設定 masquerading 的方式如下：

```bash
# 將此機器設定為 zone "work" 的 NAT
[root@server ~]# firewall-cmd --permanent --zone=work --add-masquerade
```

也可以使用 rich rule 設定較為複雜的規則：

```bash
# 在 zone "work" 中，將此機器設定為來源為 192.168.0.0/24 的 NAT
[root@server ~]# firewall-cmd --permanent --zone=work --add-rich-rule 'rule family=ipv4 source address=192.168.0.0/24 masquerade'
```

## 4.3.2 Port forwarding

設定 port forwarding 的語法如下：

> firewall-cmd --permanent --zone=\<ZONE\> --add-forward-port=port=\<PORT NUMBER\>:proto=\<PROTOCOL\>[:toport=\<PORTNUMBER\>] [:toaddr=\<IPADDR\>]

```bash
# 在 zone "public" 中，將到達 tcp:513 的流量導向 192.168.0.254:132
[root@server ~]# firewall-cmd --permanent --zone=public --add-forward-port=port=513:proto=tcp:toport=132:toaddr=192.168.0.254
```

也可以使用 rich rule 設定較為複雜的規則：

```bash
# 在 zone "work" 中，將來自 192.168.0.0/26，且到 tcp:80 的流量，導向 tcp:8080
[root@server ~]#  firewall-cmd --permanent --zone=work --add-rich-rule='rule family=ipv4 source address=192.168.0.0/26 forward-port port=80 protocol=tcp to-port=8080'
```

-------------------------------------------------------

Practice: Forwarding a port
===========================

## 目標

將來自 172.25.0.10/32　的 `tcp port 443` `ssh` 連線流量轉到 `tcp port 22`

## 實作過程

```bash
[root@server ~]# firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=172.25.0.10/32 forward-port port=443 protocol=tcp to-port=22'
[root@server ~]# firewall-cmd --reload
```

-------------------------------------------------------

4.4 Managing SELinux Port Labeling
==================================

要確保服務可以正常運行，就要確定 network port 擁有正確的 SELinux type　來與服務搭配才可以

## 4.4.1 SELinux port labeling

不僅是 file & process，連 network traffic 都屬於 SELinux 的管理範圍內，舉例來說：`22/tcp` 就有一個 `ssh_port_t` 的 label 與其相關

因此當一個 process 要監聽某一個 port 時，SELinux 會去檢查 port 有沒有相對應的 label 可供 process 進行 binding，若沒有則 process 會無法正常監聽指定的 port；這功能可以防範不正常的 process 使用到一般服務的 port

## 4.4.2 Managing SELinux port labeling

### Listing port labels

需要調整 label 通常是因為管理者要讓服務監聽一個非標準的 port 上，以下指令可以列出目前系統中已經存在的 SELinux port type：

```bash
$ sudo semanage port -l

SELinux Port Type              Proto    Port Number
.........
amavisd_send_port_t            tcp      10025
amqp_port_t                    tcp      5671-5672
amqp_port_t                    udp      5671-5672
aol_port_t                     tcp      5190-5193
.........
```

### Managing port labels

RHEL7 中提供 `semanage` 指令可用來修改 SELinux port type，指令類似如下：

> sudo semanage port -a -t PORT_LABEL -p tcp|udp PORT_NUMBER

```bash
# 允許 gopher service 監聽 tcp port 71
$ sudo semanage port -a -t gopher_port_t -p tcp 71

# 修改 port lebel 為 http_port_t
$ sudo semanage port -m -t http_port_t -p tcp 71

# 移除 port label
$ sudo semanage port -d -t http_port_t -p tcp 71
```



### 其他

詳細的 SELinux 參考文件可透過安裝 `selinux-policy-devel` 套件取得：

```bash
$ sudo yum -y install selinux-policy-devel
$ sudo mandb
$ man -k _selinux
```

-------------------------------------------------------

Practice: Managing SELinux Port Labeling
========================================

## 目標

1. 確保 `httpd.service` 啟用且執行

2. web server 監聽 `82/tcp`

## 實作過程

```bash
# 新增 port type label
$ sudo semanage port -a -t http_port_t -p tcp 82

# 重新啟動 httpd.service
[student@server0 ~]$ sudo systemctl enable httpd.service
[student@server0 ~]$ sudo systemctl restart httpd.service
[student@server0 ~]$ sudo systemctl status httpd.service
httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
   Active: active (running) since Sat 2016-05-28 10:43:14 JST; 2s ago
 Main PID: 2080 (httpd)
   Status: "Processing requests..."
   CGroup: /system.slice/httpd.service
           ├─2080 /usr/sbin/httpd -DFOREGROUND
           ├─2081 /usr/sbin/httpd -DFOREGROUND
           ├─2082 /usr/sbin/httpd -DFOREGROUND
           ├─2083 /usr/sbin/httpd -DFOREGROUND
           ├─2084 /usr/sbin/httpd -DFOREGROUND
           └─2085 /usr/sbin/httpd -DFOREGROUND

May 28 10:43:14 server0.example.com systemd[1]: Started The Apache HTTP Server.

# 以 root 身分新增 firewall 設定
[root@server ~]# firewall-cmd --permanent --add-port=82/tcp
[root@server ~]# firewall-cmd --reload
```

-------------------------------------------------------

Lab: Network Port Security
==========================

## 目標

1. 讓 `sshd.service` 同時監聽兩個 port(`22/tcp` & `999/tcp`)

2. 來自 `172.25.0.0/24` 的流量可以允許進入 zone `work`

3. `22/tcp` & `999/tcp` 在 zone `work` 中必須都要能夠使用

## 實作過程

檢查 sshd.service 狀態：

```bash
$ sudo systemctl status sshd.service
sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled)
   Active: active (running) since Sat 2016-05-28 10:48:41 JST; 2min 6s ago
  Process: 1742 ExecStartPre=/usr/sbin/sshd-keygen (code=exited, status=0/SUCCESS)
 Main PID: 1743 (sshd)
   CGroup: /system.slice/sshd.service
           └─1743 /usr/sbin/sshd -D

May 28 10:48:41 server0.example.com systemd[1]: Starting OpenSSH server daemon...
May 28 10:48:41 server0.example.com systemd[1]: Started OpenSSH server daemon.
May 28 10:48:41 server0.example.com sshd[1743]: error: Bind to port 999 on 0.0.0.0 failed: Permission denied.
May 28 10:48:41 server0.example.com sshd[1743]: error: Bind to port 999 on :: failed: Permission denied.
May 28 10:48:41 server0.example.com sshd[1743]: Server listening on 0.0.0.0 port 22.
May 28 10:48:41 server0.example.com sshd[1743]: Server listening on :: port 22.
May 28 10:48:42 server0.example.com python[1747]: SELinux is preventing /usr/sbin/sshd from name_bind access on the tcp_socket .

                                                  *****  Plugin bind_ports (92.2 confidence) suggests   ************************...
May 28 10:48:42 server0.example.com python[1747]: SELinux is preventing /usr/sbin/sshd from name_bind access on the tcp_socket .

                                                  *****  Plugin bind_ports (92.2 confidence) suggests   ************************...
Hint: Some lines were ellipsized, use -l to show in full.
```

加入 port label for sshd.service：

```bash
[root@server ~]# semanage port -a -t ssh_port_t -p tcp 999
[root@server ~]# systemctl reload sshd.service
[root@server ~]# systemctl status sshd.service
sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled)
   Active: active (running) since Sat 2016-05-28 11:10:17 JST; 2min 51s ago
  Process: 2225 ExecReload=/bin/kill -HUP $MAINPID (code=exited, status=0/SUCCESS)
  Process: 1708 ExecStartPre=/usr/sbin/sshd-keygen (code=exited, status=0/SUCCESS)
 Main PID: 1710 (sshd)
   CGroup: /system.slice/sshd.service
           └─1710 /usr/sbin/sshd -D

May 28 11:10:17 server0.example.com sshd[1710]: Server listening on :: port 22.
May 28 11:10:18 server0.example.com python[1713]: SELinux is preventing /usr/sbin/sshd from name_bind access on the tcp_socket .

                                                  *****  Plugin bind_ports (92.2 confidence) suggests   ************************...
May 28 11:10:18 server0.example.com python[1713]: SELinux is preventing /usr/sbin/sshd from name_bind access on the tcp_socket .

                                                  *****  Plugin bind_ports (92.2 confidence) suggests   ************************...
May 28 11:13:03 server0.example.com systemd[1]: Reloading OpenSSH server daemon.
May 28 11:13:04 server0.example.com sshd[1710]: Received SIGHUP; restarting.
May 28 11:13:04 server0.example.com systemd[1]: Reloaded OpenSSH server daemon.
May 28 11:13:04 server0.example.com sshd[1710]: Server listening on 0.0.0.0 port 999.
May 28 11:13:04 server0.example.com sshd[1710]: Server listening on :: port 999.
May 28 11:13:04 server0.example.com sshd[1710]: Server listening on 0.0.0.0 port 22.
May 28 11:13:04 server0.example.com sshd[1710]: Server listening on :: port 22.
Hint: Some lines were ellipsized, use -l to show in full.
```

放行相關的防火牆：

```bash
[root@server ~]# firewall-cmd --permanent --zone=work --add-source=172.25.0.0/24
[root@server ~]# firewall-cmd --permanent --zone=work --add-port=999/tcp
[root@server ~]# firewall-cmd --reload
```

-----------------------------------------------------------------------------

老師補充(iptables)
=================

- Linux 防火牆叫做 **netfilter**

- Linux 把很多功能獨立出來成一個一個小檔案，可以到 **/lib/modules/<kernel version>/kernel/** 目錄下找 `*.ko` 的檔案

> 在上面的目錄的 `netfilter` / `ipv4` / `ipv6` 等資料夾下，都是跟網路有關的模組，可參考 http://www.netfilter.org 尋找 `extention howto` 文件，查詢 ko 的使用方式

## 1、netfilter tables

主要有五個 table：

1. **filter**：主要防火牆的功能

2. **nat**：IP 分享器的功能進到主機之後，就直接轉出去的封包，歸類為 FORWARD (例如：本機為 router)

3. **mangle**：可對封包的內容小幅度的修改(例如：TTL，可偽裝成不同的作業系統)

4. **raw**：調整連線追蹤的功能，可以指定某些電腦不做連線追蹤

5. **security**

### 1.1 filter chains

- **INPUT**：從外面進來，目的地為本機應用程式的封包，歸類為 INPUT

- **FORWARD**：進到主機之後，就直接轉出去的封包，歸類為 FORWARD (例如：本機為 router)

- **OUTPUT**：本機的應用程式，目的地是外面主機的封包，歸類為 OUTPUT

> Linux netfilter 架構：`table` -> `chain` -> `rule`

> chain default policy：只有 `ACCEPT` & `DROP` 兩種，完全沒動過就會是 ACCEPT

### 1.2 連線追蹤(使用 state 模組)

狀態分為四種：

1. **NEW**：所有連線的第一個封包狀態，皆為 NEW

2. **ESTABLISHED**：當封包穿越了 FW，這條連線的後續封包狀態都會變成 ESTABLISHED

3. **RELATED**：因為主動產生的連線而發生的其他類型封包

4. **INVALID**：狀態不明的封包，建議一律丟棄

### 1.3 iptables 他使用上的注意事項

- 若要阻擋特定其他使用上的注意事項

- 若要阻擋特定網路流量，對內使用 **REJECT**、對外使用 **DROP**

- `--dport 80`：destination port = 80

- `-i eth0`：incoming interface = eth0

- `-o eth1`：outgoing interface = eth1

- 為了避免影響 loopback interface，不建議設定 chain default policy，而是加上 `iptables -t filter -A INPUT -i eth0 -j DROP` 到最後一條規則

- `iptables -L -v -n`：可看出那一條規則比較熱門

## 2、NAT 種類

- 把來源端 ip 換掉的 NAT，稱為 **SNAT** (POSTROUTING chain 的任務)

- 把目的端 ip 換掉的 NAT，稱為 **DNAT** (PREROUTING chain 的任務)

- 本機產生的封包，原本僅能有 SNAT，而 nat OUTPUT chain 是要補足 DNAT 的功能

- 要作 NAT 必須開啟 port forward 功能：`echo 1 > /proc/sys/net/ipv4/ip_forward`


## 3、iptables 使用方式

```bash
[root@server0 /]# systemctl disable firewalld
[root@server0 /]# systemctl stop firewalld

# List chains of filter table (可以看 chain default rule)
[root@server0 /]# iptables -t filter -L

# Append rule to chain FORWARD
# -p：指定 protocol，可參考 /etc/protocols
# -s：source
[root@server0 /]# iptables -t filter -A FORWARD -p tcp -s 172.25.254.250 -j ACCEPT
[root@server0 /]# iptables -t filter -A FORWARD -p udp -s 172.25.254.250 -j ACCEPT
[root@server0 /]# iptables -t filter -A FORWARD -p icmp -s 172.25.254.250 -j ACCEPT

# -n：以數字方式表示
# --line-number：顯示第幾條規則
[root@server0 /]# iptables -t filter -L -n --line-number

# 設定 chain default rule
[root@server0 /]# iptables -t filter -P INPUT DROP

# 清除 chain rules
[root@server0 /]# iptables -t filter -F
```

## 4、基本操作練習

```bash
[root@server0 ~]# iptables -t filter -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

[root@server0 ~]# iptables -t filter -A INPUT -p tcp -j ACCEPT
[root@server0 ~]# iptables -t filter -A INPUT -p udp -j ACCEPT
[root@server0 ~]# iptables -t filter -A INPUT -p icmp -j ACCEPT
[root@server0 ~]# iptables -t filter -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     tcp  --  anywhere             anywhere            
ACCEPT     udp  --  anywhere             anywhere            
ACCEPT     icmp --  anywhere             anywhere            

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

# 設定 chain default policy  
[root@server0 ~]# iptables -t filter -P INPUT DROP
[root@server0 ~]# iptables -t filter -L
Chain INPUT (policy DROP)
target     prot opt source               destination         
ACCEPT     tcp  --  anywhere             anywhere            
ACCEPT     udp  --  anywhere             anywhere            
ACCEPT     icmp --  anywhere             anywhere            

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
[root@server0 ~]# iptables -t filter -F
FORWARD  INPUT    OUTPUT   

# remove all chain rules
[root@server0 ~]# iptables -t filter -F INPUT
```

## 5、使用 shell script 建立 FW

```bash
#!/bin/bash

# 防止 sync flooding 攻擊(開啟 tcp sync cookie)
echo 1 > /proc/sys/net/ipv4/tcp_syncookies

SRV=172.25.0.11

iptables -t filter -F

# 設定 connection track
iptables -t filter -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# 避免 INVALID 封包被其他服務所接收
iptables -t filter -A INPUT -m state --state INVALID -j DROP

iptables -t filter -A INPUT -d $SRV -p tcp --dport 22 -j ACCEPT
iptables -t filter -A INPUT -d $SRV -p tcp --dport 25 -j ACCEPT
iptables -t filter -A INPUT -d $SRV -p tcp --dport 110 -j ACCEPT

iptables -t nat -A PREROUTING -i eth0 --dport 443 -j DNAT --to 127.0.0.1:22

# 用來取代 chain default policy
iptables -t filter -A INPUT -i eth0 -j DROP
```
