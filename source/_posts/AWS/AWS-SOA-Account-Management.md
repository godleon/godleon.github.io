---
layout: post
title:  "AWS SOA 學習筆記 - Storage Service(Database)"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 Account Management 相關服務(Organization, Control Tower、Service Catelog)以及費用優化相關服務(Billing Alarm、Cost Explorer、Budget、Cost Allocation Tags、Compute Optimizer)在實際操作維運上需要注意的事項"
date: 2021-11-06 11:30:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - AWS Organization
  - AWS Cost Optimization
---

AWS Service Health
===================

## AWS Serive Health Dashboard

當服務不穩定時，有時不一定是來自於程式端 or 設定的問題，也有可能來自於 AWS 底層服務的問題所造成，而若想知道目前 AWS 底層服務目前的狀況，可以到 AWS 提供的 [Service Health Dashboard](https://status.aws.amazon.com/) 查詢；這個 dashbord 中直接告訴你目前哪個 region 的哪個服務目前是否異常，往下拉可以看到一週內每天的服務健康狀況，同時也可以加入 RSS 訂閱通知。

## AWS Personal Health Dashboard

上面的是 AWS Global Service 的狀況，而若是要了解到目前我們所使用的 Service 是否被某些底層服務異常所影響，可以透過 [Personal Health Dashboard](https://phd.aws.amazon.com/) 進行了解，這個 dashboard 有以下特色：

- Global service

- 顯示哪些 AWS 底層服務的狀況會影響到你，並說明原因

- 顯示會對使用者所擁有的哪些 resource 造成影響

- 列出 issue 並提供可用來處理此 issue 的操作指示

- 會顯示相關的 maintenance event 訊息

- 可透過 AWS Health API 存取相關內容

- 可透過 AWS Organization 匯總來自於多個帳號的相同類型資訊

了解到 Personal Health Dashboard 可提供的資訊後，若是發生狀況時需要被通知，那就可以透過 EventBridge，搭配 SNS or Lambda + Slack 進行通知。


Organization
============

## Overview

Organization 是個可以協助管理多個 AWS account 的功能，有以下特色 & 使用規範：

- 屬於 Global Service 

- 允許管理多個 AWS Account，並且整併多個帳號的帳單(可針對總體用量取得使用折扣)

- 可共享 RI or Saving Plan 折扣，但需要在 payer account & 其他 account 都開啟 sharing 的功能(可以關閉不共享)

- 主要的帳號為 `master account`，無法變更；其他帳號則為 `member account`

- member account 只能隸屬於某個 organization

- 提供可進行 AWS account 自動建立的 API

那一般 Organization 是用在什麼情況呢....? 一般若是想針對 department / cost center 進行分類、甚至是對 dev/test/production 環境進行分類時可以使用，會有以下優點：

- 達到比較好的環境隔離

- 每個環境(or 分類)可以透過 SCP(Service Control Policy) 指定不同的存取策略

- 每個 account 有各自的 service quota limitation

了解了 organization 的特色後，那管理上可能會與單一的 account 有所不同：

- 改成使用 multiple account 而非一個 account 加上多個 VPC

- 透過 `tag` 的方式來對帳號進行不同分類

- 若要達到稽核的目的，需要在每個 account 都開啟 CloudTrail，並收到統一的 S3 bucket 進行管理

- 將 CloudWatch log 統一送到專門處理 loggin 的 account

- 需要建立 cross account role 作為管理用途


## Organization Unit

![AWS Organization Unit](/blog/images/aws/AccountManagement/Organization-OU.png)

Organization 機制的運行，會仰賴透過 OU(Organization Unit) 的方式，每個帳號都會屬於某一個 OU；此外，OU 底下可能還會有 OU，因此有可能長成下面的樣子：

![AWS Organization Unit example](/blog/images/aws/AccountManagement/Organization_OU-example.png)


## Service Control Policy (SCP)

有組織就會有管理，而在 organization 的機制中，就是透過 SCP 進行管理，其運作行為如下：

- 無法套用在 master account 上；而且對 service-linked role 沒有影響

- policy 可以套用在 OU or account level，且會套用在 AWS account 下的所有 user & role，包含 root account

- 透過黑白名單的方式針對可用的 IAM action 進行管理

- SCP 必須清楚指定 allow 的內容，並非預設 allow anything

- 定義的方式 & 內容跟 IAM 權限完全相同

然而 SCP 套用在 OU or account 所影響的範圍都各自不同，以下是個詳細的範例：

![AWS Organization SCP example](/blog/images/aws/AccountManagement/Organization_SCP-example.png)


## 在不同 Organization 之間 Migration Account 的方式

一般 AWS account 若要更換 organization，流程如下：

1. 從 old organization 移除 member account

2. new organization 發送 invitation 給 AWS account

3. AWS account 可以在 organization console 中看到 invitation，接受即可加入 new organization

> 若是要移動 master account，那需要先移除所有的 member account 後，再按照上面的步驟加入 new organization


## 其他管理須注意事項

### IAM policy

![AWS Organization IAM policy](/blog/images/aws/AccountManagement/Organization_IAM-policy.png)

在 IAM policy 的部份，organization 中會多出一個 `aws:PrincipleOrgID` 的 condition key，可以用來表示 organization 中所有的 AWS account；因此若是要針對 organization level 的權限設定，可以直接拿這個 condition key 來使用。

### Tag policy

以下是一個 tag policy example：

```json
{
    "tags": {
        "costcenter": {
            "tag_key": {
                "@@assign": "CostCenter"
            },
            "tag_value": {
                "@@assign": [
                    "100",
                    "200"
                ]
            },
            "enforced_for": {
                "@@assign": [
                    "secretsmanager:*"
                ]
            }
        }
    }
}
```

基本上 tag policy 是用來達成以下目的：

- 在 organization 中強制 tag 設定的標準

- 對於透過 tag 來進行 resource 的分類有一致性的標準

- 定義 tag key & 可用的 value 哪些

- 有效透過此方式管理成本分配 & 執行其他 attribute-based access control

- 避免不合法的 tag 被使用

- 方便用來產生報告

- 可透過 CloudWatch Event 隨時監控是否有不合規的 tag 出現


Control Tower
=============

前面有提到可以用 Organization 功能進行 multiple account 的管理，並透過 SCP 機制提供彈性的權限管理功能，但這些事情從完全沒有到建置完成需要花費很多功夫，而 Control Tower 就是一個用來簡化整個工作的服務。

Control Tower 的功能，就像是以 Organization + Multiple Account 為基礎，為管理者打造出完整的框架；Control Tower 是怎麼辦到的? 主要是因為它提供了以下幾項功能：

- 透過簡單的點擊設定就可以完成 Organization + Multiple Account 的設定 

- 透過 `guardrail`，可以進行 policy 的自動化管理 (service control policy)

- 可以自動偵測 policy 是否違反規定

- 提供方便的監控功能來檢視設定是否合規



Service Catalog
===============

## Overview

想像一個場景，如果公司內有很多人需要在 AWS 上建立資源並開始使用並進行開發測試(例如：建立 LAMP stack)，但這些人可能對於 AWS 的服務並非都很熟悉，那要怎麼解決這問題?

一開始想到的可能是透過 CloudFormation 建立出這些資源，但 CloudFormation 其實是個需要花不少時間學習的服務，而 `Service Catalog` 的角色正式定位在解決這樣的問題上。

![AWS Service Catalog](/blog/images/aws/AccountManagement/ServiceCatalog_overview.png)

Service Catalog 有以下特色：

- 將 CloudFormation 包裝一層，隱藏複雜的細節，並透過 `product` 的形式呈現給使用者 (其實是 `Product` + `Profolio`)

- 可以針對每個 product 作權限的管控 (透過 IAM permission)

- 面向使用者的是 product，而使用者可以使用這些 product 建立屬於自己的 AWS 資源

## Sharing Options

![AWS Service Catalog - sharing options](/blog/images/aws/AccountManagement/ServiceCatalog_sharing-option.png)

Service Catalog 中的設定可以分享給 Organization 中的其他 AWS account，並且有兩種 sharing option 可用，分別是 `reference` & `copy`，而這兩者的差別其實是顯而易見的：

- 透過 reference 的方式，origin portfolio 改變了，引用這個 portfolio 的 AWS account 也會反應這樣的改變

- 透過 copy 就不是這樣了，origin portfolio 改變了，需要 recipient account 自行進行變更

## TagOption Library

![AWS Service Catalog - TagOption Library](/blog/images/aws/AccountManagement/ServiceCatalog_TagOption-Library.png)

TagOption Library 的概念其實很簡單，就是為 portfolio 預先制定一組 tag，當使用者透過 portfolio 產生出對應的 AWS resource 後，這些 resource 都會自動帶上對應的 tag，方便後續管理。

此外，也可以定義哪些 tag 才可以使用 & 分享給 Organization 中的其他 AWS Account。


費用管理相關服務
=============

## AWS Billing Alarm

- billing 因為是 Global 等級的服務，因此相關的資料都存放在 `us-east-1` region 中

- 這對應到的是真實的費用，並非某個 project 的費用

- 需要啟用 Billing Alarm 才會有 billing 相關的 metric 可用

## AWS Cost Explorer

- 提供視覺化的 dashboard 讓使用者知道 AWS 的花費 & 對應的資源用量 (時間自訂, by month/day/hour)

- 可自訂 dashboard 來分析花費 & 資源使用狀況

- 可以協助用來選擇最合適的 saving plan 進行節費

- 可預測未來 12 個月的費用增長

## AWS Budges

- 用來設定 budget，並可藉此設定 alarm

- 不僅可以對使用花費設定 budget，共有四種類型的 budget，分別是 `Cost`、`Usage`、`Reservation`、`Saving Plans`

- 可設定 filter，條件可設定 Service、Linked Account、Tag、Purchase Option、Instance Type、Region、AZ、API operation .... 等等

- RI 可以用來追蹤利用率；RI 適用於 EC2、Elasticache、RDS, Redshift ... 等服務

- 每個 budget 最多可設定 5 個 SNS notification

## Cost Allocation Tags

- 這是以 tag 為基礎將費用進行客製化分類的功能

- tag 分為兩種，分別為 AWS 自行產生的 tag，prefix 為 `aws:`(例如：`aws: createdBy`)；另一個則是使用者自行定義的，prefix 為 `user:`(例如：`user: Application`)

- 從 Cost Allocation Tags 功能 console 中啟用指定的 tag 後，後續的 report 就會根據 tag 將其分類出來，提供更清晰的呈現

## AWS Compute Optimizer

- 可以針對 workload 推薦最佳化的 AWS resource 規格推薦，最高可以省下 25% 的費用

- 透過 Machine Learning 的技術，分析實際資源使用狀況(根據實際配置 & CloudWatch metric)，協助使用者選擇最佳的資源規格

- 支援 EC2 instance/ASG、EBS volume、Lambda Function

- 可將建議報告送到 S3



References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)

- [What is AWS Organizations? - AWS Organizations](hhttps://docs.aws.amazon.com/organizations/latest/userguide)

- [AWS Control Tower - govern a new secure, multi-account environment](https://aws.amazon.com/controltower)

- [What Is AWS Service Catalog? - AWS Service Catalog](https://docs.aws.amazon.com/servicecatalog/latest/adminguide)

- [AWS Compute Optimizer](https://aws.amazon.com/compute-optimizer/)