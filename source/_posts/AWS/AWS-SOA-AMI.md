---
layout: post
title:  "AWS SOA 學習筆記 - AMI(Amazon Machine Image)"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 AMI(Amazon Machine Image) 服務在實際操作維運上需要注意的事項"
date: 2021-08-15 15:30:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - AMI
---



使用 AMI 進行 EC2 Instance Migration
===================================

## 在不同 AZ 中遷移

要讓 EC2 instance 在不同 AZ 中遷移，是無法直接 stop instance 之後完成的，因為 EBS volume 一旦建立後就沒辦法更換 AZ，因此需要透過以下流程：

1. 在 source EC2 instance 中建立 AMI

2. 使用上一個步驟建立的 AMI，產生一個新的 instance 在另一個 AZ 中


## 分享 AMI 給不同 Account

若是要分享 AMI 給不同的 Account，其實很容易，只是需要注意以下兩個重點：

1. AMI 是否有被加密

2. 若是被加密，那就需要分享出原始加密用的 Key，另外一個 Account 才可以解密使用

![AWS EC2 Instance Connect](/blog/images/aws/AMI/AMI-share-between-accounts.png)

上面給了一個相對複雜的例子，但很明確說明了若是 AMI 有進行加密，正確的處理方式：

1. Account A 中產生了一個 AMI，透過 CMK-A 建立

2. Account A 除了分享 AMI 之外，也要提供 Account B 足夠的 IAM permission 來存取 CMK-A 進行解密

3. Account B 解密完成後，在複製 AMI 的過程中，還可以額外使用自己的 CMK-B 進行加密後再使用



EC2 Image Builder
=================

為了方便自動化的建置 AMI，AWS 提供了一個 Image Builder 的服務，這服務有以下幾個特性：

- 可自動化的建立 machine image or container image

- 不僅可協助建置 image，還可進一步的執行驗證流程

- 完成後產生的 AMI 可以發布到多個 region 中

- 可透過排程的方式進行，也可手動觸發

- 這是免費的服務，僅需要為執行 image building 工作所花費的 resource 付費就好


Image Builder 服務的運行流程如下圖：

![AWS Image Builder processes](/blog/images/aws/AMI/AMI-image-builder-process.png)

1. 根據在 Image Builder 中的設定建置 AMI

2. 進行測試，驗證 AMI 的正確性 & 可用性

3. 驗證無誤，協助發布到指定的 region



AMI 在 production 環境上的管理應用
===============================

- 可設定規則，強制要求使用者僅能使用帶有特定 tag 的 AMI (例如：`Environment = production`)

- 搭配 **`AWS Config`** 服務，找出沒有符合使用規範(必須使用特定 tag 的 AMI) 的 EC2 instance (如下圖所示)

![AWS AMI in production with AWS Config service](/blog/images/aws/AMI/AMI-in-production-with-AWS-Config-service.png)



References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)