---
layout: post
title:  "AWS 學習筆記 - S3(Simple Storage Service)"
description: "This article is a memo recorded when learning AWS S3(Simple Storage Service)"
date: 2017-05-15 04:30:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - S3
---


What is S3?
============

Amazon Simple Storage Service (Amazon S3) is object storage with a simple web service interface to store and retrieve any amount of data from anywhere on the web. It is designed to deliver 99.999999999% durability, and scale past trillions of objects worldwide.

Customers use S3 as primary storage for cloud-native applications; as a bulk repository, or "[data lake](https://aws.amazon.com/big-data/data-lake-on-aws/download/)," for analytics; as a target for [backup & recovery](https://aws.amazon.com/backup-recovery/getting-started/) and disaster recovery; and with [serverless computing](https://aws.amazon.com/lambda/details/).

It's simple to move large volumes of data into or out of Amazon S3 with Amazon's [cloud data migration](https://aws.amazon.com/cloud-data-migration/) options. Once data is stored in S3, it can be automatically tiered into lower cost, longer-term cloud storage classes like S3 Standard - Infrequent Access and [Amazon Glacier](https://aws.amazon.com/glacier/details/) for archiving.


S3 - The Basics
===============

- 單一檔案大小的限制為 `0 bytes` ~ `5 TB`

- 檔案一律存在 bucket 中 (Bucket 裏面無法再放一個 bucket，但可以放 folder)

- 單一帳號預設最大上限可存放 100 個 buckets，但可以通知 AWS 協助放大上限

- S3 裡面的每個 bucket 都會有一個全球獨一無二的 DNS 名稱(ex: https://s3-eu-west-1.amazonaws.com/YOUR_UNIQUE_BUCKET_NAME)
> 因此可以做為 static website hosting 之用

- 成功上傳檔案到 S3 後，會收到 HTTP 200 的回應

- Built for 99.99% availability for S3 platform

- Amazon 實際保證 99.9% availability

- Amazon 保證 11 個 9 的 durability for S3 information (資料遺失的可能性)
> 為避免人為不小心刪除的狀況發生，最好的方法還是建議把 versioning, cross-regsion replication, MFA 刪除等機制啟用

- Tiered Storage available

- Lifecycle Management
> 可以設定前 30 天在正常的 **standard** tier, 接著 30 天移到另外一個 IA(Infrequently Accessed) tier, 90 天後進行 archive

- Versioning

- Encryption

- 可透過 Access Control Lists & Bucket Policies 來提升安全性
> 剛建立好的 bucket or 上傳的 object 所預設的權限僅限於自己可以存取(private & inaccessible)，完全沒有預設對外開放的規則


Data Consistency Model for S3
=============================

- Read after consistency for PUTS of new Object (新增檔案後馬上就可以讀取)

- Eventually Consistency for overwrite PUTS and DELETES(can take some time to propagate)
> 若是針對已經存在的檔案進行修改 or 刪除，這樣的變更需要花點時間才會完全套用到所有的硬體設施中


S3 is a simple key, value store
================================

S3 is object based. 每個 object 都包含以下資訊：

- **Key**: object name (檔案會依照字母順序排序，新增時要考量這個問題，建議在每個檔案名稱前 random 一個字串作為開始)

- **Value**: 基本上就是此檔案的資料本身

- **Version ID**: 作為版本控管之用

- **Metadata**: 額外用來記錄 object 相關資訊的資料(ex: 上傳檔案的時間、最後變更的時間...etc)
> 使用者也可以自訂客製化的 metadata，藉此來為 object 標註不同的屬性值

- **Subresources**
  - Access Control Lists (用來做細部的存取控管)
  - Torrent (S3 支援 bittorrent protocol)



S3 vs Glacier
=============

|  | Standard | Standard - IA | Glacier |
|--|:--------:|:-------------:|:-------:|
| Designed for Durability | 11 x 9's | 11 x 9's | 11 x 9's |
| Designed for Availability | 99.99% | 99.99% | N/A |
| Availability SLA | 99.9% | 99% | N/A |
| Minumum Object Size | N/A | 128KB | N/A |
| Minumum Storage Duration | N/A | 30 days | 90 days |
| Retrieval Fee | N/A | per GB retrieved | per GB retrieved |
| First Byte Latency | ms | ms | select minutes or hours |
| Storage Class | object level | object level | object level |
| Lifecycle Transitions | yes | yes | yes |

| S3 | Glacier |
|:--------:|:-------:|
| up to 5TB object | up to 40TB archive |
| user-definable key | system-generated archive ID |
| encrypt is optional | automatically encrypted |


S3 收費標準
==========

- Storage Pricing

- Request Pricing

- Storage Management Pricing
> 例如：analysis, tagging, inventory check

- Data Transfer Pricing
> 資料存入 S3 免費，往其他地方傳則要付費，即使是 region 之間互傳

- Transfer Acceleration
> Amazon S3 Transfer Acceleration enables fast, easy, and secure transfers of files over long distances between your client and an S3 bucket. Transfer Acceleration takes advantage of Amazon CloudFront’s globally distributed edge locations. As the data arrives at an edge location, data is routed to Amazon S3 over an optimized network path.


Advanced Features
=================

## Prefix & Delimiter

以下面的 object 名稱為例：

> logs/2016/January/server42.log
> logs/2016/February/server42.log
> logs/2016/March/server42.log

- S3 可透過 prefix & delimiter 作到類似檔案階層的功能(hierachy & folder)，但事實那些都只是透過 object 的檔名虛構出來的，並沒有實際階層功能

- REST API, SDK, CLI, Management Console 都支援使用 prefix & delimiter

- 與 IAM or S3 Bucket Plicies 搭配，可以在單一個 bucket 中達成像是 department subdirectories, user home directories ... 等效果


## Storage Tiers/Classes

|  | Standard | Standard (Infrequent Access) | Reduced Redundancy Storage |
|--|:--------:|:----------------------------:|:--------------------------:|
| Durability | 99.999999999% | 99.999999999% | 99.99% |
| Availability | 99.99% | 99.99% | 99.99% |
| Concurrect facility fault tolerance | 2 | 2 | 1 |
| SSL support | Yes | Yes | Yes |
| First byte latency | Miliseconds | Miliseconds | Miliseconds |
| Lifecycle Management Policies | Yes | Yes | Yes |
| 附註 || object 大小限制為最小 128KB, 最少要存放 30 天, 存取資料要額外收費 ||

### 1. Standard

99.99 availability, 99.999999999% durability, stored redundantly across multiple devices in multiple facilities and is designed to sustain the loss of 2 facilities concurrently.

### 2. IA(Infrequently Accessed)

適合不常存取的資料，比 standard 便宜，要存取時可以馬上取得，但存取需要額外付費

### 3. Reduced Redundancy Storage (RRS)

Design to provide 99.99% durability and 99.99 availability of objects over a given year.

> durability 從 **11 x 9's** 變成 **4 x 9's**，適合存像是圖片的 thumb nails 之類的資料

### 4. Glacier

- very cheap, but used for archival only. 

- 存取前需先執行 restore 命令，並需要等待 3~5 個小時的資料準備時間

- Glacier 上的資料不會因為 restore 命令而刪除，除非明確執行刪除指令

- 資料還原時會放到 S3 **RRS(Reduced Redundancy Storage)** class

- AWS 提供每個月免費取得 5% Glacier 資料的額度 (每日為單位計算)
> 可透過設定 **retrieval policy** or **設定 max GB-per-hour limit** 來確保存取資料會在免費的額度下進行，來降低甚至避免還原費用的發生

- 雖然是 S3 Storage Class 的一個選項，但其實 Glacier 是個獨立服務且有獨立的 API，並提供一些 S3 沒有的功能

- 可用來作為取代傳統磁帶作為長期備份的選項 (在某些行業必須有保留資料 N 年的規定)

- 非常可靠，也是有 11 個 9 的 durability

#### Archive

- 在 Glacier 上儲存的備份單位稱為 **archive**（等同 S3 上 object 的概念)

- 每個 archive 最大可以到 **40TB**

- 使用者可以擁有無限數量的 archive

- 每個 archive 都會有一個 unique archive ID (無法自己取名字)

- 所有的 archive 都會被自動加密 & 無法被修改

#### Vault

- 在 Glacier 中存放 archive 的容器稱為 **vault** (等同 S3 上 bucket 的概念)

- 可透過設定 IAM policy or vault access policy 來限制存取

#### Vaults	Locks

- 使用者可透過 Vaults	Locks 針對 Glacier 設定強制管理

- 可藉由設定 Write Once Read Many(WORM) 在 vault lock policy 中來套用到未來所有存放到 valut 的 archive

- 一旦設定了 vault lock policy，就無法再度變更規則


## Lifecycle Management

- 等同於傳統 IT storage infra 中的 **automated storage tiering** 功能，是一種 data 從 `hot`(frequently	accessed) -> `warm`(less	frequently	accessed) -> `cold`(long-term	backup	or	archive) 的概念

- Can be used in conjuction with versioning.

- Can be applied to current versions and previous versions

- Following actions can now be done:
  - Transition to the **Standard - Infrequent Access** storage class (128Kb and 30 days after the creation date)
  - Archive to the **Glacier** storage class (30 days after IA, if relevent)
  - Permanently Delete (若有移到 Glacier 要把刪除的日期設為 90 天以上，因為 Glacier 費用低消是 90 days)

## Encryption

### 1. In Transit

透過 AWS SSL API endpoints 存取 S3 (**HTTPS**)，存取的流量都會被加密

### 2. At Rest

#### (1) Server Side Encryption

- S3 Managed Keys (`SSE-S3`)
  > AWS 會將資料用 master key(AES-256) 進行加密，且 key 會定期更換，全由 AWS 託管

- AWS Key Management Service, Managed Keys (`SSE-KMS`)
  > 透過 KMS 指定 master key 來進行加密，**需要額外支付 KMS 的費用**
  > 但可以控管 S3 的存取，也可以確認存取 S3 的人是擁有 key 的使用者(可以搭配 AWS 的追蹤功能之道誰在何時存取了 S3 並解密，甚至可以檢視哪個沒有權限的使用者嘗試解密資料時發生的錯誤)
  
- Server Side Encryption with Customer Provided Keys (`SSE-C`)

#### (2) Client Side Encryption (將資料加密後傳到 S3)

AWS 可從兩個管道取得 data encryption key:

- AWS KMS-managed customer master key

- client-side master key


## Versioning

- object 的所有版本資訊都會被保留下來，**包含所有的寫入資訊甚至是刪除的資訊**都會被完整保留 (**同樣也要支付多倍的費用來儲存不同的版本**)

- Great backup tool

- Once enabled, Versioning cannot be disabled, only suspended.

- Integrates with Lifecycle rules

- Versioning's MFA Delete capability, which uses multi-factor authentication, can be used to provide an additional layer of security. (**只能由 root 帳號啟用此功能**)
> 刪除 object 前必須提供 token or security code 來完成

- Cross Region Replication, requires versioning enabled on the source bucket


## Pre-Signed URLs

object owner 可以透過 **pre-signed URL** 的機制，提供給其他人**暫時**存取 object 的權限，而有效期限則是由 owner 自行指定；也可以透過來保護公開的網頁資料，避免未授權的惡意行為發生。

## Multipart Upload

當使用者有大檔案要上傳時，可開啟 multipart upload 的功能，檔案會被切成多個部份同時上傳，到 S3 上後會自動重組而成原本的檔案。

> 建議超過 100MB 的檔案使用 multipart upload；此外，超過 5GB 的檔案一定要用 multipart upload 才可以上傳

> 可以針對未完成上傳的檔案設定 lifecycle policy，如此可以減少費用的支出，讓超過期限未完成上傳的檔案自動失效

## Range	GETs

可下載指定 object 的部份內容，透過指定 byte 的範圍來取得 object 的部份資料；而這功能必須透過 SDK 搭配 Range HTTP header 才能實現。

> 若是網路品質很差，或是想要從很大的 Glacier 備份中取得部份資料時可能會用到

## Cross-Region Replication

cross-region replication 允許使用者將指定 bucket 的 object 非同步的複製到其他的 region。

- 啟用 cross-region replication 的功能前，必須先啟動 bucket 的 versioning 功能

- 任何與 object 相關的 metadata or ACL 設定更動時，都會觸發 replication 的發生

- 必須設定正確的 IAM policy 讓 S3 本身可以進行 replication 的工作

- 若是已經存在的 bucket 開啟 cross-region replication 的功能，原有的資料不會被複製，必須自行複製 or 透過額外的命令來進行資料搬移

> 此功能常被用來將 object 放在靠使用者較近的地方來減少延遲；或是滿足不同地理區域備份的需求

## Logging

- 針對 bucket 存取的 logging 功能預設關閉，可透過 S3 server access logs 功能開啟

- log 可儲存在同一個 bucket 內，也可以存在另外一個 bucket 中；但重點是設定 prefix(例如：`logs/` or `bucket_name/logs/`) 讓後續尋找 log 方便是很重要的

- log 資訊包含：
  - requestor account & IP
  - request time
  - bucket name
  - action (GET, PUT, LIST ... etc)
  - response status & error code

## Event Notification

event notification 可以用來在 bucket 狀態有變更(例如：上傳 object)時驅動某些事件的發生，使用者可以透過此特性來加入到 workflow 的設計，發送警告，或是執行特定工作...等等。

- 設定於 bucket level

- 可在 object 被建立(PUT, POST, COPY or multipart upload), 被刪除(DELETE)，或是偵測到有 RRS(Reduced Redundancy Storage) object 遺失的時候發送通知

- 可透過 Amazon SNS(Simple Notification Service) or SQS(Simple Queue Service) 發送通知，也可以直接呼叫 AWS Lambda function


S3 Management Console
=====================

## Permissions

在權限管理的部分，有分成 `Objects` & `Object permissions` 兩種：

1. **Objects**: 對於 object 本身內容的存取權限
> Grant permissions to the user to list, create, overwrite, or delete objects in the bucket.

2. **Object permissions**: 對於 object 的 ACL 的存取權限，並非 object 本身
> Grant permissions to the user to read or write to an access control list (ACL) for the bucket

即使 bucket 的權限被設定為 public readable，也不代表後續上傳到此 bucket 的 object 也是 public readable，權限是分開管理的

## Versioning

- 當此功能開啟後，之後無法移除，只能關閉

- 上傳相同名稱的檔案多次，S3 會在介面上看到一個檔案搭配多個 version 可選 (透過檢視 metadata 可以發現其實根本就是不同的 object)
> 而使用容量的計算當然就是所有版本 object 的 size 總和 (**計算成本的時候要特別注意這件事情**)

- 若移除 object 的 version，永遠就會消失無法還原

- 刪除 object 之後，會在系統上留下一個 `Delete Marker`(需要在 Versions 的地方選 **Show** 藉以顯示所有歷程記錄)，刪除 Delete Marker 之後就可以回復原本被刪除的檔案 
> 目前還原功能是在舊版的版面上找到的，新版的目前沒有看到類似的功能頁面


S3 Transfer Acceleration
========================

## What is S3 Transfer Acceleration?

S3 Transfer Acceleration utilise the CloudFront Edge Network to accelerate your uploads to S3. Instead of uploading directly to your S3 bucket, you can use a distinct URL to upload directly to an edge location which will then transfer that file to S3. You will get a distinct URL to upload to:

> your-bucket-name.s3-accelerate.amazonaws.com

若要上傳大量的資料到距離本地端很遠的 region，使用 S3 Transfer Acceleration 會很有幫助


Create a Static Website using S3
================================

## 流程

1. 建立 bucket

2. 啟用 **Static website hosting**

> 這裡會取得一個 **http://your-bucket-name.s3-website.region-alias.amazonaws.com** 的 domain name

3. 可以額外設定 index/error pages.

4. 依然要提供 object read permission 才可以


Exam Tips
=========

## S3 101

- S3 is Object based.

- Files can be from 0 Bytes ~ 5 TB.

- There is unlimited storage.

- Files are stored in Buckets.

- S3 is a universal namespace, that is, names must be unique globally.
> 網址的格式為: https://[RegionName].amazonaws.com/[YourBucketName]

- Read after Write consistency for PUTS of new Object

- Eventual Consistency for overwrite PUTS and DELETES (can take some time to propagate)

- S3 Storage Classes/Tiers
  - S3 (durable, immediately available, frequently accessed)
  - S3 - IA (durable, immediately available, infrequently accessed)
  - S3 - Reduced Redundancy Storage (data that is easily reproducible, such as thumb nails etc)
  - Glacier - Archived data, where you can wait 3~5 hours before accessing

- Remember the core fundamentals of an S3 object
  - Key (name)
  > 這是一個 1024 bytes 的 UTF-8 字元所組成，在同一個 bucket 內不會重複(但不同的 bucket 可能會有同樣的 key)
  - Value (data)
  - version ID
  - metadata
  - subresource (ACL, torrent)

- Object based storage only (for files).

- **Not suitable to install an operating system**.

- Successfuly upload will generate a HTTP 200 status code.

## Versioning

- object 的所有版本資訊都會被保留下來，**包含所有的寫入資訊甚至是刪除的資訊**都會被完整保留 (**同樣也要支付多倍的費用來儲存不同的版本**)

- Great backup tool

- Once enabled, Versioning cannot be disabled, only suspended.

- Integrates with Lifecycle rules

- Versioning's MFA Delete capability, which uses multi-factor authentication, can be used to provide an additional layer of security. (**只能由 root 帳號啟用此功能**)
> 刪除 object 前必須提供 token or security code 來完成

- Cross Region Replication, requires versioning enabled on the source bucket


## Cross Region Replication

- 要啟用 cross region replication 的功能，source & destination bucket 都必須要啟用 versioning 才可以

- 設定 replication 前已經存在的 object 不會自動被同步，只有後續上傳的 object 會被同步
> 如果上傳新版的 object，原來所有版本的記錄都會一併被同步

- 任何與 object 相關的 metadata or ACL 被變更時，都會觸發 replication 的工作執行

- 必須設定 IAM policy，用以提供 S3 合適的權限進行 replication 工作

- 單一 bucket 無法同步到多個 region，但可透過 `region1_bucket -> region2_bucket -> region3_bucket` 的方式達成相同效果 (daisy chain 似乎也不行)

- 刪除 object 後，原本同步 region 上的 object 也會被刪除
  - 所從舊版的 console 刪除 delete maker 後可以恢復檔案，但刪除 delete marker 這動作不會被同步，因此同步的 destination bucket 上的檔案不會回復
  - 若使用 `region1_bucket -> region2_bucket -> region3_bucket` 同步到多個 region，delete marker 只會被同步到 region2，region3 只會得到檔案被刪除的結果，看不到 delete marker

- Deleting individual versions or delete markers will not be replicated
> 在 source bucket 中刪除 delete marker or 特定版本(在舊版的 console)的結果，不會被同步到 destination bucket


## Lifecycle Management

- 等同於傳統 IT storage infra 中的 **automated storage tiering** 功能，是一種 data 從 `hot`(frequently	accessed) -> `warm`(less	frequently	accessed) -> `cold`(long-term	backup	or	archive) 的概念

- Can be used in conjuction with versioning.

- Can be applied to current versions and previous versions

- Following actions can now be done:
  - Transition to the **Standard - Infrequent Access** storage class (128Kb and 30 days after the creation date)
  - Archive to the **Glacier** storage class (30 days after IA, if relevent)
  - Permanently Delete (若有移到 Glacier 要把刪除的日期設為 90 天以上，因為 Glacier 費用低消是 90 days)


## CloudFront

- Edge Location: This is the location where content will be cached. This is separate to an AWS Region/AZ.

- Origin: This is the origin of all the files that the CDN will distribute. This can be either an S3 bucket, an EC2 instance, an Elastic Load Balancer or Route53.

- Distribution: This is the name given the CDN which consists of a collection of Edge Locations.
  - **Web Distribution**: Typically used for websites.
  - **RTMP**: Used for media streaming. (ex: Adobe Flash)

- Edge Locations are **not just READ only**, you can write to them too. (ie. put an object on to them).

- Objects are cached for the life of the TTL (Time to Live)

- You can clear cached objects, but you will be charged. (不需要等到 TTL 結束)


## Security

- By default, all newly created buckets are **PRIVATE**

- You can setup access control to your buckets using:
  - Bucket Policies
  - Access Control Lists (可調整個別 object 的權限)

- S3 buckets can be configured to create access logs which log all requests made to the S3 buckets. This can be done to another bucket. (也可以將 log 放到另一個 AWS 帳號 S3 bucket 中，需要進行跨帳號連結的設定) 


## Encryption

### 1. In Transit (SSL/TLS)

### 2. At Rest

- Server Side Encryption
  - S3 Managed Keys (`SSE-S3`)
  - AWS Key Management Service, Managed Keys (`SSE-KMS`)
  - Server Side Encryption with Customer Provided Keys (`SSE-C`)

- Client Side Encryption


## Storage Gateway

- **File Gateway**: For flat files, stored directly on S3

- **Volume Gateway**
    - **Stored Volumes** - Entire Dataset is stored on site and is asynchronously backed up to S3.
    - **Cached Volumes** - Entire Dataset is stored on S3 and the most frequently accessed data is cached on site.

- **Gateway Virtual Tape Library (VTL)**: Used for backup and uses popular backup applications like NetBackup, Backup Exec, Veam etc


## Snowball

- Snowball

- Snowball Edge

- Snowmobile

- Understand what Snowball is

- Understand what Import Export is

- Snowball can
    - Import to S3
    - Export from S3


## S3 Transfer Acceleration

You can speed up transfers to S3 using S3 transfer acceleration. This cost extra, and has the greatest impact on people who are in far away location.


## S3 Static Websites

- You can use S3 to hsot static websites

- Serverless

- Very cheap, scales automatically

- **STATIC** only, canonot host dynamic sites


## misc

- Write to S3 - HTTP 200 code for a successful write.

- You can load files to S3 much faster by enabling multipart upload

- Read the S3 FAQ before taking the exam. It comes up A LOT!


##　Best Practices, Patterns,	and	Performance

- S3 & Glacier 非常適合在 hybrid cloud 的環境下作為異地備份的工具

- 另一個常見的應用是將 S3 作為大量 blob 檔案的儲存位置，而把這些 blob 的 index 存於其他的服務(例如：Amazon DynamoDB or Amazon RDS)，透過此方式可以針對 key name 提高搜尋的速度並支援複雜的查詢

- S3 會自動 scale 來支援 high request rate，也會自動的根據需求針對 bucket 進行 repartition 的動作

- 若有每秒超過 100 個 reuqest 的需求，可參考 developer guide 中的資訊，讓 object key 可以隨機的分佈 (可透過像是使用 hash 值作為 key name prefix 的方式來達成)


應考前建議
=========

詳細閱讀下列資料：

- [Amazon Simple Storage Service (S3) FAQs](https://aws.amazon.com/s3/faqs/) [(中文)](https://aws.amazon.com/tw/s3/faqs/)