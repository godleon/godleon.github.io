---
layout: post
title:  "AWS CSA Associate 學習筆記 - S3(Simple Storage Service) Part 1"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 S3 服務，共分為四個部份，此為第一個部份"
date: 2020-04-26 12:00:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - S3
---


What is S3?
============

首先來看看 S3(**AWS Simple Storage Service**) 在 AWS 原廠網站上的定義：

> Amazon Simple Storage Service (Amazon S3) is an object storage service that offers industry-leading scalability, data availability, security, and performance. This means customers of all sizes and industries can use it to store and protect any amount of data for a range of use cases, such as websites, mobile applications, backup and restore, archive, enterprise applications, IoT devices, and big data analytics. Amazon S3 provides easy-to-use management features so you can organize your data and configure finely-tuned access controls to meet your specific business, organizational, and compliance requirements. Amazon S3 is designed for 99.999999999% (11 9's) of durability, and stores data for millions of applications for companies all around the world.

基本上可以歸納出幾個重點：

- 要找個安全的地方存放檔案，S3 是個很好的選擇 (有 11 個 9 個可用度，自己維護 storage 要到這種可用度幾乎是不可能啦....)

- S3 是屬於 object-based storage，與 block-based 類型的 storage 是不同的

- 各種以 file-based(可將 object 視為一個 file) 的應用，都可以與 S3 進行整合


關於 S3 個科普知識
================

## General

- S3 bucket 雖然是屬於特定 region，但在 AWS console 會以 global 的方式顯示(表示 console 中會顯示所有 region 的 bucket)

- S3 是 object storage，儲存在上面就是一般的檔案

- 單一檔案大小的限制為 `0 bytes` ~ `5 TB`，整體的儲存空間無限制

- 檔案一律存在 bucket 中 (Bucket 裏面無法再放一個 bucket，但可以放 folder；但其實不是真的 folder，實際上是透過 `key-name prefix` 所模擬出來的)

- **單一帳號預設最大上限可存放 100 個 buckets，但可以通知 AWS 協助放大上限**

- S3 裡面的每個 bucket 都會有一個全球獨一無二的 DNS 名稱(ex: `https://YOUR_UNIQUE_BUCKET_NAME.s3.amazonaws.com`)；若是要指定到特定的 region，則可能會是 `https://YOUR_UNIQUE_BUCKET_NAME.eu-west-1.amazonaws.com`；但無論如何，使用者只要記住 bucket name 即可
> 因此可以做為 static website hosting 之用 (Everyone-read 的權限必須有設定好)

-  若希望 S3 resource 可以在不同的 domain 之間互相分享，要把 `CORS` 設定開啟，允許其他 domain 來的 request

- 成功上傳檔案到 S3 後，會收到 `HTTP 200` 的回應


## AWS S3 給使用者的保證

- S3 平台本身有 99.99% 可用度

- AWS 平台保證有 99.9% 的可用度 (但可靠度要看最小的，所以還是會以 99.9% 為主)

- 對於儲存在 S3 的檔案，AWS 保證 11 個 9 的可靠度(資料遺失的可能性極低)
> 為避免人為不小心刪除的狀況發生，最好的方法還是建議把 versioning, cross-regsion replication, MFA 刪除等機制啟用


## 其他特性

- 支援多種 storage tier
> 可與檔案生命週期管理政策進行搭配

- 檔案生命週期管理
> 可以設定前 30 天在正常的 **standard** tier, 接著 30 天移到另外一個 IA(Infrequently Accessed) tier, 90 天後進行 archive(移到 Glacier)

- 版本控管

- 檔案加密 

- MFA Delete
> 重要檔案可以加上 MFA(多因素認證) 的流程進行確認，才能實際刪除檔案 (**只有 root account 可以開啟此功能**，且只能透過 CLI 開啟)

- 可透過 **Access Control List** & **Bucket Policy** 來提升存取檔案的安全性
> 剛建立好的 bucket or 上傳的 object 所預設的權限僅限於自己可以存取(private & inaccessible)，完全沒有預設對外開放的規則


## Object 中包含什麼資訊?

S3 is object based. 每個 object 都包含以下資訊：

- **Key**: object name (檔案會依照字母順序排序，新增時要考量這個問題)

- **Value**: 基本上就是此檔案的資料本身(一堆 byte 的組合)

- **Version ID**: 作為版本控管之用

- **Metadata**: 額外用來記錄 object 相關資訊的資料(ex: 上傳檔案的時間、最後變更的時間...etc)
> 使用者也可以自訂客製化的 metadata，藉此來為 object 標註不同的屬性值

- **Subresources**
  - Access Control Lists (用來做細部的存取控管)
  - Torrent (S3 支援 bittorrent protocol)


## 不是真的 folder 的 S3 folder

- S3 本身的設計是種 flat 結構，並不是以 hierarchy 的方式組織 & 存放資料

- 為了讓使用者方便使用S3，S3 支援了 `folder` 的概念，但這其實只是一種 group 的概念

- S3 是透過為 object 加上 key-name prefix 的方式達成的(此部份使用者不可見)



在 S3 上的檔案一致性如何呈現?
=========================

- 若是新增檔案(透過 `PUTS` 的方式新增)，當檔案寫入後馬上就可以讀取到

- 若是更新(`overwrite PUTS`) or 刪除檔案(`DELETE`)，則是 Eventual Consistency，這樣的變更需要花點時間才會完全套用(propagate)到所有的硬體設施中
> 在 2020/12 時，推出了 [strong consistency](https://aws.amazon.com/s3/consistency/)，現在不論是更新(`overwrite PUTS`) or 刪除檔案(`DELETE`)，馬上讀取都可以得到最新的結果了


S3 Storage Class
================

S3 根據使用者存取的頻率與需求，提供不同的 storage class 供使用者選擇：
> **調整單位可以細到 object level，而非僅僅是 bucket level**

## Standard

- 存取速度最快

- 提供 99.99% 的可用率

- 提供跨 AZ 的 11 個 9 的可靠度

- 會自動將資料儲存在多個 AZ 中，會同時有三個備份，因此在兩個實體設施損毀的情況下，還是可以取得資料

- **沒有低消問題，資料儲存多久就算多久的費用**

### 參考資料

> [物件儲存類別 – Amazon S3 Storage Class(Standard)](https://aws.amazon.com/tw/s3/storage-classes/#General_purpose)

## IA(Infrequently Accessed)

- 存取頻率相較 standard 較低，但需要的時候還是可以馬上取得

- 提供 99.9% 的可用率

- 提供跨 AZ 的 11 個 9 的可靠度

- 會自動將資料儲存在多個 AZ 中，會同時有三個備份，因此在兩個實體設施損毀的情況下，還是可以取得資料

- 費用比 standard 便宜

- 存取速度比 standard 慢

- **低消為 30 天的收費 (短時間的儲存不划算)**

- 常用在 disaster recovery、backup ... 等用途

### 參考資料

> [物件儲存類別 – Amazon S3 Storage Class(IA)](https://aws.amazon.com/tw/s3/storage-classes/#Infrequent_access)

## One Zone - IA

- 2008 年所新增的 storage class

- 沒有提供 multiple AZ 的資料保護

- 在同一個 AZ 中有 11 個 9 的可靠度；不過整個 AZ 毀了資料也就會沒了

- 提供 99.5% 的可用率

- 因為資料在單一 AZ 的關係，因此在存取效能上可以提供更低的延遲 & 更高的 throughput

- 價格比 IA 更為便宜(約 20%)

- **低消為 30 天的收費 (短時間的儲存不划算)**

- 常用在 second backup，或是存放可重建的資料 ... 等用途

### 參考資料

- [宣佈 S3 One Zone-Infrequent Access，一種新的 Amazon S3 儲存類別](https://aws.amazon.com/tw/about-aws/whats-new/2018/04/announcing-s3-one-zone-infrequent-access-a-new-amazon-s3-storage-class/)

- [物件儲存類別 – Amazon S3 Storage Class(One Zone - IA)](https://aws.amazon.com/tw/s3/storage-classes/#__)

## Intelligent Tiering

- 存取效能等同於 S3 standard

- AWS 會自動協助使用者進行優化，將不常存取的檔案移動到較為便宜的 access tier，常用的則會留在 standard tier

- 透過人工智慧分析，自動將使用成本最佳化，且不會造成效能的影響，也可以減輕管理上的負擔

> 但最多幫使用者將資料移動到 IA tier 的費率，無法更低了(例如：**One Zone-IA** or **Glacier**)

- **低消為 30 天的收費 (短時間的儲存不划算)**

### 參考資料

- [物件儲存類別 – Amazon S3 Storage Class(Intelligent Tiering)](https://aws.amazon.com/tw/s3/storage-classes/#Unknown_or_changing_access)

## Glacier

- 設計用來封存資料用(例如：稽核用的 log)，但**不應該拿來備份用的**

- 存取時間會需要數分鐘到數小時不等，因此分成 `Expedited`(1\~5 mins)、`Standard`(3\~5 hours)、`Bulk`(5\~12 hours) 三個等級，單位收費也不同
> **若是要確保可以在一定時間內取得資料，並有一定資料下載的 throughput，那就要購買 `Provisioned capacity`**

- 提供跨 AZ 的 11 個 9 個可靠度

- 一個 AZ 全毀的情況下還是可以取得資料 

- 儲存費用很低

- **低消為 90 天的收費 (短時間的儲存不划算)**

- **會自動將存進來的資料進行 AES 256-bits 的加密**

### 參考資料

- [物件儲存類別 – Amazon S3 Storage Class(Glacier)](https://aws.amazon.com/tw/s3/storage-classes/#Archive)

## Glacier Deep Archive

- 存取時間會需要 12 小時以上

- 提供跨 AZ 的 11 個 9 個可靠度

- 如果要保存好幾年的資料，可以考慮使用這個 tier

- 費用最便宜的選項


### 參考資料

- [物件儲存類別 – Amazon S3 Storage Class(Glacier Deep Archive)](https://aws.amazon.com/tw/s3/storage-classes/#____)

![S3 Storage Class comparison](/blog/images/aws/S3_Storage-Classes-comparision.png)


S3 的收費標準
===========

介紹了 S3 的眾多特性，總是要了解使用 S3 服務時，在哪些時候 AWS 會進行收費，目前收費的情況會發生在以下幾個部份：

- 資料儲存的費用 (儲存越多當然就收費越多，單位費用會根據使用的 access tier 而不同，例如：standard 會比 IA 貴)

- 對 S3 發出 request (對 S3 進行 HTTP request 是會計費的)

- 儲存管理相關的費用(例如：analysis, tagging, inventory check)

- 資料傳輸的費用
> 資料存入 S3 免費，但資料傳出則要付費；往其他地方傳則要付費，即使是不同 region 之間互傳也是需要付費

- 啟用加速傳輸([Transfer Acceleration](https://docs.aws.amazon.com/zh_tw/AmazonS3/latest/dev/transfer-acceleration.html))
> 此功能是利用 AWS 分佈在全球的 edge server，讓使用者透過 edge server 傳輸資料，並透過 AWS 佈建在全球的骨幹網路，以最佳化的傳輸路徑快速的存入 S3 storage

- 跨 Region 的備份
> 多一份的備份，資料更安全，自然也會需要額外的費用


總結
====

- S3 是 object-based storage，用來上傳一般檔案用

- 單一檔案的大小限制在 0 bytes ~ 5 TB

- 只要荷包夠深，沒有限制可使用的儲存空間

- 檔案儲存在 Bucket 中

- S3 的名稱必須是全球唯一的

- 因為是 object-based storage，因此不適合拿來安裝 OS 或是資料庫服務用

- 檔案上傳成功會回傳 HTTP 200

- 可以透過開啟 MFA Delete，可以多一層資料刪除前的保護

- S3 object 包含了 key, value(檔案內容本身), version ID, Metadata, Subresource(ACL, Torrent) ... 等資訊

- 檔案上傳後就可以馬上讀取到

- 若是更新 or 刪除檔案，則需要一段時間才會取得最新結果

- S3 有眾多的 storage class 可以選用，使用者可以根據實際需求進行不同 storage class 的搭配以降低費用；若是懶的管理可以考慮選用 `Intelligent Tiering`，AWS 會根據實際使用狀況，最佳化使用成本 (但要長期備份的資料還是要自己移到 `Glacier` or `Glacier Deep Archive`)

- 用來控制 Bucket 存取權限有兩個手段，分別是 `bucket ACL`(控制存取 bucket 本身的權限) & `bucket policies`(控制 bucket 與其他 AWS service 互相存取的權限)

- S3 FAQ 很重要，考前必讀!


考試重點
=======

## S3 服務特性

- S3 單一檔案容量上限為 5TB；但單一 HTTP PUT 操作的上限只有 5GB


## 利用 Prefix 來提高 S3 的並行處理能力

- 每個 S3 bucket 可以接收每秒 3,500 個 PUT/COPY/POST/DELETE request，或是 5,500 的 GET/HEAD request (這是以一個 prefix 為前提下所提供的效能)

- 若是希望使用單一 bucket 提供服務，也同時需要 S3 可以處理更大量的 request，則可以根據業務邏輯設計不同分類，以多個 prefix 的方式提昇 S3 bucket 可接受 request 的數量 

- 一個 bucket 中可以接受的 prefix 數量是沒有限制的

- 舉例來說，若是 S3 object path 為 `S3://your_bucket_name/folder1/sub_folder_1/f1`，那 `/folder1/sub_folder_1` 就是 prefix

## 安全性

- 若覺得 CloudTrail 所提供 S3 的 log 資訊不足，可以透過開啟 S3 的 `Server access logging` 的功能，可以提供更細緻到 bucket level 的 API 呼叫資訊



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [物件儲存類別(Storage Class) – Amazon S3](https://aws.amazon.com/tw/s3/storage-classes/)

- [Amazon Simple Storage Service (S3) – 雲端儲存 – AWS](https://aws.amazon.com/tw/s3/faqs/)


