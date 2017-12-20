---
layout: post
title:  "[CoreOS] etcd、fleet & flannel 簡介"
description: "This article introduces the concepts of CoreOS main components - etcd、fleet & flannel"
date: 2016-06-22 14:30:00
published: true
comments: true
categories: [CoreOS]
tags: [Linux, CoreOS]
---


環境說明
=======

測試環境建立於現有的 OpenStack infrastructure 上，使用以下設定：

```yaml
#cloud-config

ssh_authorized_keys:
  - [YOUR SSH PUBLIC KEY]

coreos:
  etcd2:
    discovery: https://discovery.etcd.io/ea524a83ed3f84e550417f0fb89e91c8
    advertise-client-urls: http://$private_ipv4:2379,http://$private_ipv4:4001
    initial-advertise-peer-urls: http://$private_ipv4:2380
    listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
    listen-peer-urls: http://$private_ipv4:2380
  flannel:
    interface: $public_ipv4

  units:
    - name: etcd2.service
      command: start
    - name: fleet.service
      command: start
    - name: flanneld.service
      drop-ins:
        - name: 50-network-config.conf
          content: |
            [Service]
            ExecStartPre=/usr/bin/etcdctl set /coreos.com/network/config '{ "Network": "10.1.0.0/16" }'
      command: start
    - name: redis.service
      content: |
        [Unit]
        Requires=flanneld.service
        After=flanneld.service
        [Service]
        ExecStart=/usr/bin/docker run redis
        Restart=always
      command: start
```

以上設定會把 **etcd**、**fleet**、**flannel**、**docker** 都啟動，還包含了一個執行 **redis** 服務的 container

------------------------------------------------------------------------

驗證步驟
=======

當 CoreOS 安裝完成之後，首要之務就是驗證 CoreOS 是否安裝成功，以下測試三個主要的 service 分別是 `etcd`、`docker`、`fleet`：

## etcd

在 Machine A 新增資訊到 etcd service：

```bash
core@coreos ~ $ etcdctl set /message "Hello CoreOS"
Hello CoreOS

core@coreos ~ $ curl -L http://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello CoreOS"
{"action":"set","node":{"key":"/message","value":"Hello CoreOS","modifiedIndex":1641,"createdIndex":1641},"prevNode":{"key":"/message","value":"Hello CoreOS","modifiedIndex":1455,"createdIndex":1455}}
```

可以從 Machine B 取得：

```bash
core@coreos ~ $ etcdctl get /message   
Hello CoreOS

core@coreos ~ $ curl -L http://127.0.0.1:2379/v2/keys/message
{"action":"get","node":{"key":"/message","value":"Hello CoreOS","modifiedIndex":1641,"createdIndex":1641}}
```

## docker

由於 CoreOS 使用了 systemd port activate 的機制，因此一開始 docker service 是不會啟動的，直到有 container 啟動為止：

```bash
# 一開始 docker service 不會啟動
core@coreos ~ $ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib64/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: http://docs.docker.com

# 啟動 container busybox
core@coreos ~ $ docker run busybox /bin/echo Hello CoreOS
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
385e281300cc: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:4a731fb46adc5cefe3ae374a8b6020fc1b6ad667a279647766e9a3cd89f6fa92
Status: Downloaded newer image for busybox:latest
Hello CoreOS

# docker service 目前為啟動狀態
core@coreos ~ $ systemctl status docker
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib64/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2016-06-20 07:58:52 UTC; 53s ago
     Docs: http://docs.docker.com
 Main PID: 1082 (docker)
   Memory: 41.7M
      CPU: 18.356s
   CGroup: /system.slice/docker.service
           └─1082 docker daemon --host=fd:// --exec-opt native.cgroupdriver=systemd --selinux-enabled
.........
```

## fleet

CoreOS 會使用 fleet 用來管理 container 的生命週期。

fleet 是透過接收 [systemd unit files](https://coreos.com/os/docs/latest/getting-started-with-systemd.html) 為資訊來源進行工作，將 workload 分配到 cluster 中不同的機器執行；管理者可以透過 `fleetctl` 工具執行像是查詢 unit 的狀態、log 資訊等工作

```bash
core@coreos ~ $ cat <<EOF >hello.service
[Unit]
Description=My Service
After=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill hello
ExecStartPre=-/usr/bin/docker rm hello
ExecStartPre=/usr/bin/docker pull busybox
ExecStart=/usr/bin/docker run --name hello busybox /bin/sh -c "trap 'exit 0' INT TERM; while true; do echo Hello World; sleep 1; done"
ExecStop=/usr/bin/docker stop hello
EOF

core@coreos ~ $ fleetctl load hello.service
Unit hello.service inactive
Unit hello.service loaded on 0682a8f2.../192.168.50.14

core@coreos ~ $ fleetctl start hello.service
Unit hello.service launched on 0682a8f2.../192.168.50.14

core@coreos ~ $ fleetctl status hello.service
● hello.service - My Service
   Loaded: loaded (/run/fleet/units/hello.service; linked-runtime; vendor preset: disabled)
   Active: active (running) since Mon 2016-06-20 08:52:43 UTC; 9s ago
...........
Jun 20 08:52:43 coreos docker[1355]: Status: Image is up to date for busybox:latest
Jun 20 08:52:43 coreos systemd[1]: Started My Service.
Jun 20 08:52:50 coreos docker[1367]: Hello World
...........
Jun 20 08:52:52 coreos docker[1367]: Hello World

core@coreos ~ $ fleetctl destroy hello.service
Destroyed hello.service

core@coreos ~ $ fleetctl status hello.service
Unit hello.service does not exist.
```

## flannel

flannel 是用來提供 cross hosts 的 container 互相連結之用，在上面的 cloud-config 中，每個 host 預設會從 10.1.0.0/16 的區段中取得自己的 class C subnet。

驗證 flannel 是否運作正常的的步驟如下：

1. 登入 machine A，並進入 redis container A 中，查詢到 IP A

2. 登入 machine B，並進入 redis container B 中，查詢到 IP B

3. 確認可以從 container A ping 到 container B

若是兩個位於不同機器的 container 可以互通，就表示 flannel 目前的運作是正常的!

------------------------------------------------------------------------

etcd
====

**etcd** 是種分散式 key/value 儲存服務，存在於每一台 CoreOS 機器中，並在 CoreOS cluster 中負責 shared configuration & service discovery 的工作。

而執行在 cluster 環境中的 application container 同樣可以使用 etcd 所提供的服務，可進行像是儲存資料庫連線設定、cache 設定、[feature flag](Feature Flag 功能發布控制 - 壹讀) ... 等不同的資訊。

## 在 etcd 讀寫資料

當對 etcd service 寫入資料後，cluster 內的機器都可以讀取的到(所以說是分散式的)

### 寫入資料

```bash
core@coreos ~ $ etcdctl set /message Hello
Hello

core@coreos ~ $ curl -L -X PUT http://localhost:2379/v2/keys/message -d value="Hello"
{"action":"set","node":{"key":"/message","value":"Hello","modifiedIndex":45040,"createdIndex":45040},"prevNode":{"key":"/message","value":"CoreOS","modifiedIndex":44979,"createdIndex":44979}}
```

### 讀取資料

```bash
core@coreos ~ $ etcdctl get /message
Hello

core@coreos ~ $ curl -L http://localhost:2379/v2/keys/message
{"action":"get","node":{"key":"/message","value":"Hello","modifiedIndex":45040,"createdIndex":45040}}
```

### 刪除資料

```bash
core@coreos ~ $ etcdctl rm /message
PrevNode.Value: Hello

core@coreos ~ $ etcdctl set /message Hello
Hello

core@coreos ~ $ curl -L -X DELETE http://localhost:2379/v2/keys/message
{"action":"delete","node":{"key":"/message","modifiedIndex":45482,"createdIndex":45467},"prevNode":{"key":"/message","value":"Hello","modifiedIndex":45467,"createdIndex":45467}}

```

## 從 container 中讀寫 etcd 中的資料

要從 docker container 中對 etcd 讀寫資料，必須將 ip 指向 `docker0` interface，預設情況下 ip 為 `172.17.0.1`

```bash
root@cb8e1cb26a7e:/# curl -L http://172.17.0.1:2379/v2/keys
{"action":"get","node":{"dir":true,"nodes":[{"key":"/coreos.com","dir":true,"modifiedIndex":5,"createdIndex":5}]}}

root@cb8e1cb26a7e:/# curl -L -X PUT http://172.17.0.1:2379/v2/keys/message -d value="I am container"
{"action":"set","node":{"key":"/message","value":"I am container","modifiedIndex":47075,"createdIndex":47075}}
```

## 監控 etcd directory 的變化

### 1、新增 etcd directory

除了 `/` 之外，我們可以在 etcd 中自訂所需要的 directory 用來存放自訂的訊息，以下建立名稱為 **foo-service** 的 directory：

```bash
core@coreos ~ $ etcdctl mkdir /foo-service

# 在 etcd directory 中新增訊息
core@coreos ~ $ etcdctl set /foo-service/container1 localhost:1111
localhost:1111
```

### 2、監控 etcd directory

當其他 CoreOS host 執行 `etcdctl set /foo-service/container2 localhost:2222`：

```bash
core@coreos ~ $ etcdctl watch --recursive /foo-service
[set] /foo-service/container2
localhost:2222

core@coreos ~ $ curl -L http://localhost:2379/v2/keys/foo-service?wait=true\&recursive=true
{"action":"set","node":{"key":"/foo-service/container2","value":"localhost:2222","modifiedIndex":48468,"createdIndex":48468}}
```

### 3、監控 etcd directory 並觸發執行特定任務

當其他 CoreOS host 執行執行以下指令：

1. `etcdctl set /foo-service/container2 localhost:2222`

2. `etcdctl set /foo-service/container3 localhost:3333`

3. `etcdctl rm /foo-service/container2`

4. `etcdctl rm /foo-service/container3`

```bash
core@coreos ~ $ etcdctl exec-watch --recursive /foo-service -- sh -c 'echo "\"$ETCD_WATCH_KEY\" key was updated to \"$ETCD_WATCH_VALUE\" value by \"$ETCD_WATCH_ACTION\" action"'
"/foo-service/container2" key was updated to "localhost:2222" value by "set" action
"/foo-service/container3" key was updated to "localhost:3333" value by "set" action
"/foo-service/container2" key was updated to "" value by "delete" action
"/foo-service/container3" key was updated to "" value by "delete" action
```

## Test and Set

etcd 也可作為中央協調服務，提供 **TestAndSet** 功能；設定資料的人必須提供先前的值，有符合才可以變更值的內容

```bash
# 設定值為 Hello
core@coreos ~ $ etcdctl set /message "Hello"
Hello

# 若是先前的值為 123，將值換為 Hi (操作失敗)
core@coreos ~ $ etcdctl set /message "Hi" --swap-with-value="123"
Error:  101: Compare failed ([123 != Hello]) [48752]

# 若是先前的值為 Hello，將值換為 Hi (操作成功)
core@coreos ~ $ etcdctl set /message "Hi" --swap-with-value="Hello"
Hi

# 新值已經變成 Hi
core@coreos ~ $ etcdctl get /message
Hi
```

## TTL

將資料寫入 etcd 時，還可以設定 TTL，讓資料在一定的時間之後失效：

```bash
core@coreos ~ $ etcdctl set /foo "Expiring Soon" --ttl 10
Expiring Soon
core@coreos ~ $ etcdctl get /foo
Expiring Soon
core@coreos ~ $ etcdctl get /foo
Expiring Soon
.........
core@coreos ~ $ etcdctl get /foo
Error:  100: Key not found (/foo) [49020]
```

------------------------------------------------------------------------

systemd
=======

systemd 提供了許多強大的功能作為啟動、停止、管理 process 之用，在 CoreOS 中，幾乎所有的 Docker container 的生命週期都是由 systemd 在管理的。

CoreOS 預設所使用的 systemd target 為 `multi-user.target`，裡面包含了所有常用來管理 container 的 `systemd unit`。(**target 由多個 unit symbolic link 所集合而成**)

> systemd unit 是一個描述管理者所要執行的 process 的設定檔

以下用一個簡單範例 `/etc/systemd/system/myweb.service` 來說明：

```ini
[Unit]
Description=My Web Service
After=etcd2.service
After=docker.service

[Service]
TimeoutStartSec=0
; 在 ExecStart 所要執行的命令，"=-" 表示請 systemd 忽略執行命令時所發生的錯誤
ExecStartPre=-/usr/bin/docker kill apache1
ExecStartPre=-/usr/bin/docker rm apache1
ExecStartPre=/usr/bin/docker pull coreos/apache
; service 啟動時所要執行的命令
ExecStart=/usr/bin/docker run --name apache1 -p 8081:80 coreos/apache /usr/sbin/apache2ctl -D FOREGROUND
; service 啟動後要執行的指令 (以 machine ID 為 key，寫入訊息到 etcd /domains/example)
ExecStartPost=/usr/bin/etcdctl set /domains/example.com/%m running
; service 停止時所要執行的指令 (停止 container apache1)
ExecStop=/usr/bin/docker stop apache1
; service 停止後所要執行的步驟 (以 machine ID 為 key，移除 etcd /domains/example 上的訊息)
ExecStopPost=/usr/bin/etcdctl rm /domains/example.com/%m

[Install]
WantedBy=multi-user.target
```

> 完整的設定參數可參考 => [systemd.service](https://www.freedesktop.org/software/systemd/man/systemd.service.html)

另外還可以在 system unit configuration 檔案中指定 **specifiers**，例如：

- `%n`：Full Unit Name

- `%i`：Instance Name

- `%m`：Machine ID

- `%H`：Host Name

詳細的資料可以參考 => [systemd.unit - Specifiers](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Specifiers)

上述的 systemd service unit configuration 執行的結果如下：

```bash
core@coreos ~ $ sudo systemctl enable myweb.service
Created symlink from /etc/systemd/system/multi-user.target.wants/myweb.service to /etc/systemd/system/myweb.service.
core@coreos ~ $ sudo systemctl start myweb.service

core@coreos ~ $ sudo systemctl status myweb.service
● myweb.service - My Web Service
   Loaded: loaded (/etc/systemd/system/myweb.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2016-06-21 07:36:22 UTC; 9s ago
  Process: 2403 ExecStartPost=/usr/bin/etcdctl set /domains/example.com/%m running (code=exited, status=0/SUCCESS)
  Process: 2391 ExecStartPre=/usr/bin/docker pull coreos/apache (code=exited, status=0/SUCCESS)
  Process: 2379 ExecStartPre=/usr/bin/docker rm apache1 (code=exited, status=0/SUCCESS)
  Process: 2369 ExecStartPre=/usr/bin/docker kill apache1 (code=exited, status=1/FAILURE)
 Main PID: 2402 (docker)
   Memory: 11.5M
      CPU: 5.695s
   CGroup: /system.slice/myweb.service
           └─2402 /usr/bin/docker run --name apache1 -p 8081:80 coreos/apache /usr/sbin/apache2ctl -D FOREGROUND
......
Jun 21 07:36:19 coreos docker[2391]: Status: Image is up to date for coreos/apache:latest
Jun 21 07:36:22 coreos etcdctl[2403]: running
Jun 21 07:36:22 coreos systemd[1]: Started My Web Service.

# 查詢 container 運作狀態
core@coreos ~ $ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
441cf4a6bf54        coreos/apache       "/usr/sbin/apache2ctl"   35 seconds ago      Up 26 seconds       0.0.0.0:8081->80/tcp   apache1

# 確認 web server 正常服務
core@coreos ~ $ curl http://localhost:8081
<html><body><h1>It works!</h1>
<p>This is the default web page for this server.</p>
<p>The web server software is running but no content has been added, yet.</p>
</body></html>

# 查詢 systemd 寫入 etcd 中的資料
core@coreos ~ $ etcdctl ls /domains/example.com
/domains/example.com/72bda15890c14ee89b15214c1b87d71f
core@coreos ~ $ etcdctl get /domains/example.com/72bda15890c14ee89b15214c1b87d71f
running

# 停止 service
core@coreos ~ $ sudo systemctl stop myweb.service

# 查詢 container 狀態
core@coreos ~ $ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS                        PORTS               NAMES
441cf4a6bf54        coreos/apache       "/usr/sbin/apache2ctl"   About a minute ago   Exited (137) 13 seconds ago                       apache1

# 查詢 systemd 寫入 etcd 中的資料 (已經移除)
core@coreos ~ $ etcdctl ls /domains/example.com
```

------------------------------------------------------------------------

fleet
=====

fleet 是個 cluster manager，用途是在 cluster level 上管理 systemd，因此要在 CoreOS cluster 上運行相關服務，正確的方式就是"**使用標準的 systemd unit 搭配 fleet**"來達成。

## fleet unit type

可以運行在 cluster 上的 unit 有兩種，分別是：

1. **standard unit**：會被分派到獨立的單一主機上長時間運行的 process，若是機器掛點了，standard unit 就會被轉移到其他正常的主機上重新啟動並繼續執行

2. **global unit**：global unit 將會運行在 cluster 中所有的機器上，這種 unit 適合用在像是 monitoring agent，甚至像是 high level 的 orchestration tool，例如：Kubernetes、Mesos、OpenStack .... 等等。

## 在 cluster 中啟動 container

在這邊我們利用之前寫好的 `/etc/systemd/system/myweb.service` 作示範：

```bash
core@coreos ~ $ fleetctl start /etc/systemd/system/myweb.service
Unit myweb.service inactive
Unit myweb.service launched on 0682a8f2.../192.168.50.14

core@coreos ~ $ fleetctl list-units   
UNIT		MACHINE				ACTIVE		SUB
myweb.service	0682a8f2.../192.168.50.14	activating	start-pre

(...... it takes some time to wait .......)

core@coreos ~ $ fleetctl list-units
UNIT		MACHINE				ACTIVE	SUB
myweb.service	0682a8f2.../192.168.50.14	active	running

core@coreos ~ $ curl http://192.168.50.14:8081
<html><body><h1>It works!</h1>
<p>This is the default web page for this server.</p>
<p>The web server software is running but no content has been added, yet.</p>
</body></html>

core@coreos ~ $ fleetctl list-machines
MACHINE		IP		METADATA
0682a8f2...	192.168.50.14	-
72bda158...	192.168.50.16	-
90c1e40c...	192.168.50.15	-
```

## 運行 High Availability Service

關於 fleet unit 的詳細設定方式，可以參考 => [fleet - Overview of Unit Files and Scheduling](https://coreos.com/fleet/docs/latest/unit-files-and-scheduling.html)

以下將會設計一個具有 High Availability 的 web service，其中較為重要的設定就是 `[X-Fleet]` 區塊部份的設定：

```bash
core@coreos ~/demo $ cat <<EOF >apache@.service
[Unit]
Description=My Apache Frontend
After=docker.service
Requires=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill apache1
ExecStartPre=-/usr/bin/docker rm apache1
ExecStartPre=/usr/bin/docker pull coreos/apache
ExecStart=/usr/bin/docker run --rm --name apache1 -p 80:80 coreos/apache /usr/sbin/apache2ctl -D FOREGROUND
ExecStop=/usr/bin/docker stop apache1

[X-Fleet]
; 告知 fleet 不要把 apache@* 的 service 放在同一台機器上
Conflicts=apache@*.service
EOF

# 複製兩份 unit configuration file
core@coreos ~/demo $ cp apache\@.service apache\@1.service
core@coreos ~/demo $ cp apache\@.service apache\@2.service
core@coreos ~/demo $ ls  
apache@.service  apache@1.service  apache@2.service

# 啟動第 1 個 apache container
core@coreos ~/demo $ fleetctl start apache@1
Unit apache@1.service inactive
Unit apache@1.service launched on 0682a8f2.../192.168.50.14

# 啟動第 2 個 apache container
core@coreos ~/demo $ fleetctl start apache@2
Unit apache@2.service inactive
Unit apache@2.service launched on 72bda158.../192.168.50.16

# 檢視 units
core@coreos ~/demo $ fleetctl list-units
UNIT			MACHINE				ACTIVE	SUB
apache@1.service	0682a8f2.../192.168.50.14	active	running
apache@2.service	72bda158.../192.168.50.16	active	running


# 驗證 web 服務有正確啟動
core@coreos ~/demo $ curl http://192.168.50.14
<html><body><h1>It works!</h1>
<p>This is the default web page for this server.</p>
<p>The web server software is running but no content has been added, yet.</p>
</body></html>
core@coreos ~/demo $ curl http://192.168.50.16
<html><body><h1>It works!</h1>
<p>This is the default web page for this server.</p>
<p>The web server software is running but no content has been added, yet.</p>
</body></html>
```

接著把 `192.168.50.14` 這台機器重新開機，此時 service 將會被 fleet 轉移到正常的機器上：

```bash
core@coreos ~/demo $ fleetctl list-units
UNIT			MACHINE				ACTIVE		SUB
apache@1.service	90c1e40c.../192.168.50.15	activating	start-pre
apache@2.service	72bda158.../192.168.50.16	active		running

core@coreos ~/demo $ fleetctl list-units
UNIT			MACHINE				ACTIVE		SUB
apache@1.service	90c1e40c.../192.168.50.15	activating	start-pre
apache@2.service	72bda158.../192.168.50.16	active		running

core@coreos ~/demo $ fleetctl list-units
UNIT			MACHINE				ACTIVE	SUB
apache@1.service	90c1e40c.../192.168.50.15	active	running
apache@2.service	72bda158.../192.168.50.16	active	running
```

> 存取這些 HA service 比較好的方式是透過 **sidekick container**，這類的 container 是用來提供像是 service discovery、load balancer、DNS 等工作，不建議直接存取 service container

## 運行 sidekick

```bash
core@coreos ~/demo $ cat <<EOF >apache_discovery@.service
[Unit]
Description=Announce Apache1
BindsTo=apache@%i.service
After=apache@%i.service

[Service]
ExecStart=/bin/sh -c "while true; do etcdctl set /services/website/apache@%i '{ \"machine\": \"%m\", \"port\": 80, \"version\": \"52c7248a14\" }' --ttl 60;sleep 45;done"
ExecStop=/usr/bin/etcdctl rm /services/website/apache@%i

[X-Fleet]
MachineOf=apache@%i.service
EOF
core@coreos ~/demo $ cp apache_discovery\@.service apache_discovery\@1.service
core@coreos ~/demo $ cp apache_discovery\@.service apache_discovery\@2.service
core@coreos ~/demo $ ls
apache@.service  apache@1.service  apache@2.service  apache_discovery@.service  apache_discovery@1.service  apache_discovery@2.service

# 啟動 service discovery
core@coreos ~/demo $ fleetctl start apache_discovery@1         
Unit apache_discovery@1.service inactive
Unit apache_discovery@1.service launched on 90c1e40c.../192.168.50.15
# 啟動 service discovery
core@coreos ~/demo $ fleetctl start apache_discovery@2
Unit apache_discovery@2.service inactive
Unit apache_discovery@2.service launched on 72bda158.../192.168.50.16


core@coreos ~/demo $ fleetctl list-units
UNIT				MACHINE				ACTIVE	SUB
apache@1.service		90c1e40c.../192.168.50.15	active	running
apache@2.service		72bda158.../192.168.50.16	active	running
apache_discovery@1.service	90c1e40c.../192.168.50.15	active	running
apache_discovery@2.service	72bda158.../192.168.50.16	active	running

core@coreos ~/demo $ etcdctl ls /services/ --recursive
/services/website
/services/website/apache@1
/services/website/apache@2

core@coreos ~/demo $ etcdctl get /services/website/apache@1
{ "machine": "90c1e40ca1e749abbc23fa8b2391ce27", "port": 80, "version": "52c7248a14" }
```

## Global Unit

Global Unit 與其他一般的 unit 沒什麼太大差別，只有在 `[X-Fleet]`區段中多加了 `Global=true` 設定而已，如此一來這個 system unit 就會在所有機器上執行

> 若要把 Global Unit 執行在特定幾台機器上，可搭配 Machine Metadata 使用，就可以達到此目的

## Machine Metadata

Mechine Metadata 的來源是從 cloud-config 所設定的，設定的語法如下：

> metadata="platform=metal,provider=rackspace,region=east,disk=ssd"

接著 Machine Metadata 會設定在 unit configuration 中，以下是幾個範例：

```bash
[X-Fleet]
MachineMetadata=disk=ssd
```

```bash
[X-Fleet]
Conflicts=webapp*
MachineMetadata=provider=rackspace
MachineMetadata=platform=metal
MachineMetadata=region=east
```

------------------------------------------------------------------------

References
==========

- [CoreOS Quick Start](https://coreos.com/os/docs/latest/quickstart.html)

- [Getting Started with etcd on CoreOS](https://coreos.com/etcd/docs/latest/getting-started-with-etcd.html)

- [Getting Started with docker](https://coreos.com/os/docs/latest/getting-started-with-docker.html)

- [Getting Started with systemd](https://coreos.com/os/docs/latest/getting-started-with-systemd.html)

- [Launching Containers with fleet](https://coreos.com/fleet/docs/latest/launching-containers-fleet.html)

- [unit-examples/simple-fleet at master · coreos/unit-examples](https://github.com/coreos/unit-examples/tree/master/simple-fleet)

- [fleet - Overview of Unit Files and Scheduling](https://coreos.com/fleet/docs/latest/unit-files-and-scheduling.html)

- [Customize with Cloud-Config](https://coreos.com/os/docs/latest/cloud-config.html)

- [Configuring flannel Networking for CoreOS](https://coreos.com/flannel/docs/latest/flannel-config.html)

- [DockOne技术分享（十八）：一篇文章带你了解Flannel - DockOne.io](http://dockone.io/article/618)
