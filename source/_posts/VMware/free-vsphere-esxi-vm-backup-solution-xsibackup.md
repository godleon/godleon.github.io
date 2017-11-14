---
layout: post
title:  "[VMware] 免費的 vSphere ESXi VM 備份方案 - XSIBACKUP"
date:   2014-11-14 11:45:00
comments: true
categories:
  - VMware
tags:
  - VMware
  - Backup
---

最近公司在找給 VMware vSphere ESXi 用的 shared storage，想當然爾也會考慮到備份的問題

後來學長提供了 [xsibackup](http://sourceforge.net/projects/xsibackup/) 這個 opensource 的免費軟體，雖然是免費，可是備份功能也不差呢。

## 環境設定

- vSphere ESXi **5.5 Update 2**
- esxibackup **4.1.6**


## 使用方式

1、 首先必須先開啟 ESXi Host 的 SSH servive，並透過 ssh client 登入到 esxi 中

2、 下載 xsibackup 程式並解壓縮，將 xsibackup 程式設定為可執行

``` bash
# 要將檔案放到 ESXi 重開機後不會回復初始設定的路徑(可以是任何 DataStore 的目錄下，只要是 persistent folder 即可)
# 切換到 datastore1 folder，避免 ESXi 重開機之後將檔案刪除
$ cd /vmfs/volumes/datastore1
$ wget http://sourceforge.net/projects/xsibackup/files/xsibackup_4.1.6.zip/download -O xsibackup.zip
$ unzip xsibackup.zip
$ chmod 0700 xsibackup*
```

3、執行備份工作

``` bash
# 備份檔存放路徑：/vmfs/volumes/backup
# 備份類型：目前運行中的 VM (running)
# mail & smpt 的相關設定都是與寄信相關
$ ./xsibackup --backup-point=/vmfs/volumes/backup --backup-type=running 
--mail-from=email.sender@yourdomain.com --mail-to=email.recipient@anotherdomain.com 
--smtp-srv=smtp.yourdomain.com --smtp-port=25 --smtp-usr=username 
--smtp-pwd=password
```
`/vmfs/volumes/backup 目錄也可以是 remote host 所提供的 NFS share folder`

其中 `--backup-type` 有以下三種：
- **all** (所有 vm)
- **running** (執行中的 vm)
- **custom** (指定 vm，需搭配 --backup-vms 參數指令要備份的 vm，多個 vm 可用逗號隔開)

custom 應用如下：

``` bash
# 指定備份 WINDOWSVM1 & LINUXVM2 兩台 vm
$ ./xsibackup --backup-point=/vmfs/volumes/backup --backup-type=custom --backup-vms=WINDOWSVM1,LINUXVM2
```

## 其他參數

- `--test-mode=true` (測試模式，不實際進行備份；但若有設定 EMail 相關參數則會發信)
- `--backup-how (hot | cold)` (hot 會在 vm 開機情況下備份，cold 則會將 vm 關機後再備份)`

## 寄送 Mail 的問題

設定了 EMail 發送相關參數後，實際執行會發現竟然不行，排除方法如下：

xsibackup 程式會在 **/etc/vmware/firewall/service.xml** 這個檔案補上這一段內容：

``` xml
<service id='9999'>
	<id>SMTPout</id>
	<rule id='0000'>
		<direction>outbound</direction>
		<protocol>tcp</protocol>
		<porttype>dst</porttype>
		<port></port>
	</rule>
	<enabled>true</enabled>
	<required>false</required>
</service>
```

但其實這是錯誤的，要把 `<port></port>`這個部分改成 `<port>25</port>`，並執行以下指令：

``` bash
$ esxcli network firewall refresh
```

如此一來 EMail 的功能就會正常啟動了!


## 排程備份

xsibackup 也支援排程喔! 設定方式如下：

1. 在 ESXi 主機上執行 `xsibackup --install-cron` 指令，此時會在 **/vmfs/volumes/datastore1** 目錄中產生 `xsibackup-cron`這個檔案，可以直接進入編輯：(若是星期一、五晚上 20:00 要備份)

``` bash
# 加入 --time 參數，格式為 --time="Day HH:mm"(注意這邊要用 UTC 時間)
# 星期一 20:00 備份
/vmfs/volumes/datastore1/xsibackup --time="Mon 12:00" --backup-point=/vmfs/volumes/backup --backup-type=running --mail-from=email.sender@yourdomain.com --mail-to=email.recipient@anotherdomain.com --smtp-srv=smtp.yourdomain.com --smtp-port=25 --smtp-usr=username --smtp-pwd=password
# 星期五 20:00 備份
/vmfs/volumes/datastore1/xsibackup --time="Fri 12:00" --backup-point=/vmfs/volumes/backup --backup-type=running --mail-from=email.sender@yourdomain.com --mail-to=email.recipient@anotherdomain.com --smtp-srv=smtp.yourdomain.com --smtp-port=25 --smtp-usr=username --smtp-pwd=password
```

2. 重新啟動 ESXi Host 讓 cron 的功能啟用


## 總結

因此總結一下，優缺點大致如下：

### 優點

1. 免費、開放

2. 可進行完整備份，非特殊格式，不需要透過其他軟體還原

3. 在單純的環境下使用簡單


### 缺點

1. 無法執行差異備份，自然也就沒有 dedupication 的功能。

2. 目前沒有 exclude 的參數，若是有不想備份的 VM(例如：VDP)，只能透過 custom or running(搭配將 vm 關機)的方式來完成 (也可以透過改 source code 的方式來做....)

3. 若是 vSphere 授權版本有 DRS(Dynamic Resource Scheduler) 的話，VM 可能會隨著資源耗損不同而跑來跑去，備份工作就很難透過 custom 方式來達成。


## 參考資料

- [Free Backup Software for VMware ESXi VMs | SourceForge.net](http://sourceforge.net/projects/xsibackup/)

- [Add outbound port 25 for SMTP in VMware ESXi v5 – depicus](http://blog.depicus.com/add-outbound-port-25-for-smtp-in-vmware-esxi-v5/)

- [VMware KB: Creating custom firewall rules in VMware ESXi 5.x](http://kb.vmware.com/selfservice/microsites/search.do?language=en_US&cmd=displayKC&externalId=2008226)