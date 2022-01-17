---
layout: post
title:  "AWS SOA 學習筆記 - Security & Compliance"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 Security 相關服務(Inspector、Trusted Advisor、Secrets Manager)在實際操作維運上需要注意的事項"
date: 2021-11-10 06:20:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - SOA
  - Security
  - AWS Inspector
  - AWS Trusted Advisor
  - AWS Secrets Manager
---



Penetration Test
================

AWS 允許客戶為了做些安全性上的評估，對特定的服務進行滲透測試，例如：EC2 instance、NAT Gateway、ELB、RDS、Aurora、CloudFront .... 等等；但並非 AWS 允許客戶為所欲為的進行安全性相關的測試，有些行為還是會被禁止的，例如：[DNS zone walking](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/best-practices-resolver-zone-walking.html)、DDoS、port flooding ... 等等。

以下是 AWS 官網文件提供的詳細列表資訊供參考：

![Customer Service Policy for Penetration Testing](/blog/images/aws/Security/PenetrationTesting_CustomerServicePolicy.png)



AWS Inspector
=============

- AWS Inspector 是個自動化的安全性評估工具，可以協助用來分析目前運作中的 OS 是否包含了一些潛在的漏洞威脅，目前僅能用在 EC2 instance 上

- 也可以針對網路存取進行安全性相關分析(這屬於 Agentless 的操作)

- Inspector 是透過在 EC2 instance 中安裝 inspector agent 來達成資安分析的目的，稱為 `Host Assessment`，可分為以下三個部份：
  - Common Vulnerabilities and Exposures
  - Center for Internet Security(CIS) Benchmarks
  - Security Best Practice

- 分析完後可以將結果透過 SNS 發送通知，也可以產生安全性相關的評估報表

![AWS Inspector](/blog/images/aws/Security/Inspector_overview.png)



Logging
=======

為了提供強化安全性 or 提供稽核的目的，AWS 很多服務都有相對應的 service log 可以提供，例如：

- CloudTrail trails：追蹤所有 API call

- Config Rules：針對 config & 相關合規需求進行持續檢查驗證

- CloudWatch Logs：可以將 application log 都往這邊送

- VPC Flow Logs：提供 VPC 內 IP traffic 的資訊

- ELB Access Log：記錄經過 ELB 的 reqeust 的 metadata

- CloudFront Log：每個 edge server 收到 HTTP request 的 access log

- WAF Log：HTTP reqeust 經過 WAF 分析處理後的結果

這些 log 可以長期的存放在 S3 中，並透過 Athena 進行進一步的分析；若是這些資料需要封存，可以送到 Glacier。



GuardDuty
=========

GuardDuty 是個使用 Machine Learning 來偵測威脅是否存在的服務，而要進行威脅偵測，總是需要有資料來源，這些資料來源包含以下內容：

- CloudTrail Logs：偵測不尋常的 API call，未授權的佈署行為

- VPC Flow Logs：異常的內部流量 & IP

- DNS Logs：偵測被入侵的 EC2 instance 送出包含編碼內容的 DNS 查詢

![AWS GuardDuty](/blog/images/aws/Security/GuardDuty.png)

對於偵測到的異常，可以透過 EventBridge(CloudWatch Event) 送到 SNS or Lambda 進行更進一步的處理。

> 可用來偵測加密貨幣攻擊



Trusted Advisor
===============

這是一個用來分析 AWS account 下的服務使用狀態，並提供以下的建議：

- Cost Optimization

- Performance 

- Security

- Fault Tolerance

- Service Limit

一般的帳號僅提供 core checks；若要完整的檢查功能，則需要購買 Business & Enterprise support plan 才會有。

以下是 Trusted Advisor 可能給出的建議範例：

- Cost Optimization
  - 資源使用率低的 EC2 instance、沒在用的 ELB & EBS volume
  - 提供 RI & Saving Plan 採購建議

- Performance 
  - 使用率高的 EC2 instance、CloudFront CDN 相關優化手段
  - EC2 與 EBS 的 throughput 優化、DNS alias record 的優化建議

- Security
  - 建議 root account 啟用 MFA、IAM Key rotation、被外部服務使用的 Access Key
  - 開放太多的 S3 bucket permission、沒有限制的 security group 設定

- Fault Tolerance
  - ELB AZ balance
  - ASG/RDS Multi-AZ



Secrets Manager
===============

## Overview

- 顧名思義，專門用來儲存 secret 用，內容會透過 KMS 進行加密；可強制設定每隔多少天就需要 rotate

- 需要仰賴 Lambda 進行 rotation，而每次 rotate 就會產生新的 secret

- 可與 RDS 直接整合

## 監控相關事件

在 AWS 要執行事件監控這件事情，肯定是要搭配 `CloudTrail` & `CloudWatch`(or `EventBridge`) 來完成，如下圖：

![Secrets Manager - Monitoring](/blog/images/aws/Security/SecretsManager_monitoring.png)

- CloudTrail 會記錄 Secrets Manager 相關的 API

- 搭配 CloudWatch 的 metric filter，可以用來協助捕捉與安全 or 合規相關的 API call，也可用來協助排除維運時發生的錯誤

- CloudTrail 還會記錄以下非 API 的事件：
  - RotationStarted
  - RotationSuccessed
  - RotationFailed
  - RotationAbandoned
  - StartSecretVersionDelete
  - CancelSecretVersionDelete
  - EndSecretVersionDelete

- 有了上面的 metric & event，就可以搭配 CloudWatch Alarm 來進行些自動化的工作

## 排除 Rotation 時發生的問題

rotation 是 secrets manager 一向相當重要的功能，不過因為這功能仰賴於 Lambda Function，因此可能在程式 or 權限設定上都有出錯的可能，因此了解以下的架構，會方便管理者遇到問題時進行查找：

![Secrets Manager - Troubleshooting Rotation](/blog/images/aws/Security/SecretsManager_debug.png)

- Secrets Manager 所有的 API call & non-API event 都會送到 CloudTrail

- 要進行 rotation 時，會呼叫 Lambda Function，同時更改 RDS 密碼(如果有的話)，而相關執行的 log 都會發送到 CloudWatch Logs 上

- 上述過程若是不順利，管理者就可以透過 CloudTrail or CloudWatch Logs 的資訊來排除問題

## Secrets Manager v.s. SSM Parameter Store

SSM Parameter Store 同樣也可以存 secure string，那與 Secrets Manager 有什麼差異呢? 列舉如下：

- Secrets Manager 比較貴

- Secrets Manager 可以搭配 Lambda Function 做自動的 secret rotatio；在 SSM Parameter Store 要完成這件事情需要搭配 EventBridge(CloudWatch Event)，架構如下圖：

![Key Rotation comparison](/blog/images/aws/Security/SecretsManager_rotation-compare-with-SSM.png)

- Secrets Manager 必須搭配 KMS；SSM Parameter Store 則不一定

- 可以使用 SSM Pameter Store API 拉取 Secret Manager 中的 secret



IAM 安全工具
===========

IAM 服務中有提供幾個提昇安全性的工具，分別是 `Credentials Report` & `Access Advisor` & `Access Analyzer`，以下分別進行介紹：

## Credentials Report

其中 `Credentials Report` 是屬於 account level，可以用來產生報告，報告中會列出 AWS account 中的 IAM user & 設定(運行)狀況，可協助管理者用來評估是否要求使用者提昇安全性。(例如：啟用 MFA)

## Access Advisor

`Access Advisor` 屬於 user level，會列出當前的 IAM user 被賦予的權限，以及最近存取的服務有哪些；而透過這些資訊，就可以適時的調整 IAM policy，進一步達成 least privilege 的目標。

## Access Analyzer

![IAM Access Analyzer](/blog/images/aws/Security/IAM-Access-Analyzer.png)

- 可用來檢測哪些 AWS resource 可被外界進行存取(例如：S3 Bucket、SQS queue、Secret Manager secret ... 等等)

- 可定義 **`Zone of Trust`**，範圍可能像是 AWS Account or Organization

- 若是有發現在 Zone of Trust 定義之外的範圍被存取，就會提供這些資訊出來



References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)

- [Penetration Testing - Amazon Web Services (AWS)](https://aws.amazon.com/security/penetration-testing/)

- [What is Amazon Inspector? - Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/userguide)

- [AWS Trusted Advisor](https://aws.amazon.com/premiumsupport/technology/trusted-advisor/)

- [What is AWS Secrets Manager? - AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

- [Getting credential reports for your AWS account - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_getting-report.html)