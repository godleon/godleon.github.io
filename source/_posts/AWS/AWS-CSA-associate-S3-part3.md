---
layout: post
title:  "AWS CSA Associate 學習筆記 - S3(Simple Storage Service) Part 3"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 S3 服務，共分為四個部份，此為第三個部份"
date: 2020-05-10 10:00:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - S3
---


Transfer Acceleration
=====================

S3 transfer acceleration 是利用 AWS 佈署在全世界的 edge server 來達到上傳加速的功能，有以下幾點特性：

- 使用者上傳檔案的 endpoint 會從 region 變成 edge server，因此必須透過另一個不同的 endpoint 上傳檔案

- 檔案傳到 edge server 後，就是透過 AWS 的骨幹網路回到 s3 bucket 所在的 region 內

- 啟用後，最多需要 20 mins 的生效時間

- 啟動此功能後，AWS 會產生一個 `bucketname.s3-accelerate.amazonaws.com` 的網域名稱，透過此網域名稱，使用者就可以將檔案上傳到離自己最近的 edge server
> 為什麼一組域名就可以? AWS 會自行根據使用者查詢 DNS 的來源，回應一個最靠近使用者的 edge server，這是每個 CDN provider 有的功能

![S3 Transfer Acceleration](/blog/images/aws/S3_Transfer-Acceleration.png)


此外，AWS 也提供了一個[測速的網頁](https://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html)，使用者可以自行評估須不需要開啟 S3 Transfer Acceleration 功能。



CloudFront
==========

CloudFront 就是 AWS 透過佈建在全世界各地的 edge server 所提供的 CDN 服務，有以下幾個重點知識需要了解：

- AWS 在全世界各地建立了 edge server，確保大多數的使用者都有一個相對源頭教近的地方可以取得資料

- **Edge Location**：實際就是快取靜態資源的地方(edge server)，跟 Region & AZ 兩個是不同的概念

- **Origin**：這就是靜態資源的原始儲存位置，當使用者對 edge server 要求檔案，但 edge server 沒有時，edge server 就會回源到此處取得所需的檔案
> Edge server 會將回源取到的資料進行 cache，下一位使用者來存取時，就不需要再度回源了
![S3 CloudFront](/blog/images/aws/S3_CloudFront.png)

- **Distribution**：這其實就是 CDN 服務本身所提供的 endpoint，搭配其智慧化的 DNS 解析能力，可以讓使用者取得離自己最近的 edge server 紀行存取，因此可以視為 edge location collection；而 AWS CloudFront 目前支援兩種資源的發佈
  
  - **Web Distribution**: 通常是一般網站使用的靜態資源，像是 image, css, javascript, music ... 都屬於此類；也包含透過 HTTP 協議傳輸的視訊協定，例如：HTTP Live Streaming(HLS), Dynamic Adaptive Streaming over HTTP (DASH), Microsoft Smooth Streaming (MSS), or HTTP Dynamic Streaming (HDS) .... 等等。
  
  - **RTMP**: 用於視訊串流類的服務中(但這通常會根據服務型態不同，還需要額外考慮視訊延遲的問題)
  > CloudFront 將會在 2020 年底停止 RTMP 的支援

- Edge Locations 並非只是用來拉取資料，也會有寫入資料的應用場景(例如：**S3 Transger Acceleration**)

- 快取在 edge location 的靜態資源，會有 TTL(Time to Live) 的屬性，不會永久存在

- 為了讓使用者可以很快取得最新的資料，可以執行 CDN 清理的工作或是讓快取失效(invalidate)，但因為需要重新回源，因此會有額外費用產生

## 進階設定

- `Restrict Viewer Access(Use Signed URLs or Signed Cookies)`：開啟此功能，可以限定使用者必須要登入後取得特定域名的 cookie 才可以存取 CDN 上的快取資源


Snowball 系列服務
================

Snowball 是設計用來協助使用者移動大量資料用，不論是從本地端到 AWS，或是從 AWS 回到本地端，都可以利用此服務。

## Snowball Edge

- Snowball 是個安全且兼顧的硬體裝置，被設計用來移動大量資料，不論是從本地端到 AWS，或是 AWS 到本地端都可；簡單來說，就是被設計用來做資料遷移和邊緣運算裝置，可簡單稱為 `Snowball Edge`

- 為了加強資料移動的安全性，除了有堅固的外殼外，Snowball 還帶有加密功能 & 內建 TPM；並且還設計了資料抹除的功能，確保資料移動完畢後可以完全清除

- 與透過現有網際網路來傳輸資料相比，透過 Snowball 有便宜、安全 ... 等優點

- Snowball Edge 可以分為 `Snowball Edge Storage Optimized` & `Snowball Edge Compute Optimized` 兩類，可以讓使用者在移動大量資料時，也可以根據需求進行一定程度的運算工作(例如：EC2 & Lambda function)

- 多台 Snowball 裝置還可以組成 cluster，變成一個臨時的大型設備

## Snowmobile

{% youtube 8vQmTZTq7nw %}

- Snowmobile 本身是個 數百 PB ~ EB 等級的資料移動服務，這樣量級的資料量，需要拖車來運送了....

- Snowmobile 採用了多層安全保護，包含專門安全人員、GPS 追蹤、警示監控、24 小時全年無休的視訊監視，以及運輸期間選擇性的安全護衛車隊(帶保鏢的意思....)


Storage Gateway
===============

## 什麼是 Storage Gateway ?

AWS Storage Gateway 是一種混合雲端儲存服務，有以下兩個特性：

- 是一個連結地端(on-premise) & 雲端(AWS)的工具，可以是一個 software appliance(以 VM 的形式存在，目前支援 VMware ESXi & Microsoft Hyper-V)，也可以是一台硬體

- 提供非常簡易且安全的方式，協助使用者將資料移到 AWS 儲存服務(S3, EBS ... etc)中

其實最終的目的就是為了讓使用者可以大幅簡化資料移到 AWS 進行儲存的工作。


## Storage Gateway 如何處理不同類型的資料 ?

使用者購買 storage 進行資料的儲存，雖然同樣是存放資料，但跟資料的使用方式、用途、情境，其實還可以細分為以下三類：

- `一般檔案`：此類型檔案通常會以 **NFS** or **SMB** 的形式提供存取

- `Volume`：此類檔案則是由 block storage 來提供，通常會以 **iSCSI** 的方式提供存取，使用上就像一個硬碟一樣

- `Tape`：這類的檔案通常都是因為法令關係需要將資料封存一段時間而存在於磁帶中

因此，為了處理上述的不同的資料類型，因此 AWS 也提供了三種不同的 Storage Gateway 來處理這些資料，分別是 `File Gateway`、`Volume Gateway`、`Tape Gateway`。

### File Gateway

![Storage Gateway - File Gateway](/blog/images/aws/S3_StorageGateway-FileGateway.png)

- 會將檔案以 object 的形式存放在 S3 bucket 中

- 本地端可透過 NFS 的方式進行儲存

- 與檔案相關的 ownership, permission, timestamp 等資訊，則會存放在 S3 object user-metadata 中

- 一旦將資料移動到 S3 後，就可以使用 S3 提供的 versioning, lifecycle management, cross-region replication ... 等功能

### Volume Gateway

Volume Gateway 提供了 `iSCSI` 的方式，讓資料可透過**單一磁碟(volume)**為單位的角度來進行資料的儲存與管理；寫入 volume 的資料會以非同步的方式透過 volume snapshot 的方式進行儲存，也因為是 block device 的關係，因此每次的變更備份都是只有處理變更的 block 部份而已。

而根據 volume 使用情境的不同，Volume Gateway 還可以分成以下兩種：

#### Stored Volume

![Storage Gateway - File Gateway](/blog/images/aws/S3_StorageGateway-VolumeGateway-StoredVolume.png)

- 資料主要儲存在本地端(對本地端的應用程式相對延遲低)，非同步備份到 AWS

- 實際寫進 volume 的資料會先存在於本地端的儲存設備中

- 資料備份會以非同步的方式，並以 EBS(Elastic Block Volume) snapshot 的形式存放於 S3 中 

- 目前支援的 volume 大小範圍為 1GB ~ 16 TB

#### Cached Volume

![Storage Gateway - File Gateway](/blog/images/aws/S3_StorageGateway-VolumeGateway-CachedVolume.png)

- 資料其實是存放在 S3 中

- 但為了讓本地端使用上可以更為快速，會將常存取的資料 cache 在本地端
> 透過此方式，可以讓地端設備不需要購置大容量的儲存設備，因為資料其實大部份都是存放在 S3，本地端只存放常用的一小部份資料

- 支援的 volume 大小範圍為 1GB ~ 32TB

### Tape Gateway

- 用來解決磁帶備份 & 保存的問題

- 以 `iSCSI` device 的形式提供給使用者進行資料存放

- 需要與現有的磁帶備份軟體(例如：NetBackup, Backup Exec, Veeam ... 等)搭配使用

- 會將資料非同步的回傳至 S3，並搭配 Glacier 將成本降低


Athena vs Macie
===============

這兩個服務提供了使用者直接對儲存在 S3 的資料進行查詢 & 分析的能力。

## Athena

- Athena 是種互動是的查詢服務，以 serverless 的方式運行

- 可透過標準的 SQL 對 S3 中的資料進行查詢，不需要設置複雜的 ETL(Extract/Transform/Load) 程序

- 實際查詢行為發生時才需要付費

- 通常用來查詢 Log(例如：ELB log, S3 access log)、產生報表

## Macie

- 同樣也是針對儲存在 S3 的資料進行分析查詢(也可以分析 CloudTrail logs)

- 透過機器學習 & 自然語言處理(NLP, Natural Language Processing)，自動探索、分類和保護 AWS 中的敏感資料

- 可辨識個人識別資訊(PII, Personal Identification Information) 或智慧財產等敏感資料

- 很適合與 PCI-DSS 標準進行整合，並用來防止 ID 被盜取的問題

- 也可用來分析 CloudTrail log 來發現可疑的 API 存取行為


References
==========

## S3

- [Amazon S3 Transfer Acceleration - Amazon Simple Storage Service](https://docs.aws.amazon.com/zh_tw/AmazonS3/latest/dev/transfer-acceleration.html)

- [Amazon Simple Storage Service (S3) FAQs](https://aws.amazon.com/s3/faqs/) [(中文)](https://aws.amazon.com/tw/s3/faqs/)

## CloudFront

- [以 Amazon CloudFront 加速您的網站 - Amazon Simple Storage Service](https://docs.aws.amazon.com/zh_tw/AmazonS3/latest/dev/website-hosting-cloudfront-walkthrough.html)

- [FAQs | CDN, Zone Apex, Edge Cache | Amazon CloudFront](https://aws.amazon.com/cloudfront/faqs/?nc=sn&loc=6)

## Snowball

- [AWS Snowball | 安全邊緣運算和離線資料傳輸 | Amazon Web Services](https://aws.amazon.com/tw/snowball/)

- [AWS Snowball 常見問答集](https://aws.amazon.com/tw/snowball/faqs/)

## Snowmobile

- [AWS Snowmobile | EB 級資料傳輸 | Amazon Web Serivces](https://aws.amazon.com/tw/snowmobile/)

- [https://aws.amazon.com/tw/snowmobile/faqs/](https://aws.amazon.com/tw/snowmobile/faqs/)


## Storage Gateway

- [AWS Storage Gateway - Amazon Web Services](https://aws.amazon.com/tw/storagegateway)

- [AWS Storage Gateway – 磁碟區閘道](https://aws.amazon.com/tw/storagegateway/volume/)

## Athena & Macie

- [Amazon Athena - 無伺服器互動式查詢服務 - Amazon Web Services](https://aws.amazon.com/tw/athena/)

- [Amazon Macie | 探索、分類和保護敏感資料 | Amazon Web Services (AWS)](https://aws.amazon.com/tw/macie/)