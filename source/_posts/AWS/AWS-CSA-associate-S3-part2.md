---
layout: post
title:  "AWS CSA Associate 學習筆記 - S3(Simple Storage Service) Part 2"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 S3 服務，共分為四個部份，此為第二個部份"
date: 2020-04-29 05:50:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - SAA
  - S3
---


安全與加密
========

## Security

- 預設所有的 Bucket 建立時，都是 **PRIVATE** 無法被公開存取的

- 要調整 Bucket 存取的安全性有兩種方式，分別是：
  - **Bucket Policies** (用來控制 Bucket 與其他 AWS 互相存取的權限)
  - **Access Control Lists** (可用來調整個別 object 的權限)

- 可以開啟將所有對 S3 Bucket 存取的行為全部 log 下來的功能，而這些存取的 log 可以存到另一個 S3 Bucket(可以是同一個帳號 or 不同帳號)  

## Encryption

AWS S3 可以為每個 object(最細可以到 object 為單位) 進行資料的加密，這對於非常注重資料安全性的使用者是個很有用的功能；一旦加密功能開啟後，資料被加密，即使有人進入了 AWS 資料中心將帶有資料的硬碟取走，也沒辦法取得正確的檔案內容。

加密的方式有兩種：

### 在傳輸中進行加密

這也就是標準的 HTTPS，透過 SSL/TLS 來完成，基本上 S3 本身就是透過 HTTPS 存取，這個部份已經是早就存在的加密方式了。

### 在終端進行加密

終端可分為兩個部份，分別是 `server side` & `client side`：

- **Server Side Encryption(SSE)**：加密金鑰會由 AWS 管理，而加密金鑰可透過以下幾種方式取得：
  - 由 S3 管理的 key (`SSE-S3`)
  - 透過 AWS Key Management Service 來管理金鑰 (`SSE-KMS`)
  - Server Side Encryption with Customer Provided Keys (`SSE-C`)

- **Client Side Encryption**：則是由使用者自行加密後，再將加密後的資料傳到 S3 上



版本控制(Version Control)
========================

版本控制(version control)同樣也是 S3 增加資料安全性的一種方式，而 S3 版本控制有以下幾個特性需要了解：

- object 的所有版本都被保留，**包含所有的寫入資訊甚至是刪除的資訊**都會被完整保留 (**同樣也要支付多倍的費用來儲存不同的版本**)

- 可以作為一個強大的備份手段使用

- 一旦版本控制啟用，就無法關閉，僅能暫停而已

- 可以與其他資料儲存服務(例如：Glacier)搭配，並設定相關生命週期規則來進行進階的管理

- 可將刪除版本的動作綁定 MFA，強制使用者執行刪除動作前進行認證，進一步提高資料的安全性

## Versioning

- object 的所有版本資訊都會被保留下來，**包含所有的寫入資訊甚至是刪除的資訊**都會被完整保留 (**同樣也要支付多倍的費用來儲存不同的版本**)

- Great backup tool

- Once enabled, Versioning cannot be disabled, only suspended.

- Integrates with Lifecycle rules

- Versioning's MFA Delete capability, which uses multi-factor authentication, can be used to provide an additional layer of security. (**只能由 root 帳號啟用此功能**)
> 刪除 object 前必須提供 token or security code 來完成

- Cross Region Replication, requires versioning enabled on the source bucket


生命週期管理(Lifecycle Management)
================================

AWS S3 生命週期管理功能有以下幾個重點：

- 可以根據預先定義好的規則，自動協助使用者將存放在 S3 的 object 在不同的 storage tier 中移動，讓檔案儲存的方式以更自動化且節省成本的方式進行，
> 類似地端儲存中，是將資料從 `hot`(frequently	accessed) -> `warm`(less	frequently	accessed) -> `cold`(long-term	backup	or	archive) 的概念)

- 可與版本控管(version control)的機制結合

- 結合了版本控管的機制後，就可以針對現有版本(current version)與先前所有版本(previous versions)進行更細粒度的管理


以下是幾種常見的生命週期規則設定：

- 建立日期超過 30 天的檔案，自動移動到 `Standard - IA` tier 中

- 超過 60 天的檔案，再從 `Standard - IA` tier 移動到 `Glacier` tier

- 超過 90 天的檔案，可以永久刪除
> 若資料有移到 Glacier，要把刪除的日期設為 90 天以上，因為 Glacier 費用低消是 90 days

比較需要注意的地方是，上述的規則是以`檔案建立時間`為基準計算，但有可能檔案建立了很久，依然有大量的存取；因此或許應該考量的是`檔案存取時間`可能相對的較為合理點，但目前 AWS 並沒有提供這功能，因此這邊先列入後續追蹤。


Cross Region Replication
========================

- 要啟用 cross region replication 的功能，source & destination bucket 都必須要啟用 versioning 才可以

- 設定 replication 前已經存在的 object 不會自動被同步，只有後續上傳的 object & 版本資訊會被同步

- 任何與 object 相關的 metadata or ACL 被變更時，都會觸發 replication 的工作執行

- 開啟 object 的 public access 屬性也同樣會被複製到 destination bucket，**但必須兩個 bucket 都先打開 public access 才可以**

- 必須設定 IAM policy，用以提供 S3 合適的權限進行 replication 工作

- 單一 bucket 無法同步到多個 region，但可透過 `region1_bucket -> region2_bucket -> region3_bucket` 的方式達成相同效果 (daisy chain 似乎也不行)

- Delete marker 本身是不會被跨 region 複製的，因此刪除 source bucket 的資料的行為並不會複製到另一個 region 的 destination bucket

- 在 source bucket 刪除特定的版本 or delete marker 的行為，都不會被複製到另一個 region 的 destination bucket



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)


## Version Control

- [使用版本控制 - Amazon Simple Storage Service](https://docs.aws.amazon.com/zh_tw/AmazonS3/latest/dev/Versioning.html)

## Lifecycle Management

- [物件生命週期管理 - Amazon Simple Storage Service](https://docs.aws.amazon.com/zh_tw/AmazonS3/latest/dev/object-lifecycle-mgmt.html)

## Cross Region Replication

- [複寫 - Amazon Simple Storage Service](https://docs.aws.amazon.com/zh_tw/AmazonS3/latest/dev/replication.html)