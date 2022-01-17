---
layout: post
title:  "AWS SOA 學習筆記 - Storage Service(Database)"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 Monitoring & Auditing 在實際操作維運上需要注意的事項，相關的服務包含 CloudWatch、EventBridge、CloudTrail, AWS Config ... 等等"
date: 2021-11-02 10:30:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - SOA
  - CloudWatch
  - EventBridge
  - CloudTrail
  - AWS Config
---


CloudWatch
==========

## CloudWatch Metric

- 若是希望 Auto Scaling Group 可以有更快的擴展速度，那就需要打開 detailed monitoring 功能

- Free Tier 足夠提供 10 個 detail monitoring metric 的額度


## CloudWatch Custom Metric

- 用來上傳自訂的 metric 到 CloudWatch 用 (例如：EC2 instance memory 使用狀況、磁碟空間)

- 透過 `PutMetricData` API

- 可為 metric 增加 dimension(其實就是 attribute) 變成多維的資料

- 以上傳頻率來說，標準是一分鐘傳一次。若是要密集一點，最短可以到 1 秒鐘


## CloudWatch Dashboard

- Global service

- 可以包含多個 region 的監控資訊，甚至來自不同 AWS 帳號也可以

- 可設定 dashboard 的 time zone & range，以及 auto refresh 的時間(10s, 1m, 2m, 5m, 15m)

- 可透過 EMail or Cognito+SSO，將 dashboard 可以分享給沒有 AWS 帳號的使用者

- 價格部份，使用 3 個 dashboard 內(上限為 50 個 metric)是免費的；之後一個 dashboard 每個月要收費 $3


CloudWatch Log
==============

## Overview

要使用 CloudWatch Log，必須先知道 Log 透過什麼機制送進 CloudWatch 的，首先需要了解兩個部份：

- Log Stream：這指的是同一個來源所傳過來一連串的 log event，這來源可能是 application、log file、container .... 等等

- Log Group：這是為了管理方便而再多包裝一層的功能，讓多個 log stream 可以共用 retenstion、monitoring、ACL ... 等設定

當 log 資料送進 CloudWatch Log 後，為了進一步處理方便後續利用，可以繼續送到以下幾個地方：
- S3：透過 S3 Export，利用 `CreateExportTask` API，設定需要 12 小時左右才會生效 (非 near-real time or real-time 的作法)
- Kinesis Data Stream
- Kinesis Data Firehose
- Lambda
- Elasticsearch


## Data Source

Log 的來源可以來自以下地方：

- SDK、CloudWatch Log Agent、[CloudWatch Unified Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/UseCloudWatchUnifiedAgent.html)

- Elastic Beanstalk application log

- ECS container log

- AWS Lambda Function log

- VPC Flow log

- API Gateway

- CloudTrail (可以加上 filter)

- Route53：DNS 查詢相關 log


## Metric Filter & Insight

- CloudWatch Log 中有提供 filter 可讓使用者容易找到所需要的 log 資訊，例如：包含特定 IP 的 log，出現 "ERROR" 關鍵字的 log 資訊

- `Metric Filter` 這項功能，可以讓使用者自訂 log filter pattern，當出現符合 filter pattern 的 log 後，對應的 metric 就會增加；而這 metric 後續可以用來設定 CloudWatch Alarm 之用

- CloudWatch Logs Insight 可以用來查詢 log 並可將查詢結果加到 CloudWatch dashboard 中


## 與其他服務整合應用的情境

CloudWatch Logs 可以透過 `Subscription` & `Aggregation`，與其他服務整合，讓 Log 資訊可以延伸出更多種應用，以下舉兩個例子：

![CloudWatch Logs Subscription](/blog/images/aws/MonitorAudit/CloudWatch-Logs_subscription.png)

透過 subscription filter，可選擇將傳進 CloudWatch Logs 的 logs 轉發到其他服務，例如：

- 轉發到 Lambda Function，經過自定義的處理後，再轉送到 Elasticsearch 中儲存 (real time)

- 轉發到 Kinesis Data Firehose，經過自定義的處理後(當作實際送儲存之前的緩衝)，再批次轉發到 Elasticsearch or S3 ... 等服務 (near real time)

- 集中轉發到 Kinesis Data Streams 服務後，再轉送給 Kinesis Data Firehos/Analytics、EC2、Lambda ... 等服務進行後續處理

![CloudWatch Logs Aggregation](/blog/images/aws/MonitorAudit/CloudWatch-Logs_aggregation.png)

透過上圖的架構，可以將來自多個 AWS account(可能來自不同的 region) 的 CloudWatch Logs 統一集中轉發到 Kinesis Data Streams 服務後，接著轉送給 Kinesis Data Firehose 後，再批次儲存到 S3 中。(near real time)



EventBridge
===========

## Overview

Event Bridge 前身為 **CloudWatch Event**，主要功能是讓使用者可以針對 Event 進行更多客製化管理 or 操作，而這個服務帶來了幾個優點：

- Decoupling：將 event source, routing rule、event target ... 等角色拆分開來，可以分別獨立設計 & 擴充

- Simplified event routing：使用者可以根據實際業務 or 管理需求，設定對應的 routing rule，將 event 送到有需要得知訊息的地方

- Improved Availability：這是個全託管的 serverless 服務，可用性交給 AWS 負責，使用者只要負責使用即可

- Third party integration：甚至可以整合外部的 SAAS 服務

## CloudWatch Event

Event Bridge 包含了所有 CloudWatch Event 的所有功能，而這功能主要內容是：

- 可以攔截眾多 AWS service 所產生的 event，例如：EC2 Instace Start、CodeBuild Failure、S3、Trust Advisor .... 等等

- 可與 CloudTrail 整合，攔截所有 API call

- 可設定 schedule(or Cron)，定期發送 event

- 有了上面的 event 後，這些 event 會產生對應的 JSON payload，可將其轉送到指定的 target(不同類型的其他服務)，例如：
  - Compute：Lambda、Batch、ECS task
  - Integration：SQS、SNS、Kinesis Data Streams、Kinesis Data Firehose
  - Orchestration：Step Functions、CodePipeline、CodeBuild
  - Maintenance：SSM、EC2 Actions

## 進化的 Event Bridge

Event Bridge 是更進化的服務，還可以整合外部的 SAAS，因此也對原本的 CloudWatch Event 進行了一層封裝，因此有些新的設計概念需要釐清，大致列舉如下：

- Event Bus：將 event 的來源透過 event bus 的方式來呈現，預設為 `default`，也就是泛指來自 AWS service 的相關 event；也可以對外部的 SAAS 服務設計 `partner` event bus(針對 AWS 已經整合完成的服務)，甚至可以設定 `custom` event bus，讓 application 可以送 event 到 Event Bridge 中

- Rules：這個部份定義了要如何處理 & filter 從 event bus 收到的 event 

- Schema Registry：由於每個 event 的內容不一樣，自然會產生不一樣的 JSON payload，而這些 payload 所代表的意義，就需要使用 schema 來定義，方便後續處理 event 的 target 可以了解每個 event JSON payload 所代表的意義為何，才可以進行相對應的處理



Service Quota
=============

這裡需要注意的就是當 service quota 快接近上限時，如何主動得到 CloudWatch Alarm 的通知，有兩種方法：

1. 進入 Serivce Quota 後，找到要設定 Alarm 的選項，直接將其加入到 CloudWatch Alarm 中，後續再到 CloudWatch 中設定其他 notification or action 的細節；這樣的作法可以為所有 service quota 都設定對應的 alarm

2. 透過 Trusted Advisor：由於 Trusted Advisor 有內建常用的 service quota 的檢查，超過預定上限(約 80%) 就會檢查失敗進行通知；只是 Trusted Advisor 的檢查並不會涵蓋所有的 service quota，因此在選用此方案時還是要注意一下



CloudTrail
==========

## Overview

- 可將 CloudTrail 產生的 log 存放至 S3 or CloudWatch Logs

- CloudTrail 可以針對全部 region 套用，也可以針對單一 region 套用

- CloudTrail Events 預設儲存 90 天(指的是可以從 CloudTrail console 的 Event history 中查詢到的時間範圍)，若是要查詢到更久之前的 event，那就要透過 Athena 對存放 event 的 S3 bucket 進行查詢


## CloudTrail Event

CloudTrail Event 分為三類，分別是 `Management Events`、`Data Events`、`CloudTrail Insight Events`

### Management Events

- 在 AWS resource 產生的操作都算是此類，例如：安全性設定(IAM `AttachRolePolicy`)、routing rule 設定(EC2 `CreateSubnet`) ... 等等

- CloudTrail 預設會記錄 Management Events (只有此項預設是開啟的)

- 可將 Read Events & Write Events 分開

### Data Events

- CloudTrail 預設不會記錄 Data Events

- 此類 Event 分成兩個來源，分別是 `S3` & `Lambda Function`

- S3 記錄的是 object-level 的相關操作，例如：**GetObject**、**DeleteObject**、**PutObject** ... 等等；同時也可以將 Read/Write Event 分開

- Lambda Function 的部份則是記錄執行操作(**Invoke**)

### CloudTrail Insight Events

![CloudTrail Insight Event](/blog/images/aws/MonitorAudit/CloudTrail-Insight.png)

- 這是用來偵測帳號中的異常 API 存取行為，例如：異常資源佈署行為、達到 service limit、突然大量的 IAM 相關操作 ... 等等

- CloudTrail Insight 會分析日常的存取行為後，建立一個 baseline 作為判斷異常的基礎

- baseline 建立後就會持續的分析 `**write**` event 來偵測異常狀況

- 當 Insight Event 產生後，可以送到 CloudTrail console、S3，或是產生 EventBridge event 來進行後續自動化操作的整合



AWS Config
==========

## Overview

- AWS Config 是個可以用來協助檢查、稽核，並記錄 AWS resource 相關配置是否有符合預先定義好的規範的服務

- 可以記錄並提供 resource 隨著時間變更的過程 & 內容

- 此服務主要用來解決類似以下問題：
  - 是否有不受限制來源的 SSH 存取的 security group 存在 ?
  - 是否有任何 S3 bucket 設定了 public access ?
  - 某段時間內，ALB 做了哪些變更

- 可指定針對檢查到的任何變更都發送通知到 EventBridge or SNS

- 是個 per-region 的服務，但可以 cross-region 蒐集資料並彙整(aggregate)至單一 account

- 可將服務過程中產生的記錄資訊統一存放在 S3，日後可用 Athena 進行分析

## Config Rules

- Config Rule 其實就是定義了檢查(or 稽核)所要實現的細節，例如：檢查 security group 是否有不限制來源的 SSH 存取設定

- AWS 定義了近百個 managed config rules，可以直接拿來利用

- 若是需要自行開發 custom config rule，那需要透過 AWS Lambda (例如：檢查 EBS volume 是否屬於 gp2)

- 可以指定在任何 configuration 變更進行 Config 檢查，也可以指定一段時間檢查一次

- **需要注意的是，AWS Config** 無法阻止不合規的設定發生，單純只是個檢查後，提供相關資訊給管理者(頂多後續可以指定 remidiation action)

- 這是個收費服務，記錄的資訊越多 or 檢查的項目越多，費用就越高

## Remediation

- AWS Config 可以結合 SSM Automation，針對非合規的 configuration 部份進行自動化的處理

- 自動化的處理可以搭配 AWS managed automation document 或是自訂的 automation document
> 可用 automation document 來呼叫 Lambda Function 執行更複雜的客製化作業

以下舉個 Remediation 的例子，當發現 IAM user 已經超過一定天數沒有存取行為後，就直接廢止該 IAM user 的 access key

![AWS Config - Remediation](/blog/images/aws/MonitorAudit/Config_remidiation.png)

## Notification

notification 的部份可以搭配 EventBridge or SNS

![AWS Config - notification EventBridge](/blog/images/aws/MonitorAudit/Config_notification-EventBridge.png)
> 搭配 EventBridge 可以設定 filter，並將相關資訊往後丟給 Lambda Function / SNS / SQS ... 等服務進行後續處理

![AWS Config - notification SNS](/blog/images/aws/MonitorAudit/Config_notification-SNS.png)
> 比較簡單的作法就是直接送給 SNS 進行通知



References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)

- [What Is AWS CloudTrail? - AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide)

- [What Is AWS Config? - AWS Config](https://docs.aws.amazon.com/config/latest/developerguide)