---
layout: post
title:  "AWS SOA 學習筆記 - Storage Service(Database)"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 Database 相關服務(RDS, RDS Proxy, Aurora, Elasticache)在實際操作維運上需要注意的事項"
date: 2021-10-31 13:50:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - SOA
  - RDS
  - RDS Proxy
  - Aurora
  - Elasticache
---


RDS
===

## Encryption & Security

RDS Encryption 分為兩個部份，分別是 `at-rest encrption` & `in-flight encryption`，兩者的思路跟作法其實都不太一樣

### at-rest encrption

這部份屬於資料內容的加密：

- 與 KMS 服務搭配，使用 AES-256 加密 (for master & read replica)

- encryption 功能在 RDS instance 新增啟動時就要決定是否啟用

- 若 master DB 沒有設定加密，那 replica DB 也無法

- Oracle & SQL Server 另外還有自己本身支援的 TDE(Transparent Data Encryption) 可用

### in-flight encryption

這部份屬於傳輸過程中的加密：

- 透過 SSL certification 將傳遞到 RDS 過程中的資料加密

- 要啟用 SSL，連線到 DB 時必須明確指定 (正確操作方式可參考[文件說明](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html))


## 對 RDS Instance 執行加密操作

執行加密操作需要搭配 snapshot 的功能，首先要了解 snapshot & encryption 之間的特性，會比較清楚正確的流程該如何進行：

- 未加密的 RDS instance 所產生的 snapshot 也是未加密的型態

- 已加密的 RDS instance 所產生的 snapshot 也是已加密的型態

- 未加密的 snapshot 可以轉換為已加密的 snapshot 

了解上面的特性後，要將一個未加密的 RDS instance 轉換為加密的 RDS instance 流程如下：

1. 透過未加密的 RDS instance 建立一個 snapshot

2. snapshot 透過 COPY 的操作轉換成已加密的 snapshot

3. 將已加密的 snapshot 還原成已加密 RDS instance

4. 將 application 端的連線改到新的 RDS instance 上


## RDS & IAM

在 IAM 的部份主要作為 Access Management 之用，分為兩個部份：

- 可透過 IAM policy 來控制誰可以管理 RDS instance (需要透過 RDS API)

- 以 IAM 的方式進行 DB 連線登入的認證 (僅 MySQL & PostgreSQL 支援)

### IAM Authentication

![RDS IAM authentication](/blog/images/aws/DB/RDS_IAM-authentication.png)

使用 RDS IAM Authentication 需要注意的是：

- 僅適用於 MySQL & PostgreSQL

- 不需要密碼，只要透過 IAM & RDS API call 取得認證用 token 即可 (token 在 15 分鐘內有效)

可以帶來以下好處：

- cliet 與 RDS instance 之間的連線必須透過 SSL 加密

- 可統一透過 IAM 管理權限


## Parameter Group

- [Parameter Group](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithParamGroups.html) 是調整 RDS instance 運作行為的方法

- Parameter Group 中有很多設定，分為 dynamic & static 兩種；**其中 dynamic 修改後立即生效，而 static 修改後需要 reboot RDS instance 才會生效**

- 修改 RDS instance 所使用的 Parameter Group 也需要 reboot 才會生效

### 應考須知

- 在 PostgreSQL & SQL Server 中，可透過 Parameter Group 中的 `rds.force_ssl=1` 來強制要求使用 SSL connection

- 上面的參數對 MySQL & MariaDB 無效，需要透過內部的權限設定來處理 (類似下方的設定方式)
> GRANT SELECT on mydb.* TO 'myuser@%' IDENTIFIED BY 'xxxx' REQUIRE SSL;



RDS Proxy
=========

這裡以 Lambda Function 連接 RDS instance 的角度來出發，說明 RDS proxy 的定位 & 可帶來的優點。

首先先了解一下使用 Lambda Function 會需要考慮到以下問題：

- Lambda Function 預設放在 AWS 自行管理的 VPC 中，並非我們自己所建立的 VPC，若要存取 RDS instance，就需要考慮到 public subnet 來存取位於 private subnet RDS instance 的問題
> 問題可以透過將 Lambda Function 放到我們自己管理的 VPC 來解決

- Lambda Function 若是執行數量很多，很有可能遇到 RDS instance 出現 too many connection 的錯誤

而上面的問題，透過 [RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html) 可以很容易被解決：

![RDS Proxy](/blog/images/aws/DB/RDS_Proxy.png)

- 可將 RDS Proxy 放到 VPC 中的 public subnet，而 RDS instance 僅允許來自 RDS Proxy 的連線

- Lambda Function 改連 RDS Proxy

透過上面的方式，可以帶來以下好處：

- RDS Proxy 可將認證管理的部份包一層，並統一對外改由 IAM authentication 的方式來作為連線認證方式

- RDS Proxy 具備 auto-scaling 功能

- Lambda Function 可以不用管理 idle connection 的問題，交給 RDS Proxy 處理即可



Aurora
======

## Multi-AZ

![Aurora Multi-AZ](/blog/images/aws/DB/Aurora_Multi-AZ.png)

- 從上圖可看出，Aurora shared storage volume 提供了 Replication + Self Healing + Auto Expanding .... 等功能

- Aurora 的效能來自於將 storage 拆分成超過 100 個 volume 來達成

- automatic failover 可以在 30 秒內完成

## Backup / Backtracking / Restore

- Automatic Backup 的部份跟 RDS 是一樣的

- backtracking 的機制可以讓 Aurora 指定 在72 小時之的時間範圍內還原

- 這功能目前只有 Aurora MySQL 有提供

## Database Cloning

- 透過原本 cluster 使用的 cluster volume 來建立新的 DB cluster

- 使用 `copy-on-write` 的技術實現此功能，因此速度極快

- 適合用來為測試環境產生一份來自 production 的資料時使用

## Failover Priority

- 每個 Aurora read replica 都有一直 priority tier 的設定，分別從 0~15

- 當 write instance 發生問題時，RDS 會自動 promote read replica 為 write instance，而 priority tier 越小優先權越大

- 若 priority tier 相同，那會選擇 size 最大的

- 若 priority tier & size 都相同，那就隨機挑一個



Elasticache
===========

## Redis v.s. Memcached 

目前 AWS Elasticache 提供了 Redis & Memcached 兩個選擇，以下是官方文件給出的比較表：

![Elasticache comparision](/blog/images/aws/DB/Aurora_Multi-AZ.png)


## Redis Replication with Cluster Mode Enabled(Disabled)

因為只有 Redis 有 replication 的功能，因此這部份的說明就以 Redis 為主

### cluster mode disabled

![Elasticache Redis Replication - cluster mode disabled](/blog/images/aws/DB/Elasticache_replication-with-cluster-mode-disabled.png)

這是比較單純的模式，將 cluster mode 關閉的情況下，以下列出在 repication 的運作行為需要注意的地方：

- 非同步複製 (不論有沒有啟用 cluster mode 都是相同的)

- primary node 可用來 read/write，其他就只有 read-only

- 只會有一個 shard，其他所有的 replica node 都具備有一模一樣的資料

- Multi-AZ 是預設啟用的 (目的是為了 failover 用)

- 對提昇 read performance 是有幫助的

### cluster mode enabled

![Elasticache Redis Replication - cluster mode enabled](/blog/images/aws/DB/Elasticache_replication-with-cluster-mode-enabled.png)

cluster mode 開啟後事情會變得複雜些，以下列出在 replication 的運作行為需要注意的地方：

- 資料會根據 shard 的數量被切份，並平均分散在不同的 shard

- 承上，可作為提昇 write performance 的手段

- 每個 shard 都可以各自有最多 5 個 replica node

- 同樣支援 Multi-AZ

- 一個 cluster 最大支援 500 個 node
  - 500 shards (1 master)
  - 250 shards (1 master + 1 replica)
  - ...
  - 83 shards (1 master + 5 replicas)


## Redis Scaling with Cluster Mode Enabled(Disabled)

此部份也是一樣以 Redis 為主進行討論

### cluster mode disabled

進行水平擴展，只要簡單的新增/減少 read replica 即可。

若是垂直擴展，需要調整 node type，那 Elasticache 內部其實是建立一個新的 Node Group，完成資料同步後，再透過 DNS update 的方式完成(如下圖)

![Elasticache Redis scaling - cluster mode disabled](/blog/images/aws/DB/Elasticache_scaling-with-cluster-mode-disabled.png)

### cluster mode enabled

cluster mode 開啟的情況下，scaling 就會分成 Online/Offline scaling 兩種，其中 Online mode 不會有 downtime，但存取效能會稍微有影響，而 Offline mode 則會有 downtime(但可支援更多類型的調整，例如：更換 node type，升級 engine 版本 ... 等等)。

進行水平擴展有以下方式：
- Resharding：增加 or 移除 shard
- Shard Rebalancing：讓 shard 的分佈更平均
- 這個可選擇 online scaling or offline scaling

進行垂直擴展的部份一樣是調整 node type，但 cluster mode enabled 情況下可以作到 online scaling


## Memcached Scaling

- Memcached cluster 可包含 node 數量為 1~40

- 水平擴展很單純，就是單純新增(or 移除) node，只要 [Auto Discovery](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/AutoDiscovery.html) 功能有啟用，application(client) 就會自動找到新增的 node，不需要進行額外調整 

- 垂直擴展的流程是：
  1. 使用新的 node type 建立新的 cluster
  2. 更新 application 指向新的 cluster
  3. 移除舊的 cluster
> 需要注意的是，由於 Memcached 不會協助複製資料，因此新的 cluster 啟動實是完全沒資料的，application 需要再重新填入所需要的資料



References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)

- [What is Amazon Relational Database Service (Amazon RDS)? - Amazon Relational Database Service](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/)

