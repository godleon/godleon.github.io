---
layout: post
title:  "[AWS Elasticsearch] 建置 & 開始使用 AWS Elasticsearch service"
description: "此篇文章介紹如何開始使用 AWS Managed Elasticsearch service & 不同的網路配置在使用上有何優缺點，並驗證佈署成功並使用"
date: 2021-01-16 21:55:00
published: true
comments: true
categories:
  - Elasticsearch
tags:
  - Elasticsearch
  - AWS
---


Overview
========

Amazon Elasticsearch Service 是個 AWS 提供的全託管 Elasticsearch 服務，基本上想要使用這服務，原因當然是因為 AWS 會協助維護底層 infra 的部份，省下維運的心力，可以把時間主要用在使用 Elasticsearch 在不同的應用上。

Amazon Elasticsearch Service 有一些強大的特性，值得了解一下：

- 非常容易建置 & 設定：不論是透過 AWS Management Console、CLI、或是 terraform 這一類的 IaaS 工具都可以完成建置

- [不停機升級]((https://aws.amazon.com/blogs/database/in-place-version-upgrades-for-amazon-elasticsearch-service/))：透過 blue/green deployment 的方式達到不停機升級，不論是版本的更新或是硬碟容量的增加，升級過程都可以維持服務穩定

- 監控 & 告警：AWS 本身提供了不少為 ES cluster 量身打造的監控告警工具，可以方便使用者了解目前 ES cluster 的狀態，也可以透過內建的 [Alerting plugin](https://opendistro.github.io/for-elasticsearch/features/alerting.html) 進行告警的相關設定

- 內建 Kibana：Kibana 可以不用自己裝了，而且還可以跟 AWS 現有的認證機制整合


關於設定變更
==========

ES domain 會透過 blue/green deployment 的方式進行變更，因此可以將 downtime 降低到最少，如果升級的過程中失敗了，就不會切到新的 deployment 上去，算是非常安全的更新方式。

而有不少操作都會觸發 ES domain blue/green deployment，以下舉幾個例子：

- 升級 Elasticsearch 版本

- 調整 instance type

- 在沒有 dedicated master node 的情況下調整 data instance 數量

- 啟用或停止 dedicated master node

- 啟用/停用 Multi-AZ

- 調整 storage type、volume type、volume size

此外還有不少其他操作，詳細資訊可參考 AWS 官方文件 [Configuration Changes - Amazon Elasticsearch Service](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/es-managedomains-configuration-changes.html)。

需要注意的是，blue/green deployment 會讓設定調整過程中的 node 數量變成兩倍，因此有可能消耗 dedicated master node 大量的資源，因此若是使用 dedicated master node，進行設定調整時，就要注意一下是否有足的的資源空間可以處理設定的變更，避免影響正在運行中的服務。



關於 Software Update
====================

有些軟體更新是必要的，有些是選擇性的，AWS 管理的策略如下：

- 如果使用者不進行必要的更新，那一段時間後(通常為 2 周) AWS 會自動協助更新

- 如果 console 中沒看到 automatic deployment date，那就表示更新是選擇性的

但有些情況會造成軟體更新無法進行，例如：

- ES Domain 正在進行設定變更

- cluster status 為紅色

- 對於 request 回應大量的 5xx error

- split brain (要解決此問題，則是要使用合適數量的 dedicated master node)

- 發生 Amazon Cognito 服務整合問題時



High Availability 設計考量
=========================

- 跨 3 個 AZ 進行 ES domain 的佈署

- 使用目前世代的 instance type

- 使用三個 dedicated master node + 至少三個 data node

- 至少為 index 建立一個 replica (這是 index 的預設值)

基本上，要達成一個 AZ 掛掉卻沒有 downtime 的要求，要選擇三個 AZ，而是否有沒有用 dedicated master node 則沒有關係；但需要注意的是，一旦有一個 AZ 掛掉導致某個 node 無法提供服務，那剩下的 node 就必須要扛下所有負載，這個條件在資源利用率很緊繃的情況下可能會造成 ES domain 出問題。



Public Accessible Elasticsearch Domain
======================================

Amazon ES domain 其實就是 `ES cluster` + `settings` + `instance type` + `instance count` + `storage source` 的綜合體。

假設建立 ES domain 時在 Network Configuration 的部份選擇 `Public access`，那這個 ES Domain 就可以透過 Internet 連到，而建立 ES Domain 需要花費十幾分鐘，完成後就可以透過下面的指令進行測試。

假設 **master user** 選擇以帳號密碼的方式進行建立，那就可以透過 `HTTP Basic Authetication` 的方式進行認證，可透過以下的測試指令進行驗證：

```bash
$ export ES_DOMAIN="https://[YOUR_ES_DOMAIN_ENDPINT]"

# 單筆資料寫入
$ curl -XPUT -u 'master_username:master_password' "${ES_DOMAIN}/movies/_doc/1" -d '{"director": "Burton, Tim", "genre": ["Comedy","Sci-Fi"], "year": 1996, "actor": ["Jack Nicholson","Pierce Brosnan","Sarah Jessica Parker"], "title": "Mars Attacks!"}' -H 'Content-Type: application/json'

# 批次寫入
$ curl -XPOST -u 'master_username:master_password' "${ES_DOMAIN}/_bulk" --data-binary @/tmp/bulk_movies.json -H 'Content-Type: application/json'

# 查詢資料
$ curl -XGET -u 'master_username:master_password' "${ES_DOMAIN}/movies/_search?q=mars&pretty=true"
```

public access 的 ES domain 表示可以被任何地方存取到，因此只要 master user 的帳號密碼被猜到，就可以登入 ES Domain 了，因此**請至少設定 ip address 的過濾**，限制特定 IP 才能存取。



VPC-Based Elasticsearch Domain
==============================

除了 public access 之外，若要將 ES Domain 建置在 VPC 內，那 Network Configuration 的部份就要選擇 VPC Access 了，以下針對優點 & 相關限制進行說明。

## 優點

將 ES domain 放入 VPC 中有以下幾個好處：

- VPC 內部服務可以與 ES domain 直接溝通，不用透過 NAT Gateway or Internet Gateway (同時也節省流量費用)

- VPC 本身提供了額外的安全性，保護 ES domain 不會暴露在外

## 限制

- ES domain 僅能運行在 VPC 內，或是獨立在外搭配一個 public endpoint (無法同時兩個一起，建立 ES domain 時就必須確認)

- 上面的選項(in VPC or with public endpoint)一旦選擇，是無法變更的；要變更就必須要重建 ES domain 並轉移資料

- 在 VPC 中的 ES domain 僅能選擇 `Default` tenancy，不能選擇 `dedicated` tenancy

- 在 VPC 中的 ES domain 僅能調整 subnet & security group 設定，不能變更 VPC

- VPC domain 比起 public domain 顯示更少的資料

- 要存取 VPC 內部的 Kibana，就只能透過 Proxy server、VPN、或是 transit gateway 這些方式來達成了

## 透過 SSH Tunnel 連線

這需要滿足幾個條件：

1. 有一個 proxy server (bastion node)

2. 要有一個免密碼的 SSH Keypair，並設定於 proxy server 中

3. proxy server 是可以存取 ES domain endpoint 的 (確定 IAM 存取控制要設定好，基本上用 IP-based Policies 最快)

滿足了上述條件後，就可以建立 SSH Tunnel 在 local 端與 ES domain 連線。

假設 ES Domain endpoint 為：`vpc-myes-12345.ap-northeast-1.es.amazonaws.com`，那就可以透過下面的指令建立一個 SSH Tunnel

```bash
$ ssh ubuntu@at -N -L 9200:vpc-myes-12345.ap-northeast-1.es.amazonaws.com:443
```

接著就可以透過連線 [https://localhost:9200/_plugin/kibana/](https://localhost:9200/_plugin/kibana/) 連到 Kibana 了!



References
==========

- [In-place version upgrades for Amazon Elasticsearch Service | AWS Database Blog](https://aws.amazon.com/blogs/database/in-place-version-upgrades-for-amazon-elasticsearch-service/)

- [Set alerts in Amazon Elasticsearch Service | AWS Big Data Blog](https://aws.amazon.com/blogs/big-data/setting-alerts-in-amazon-elasticsearch-service/)


- [Use NGINX to Access Kibana with Amazon Cognito Authentication](https://aws.amazon.com/premiumsupport/knowledge-center/kibana-outside-vpc-nginx-elasticsearch/)