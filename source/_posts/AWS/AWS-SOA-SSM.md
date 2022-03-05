---
layout: post
title:  "AWS SOA 學習筆記 - SSM(System Manager)"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 SSM(System Manager) 服務在實際操作維運上需要注意的事項"
date: 2022-01-22 20:45:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - SOA
  - SSM
---

Overview
========

在 SSM & OpsWorks 這個部份，主要關注的重點在於：

- 如何有效管理一群 EC2 Instance 以及地端環境的機器

- 如果有效率的在大規模的環境中執行 software patch 相關工作

- 如何自動化執行工作

- 不同環境之間參數的管理

- 如何利用 Configuration Management Tool(例如：`Chef` & `Puppet`)



SSM (System Manager)
====================

## 功能目標

AWS SSM 中提供了相當多的工具，主要就是要達成以下目標：

- 讓使用者方便大規模管理 EC2 Instance & 地端環境的機器

- 取得底層 infrastructure 的狀態，提供維運人員方便進行管理

- 更容易發現問題

- 讓 software patch 相關工作自動化，以符合效能 or 安全上的規範

- 同時支援 Linux & Windows

- 與 CloudWatch & AWS Config 有很好的整合

以上提到了很多點，其實總結來說，這些功能都是設計出來方便使用者管理一群機器、提高安全性 ... 等用途。


## 功能分類

由於 SSM 功能很多，因此分成以下幾個大類：(子項目為重點項目)

- Resource Group

- Operation Management

- Application Management
  - Parameter Store

- Change Management
  - Automation
  - Maintenance Windows

- Node Management
  - Fleet Manager
  - Inventory
  - Session Manager
  - Run Rommand
  - State Manager
  - Patch Manager

- Shared Resources
  - Documents


## SSM 相關功能如何運作 ?

AWS SSM 所提供的功能可以同時管理 EC2 Instance & 地端的機器，這件事情是怎麼辦到的? 架構如下圖：

![AWS SSM - How it works?](/blog/images/aws/SSM/SSM-how-it-works.png)

需要完成以下的流程：

1. 使用者必須在要管理的機器中安裝 SSM agent，不論是 EC2 instance or 地端的機器

2. 在 Amazon Linux 2 or 某些 Ubuntu AMI 中已經內建不需要安裝
> 若是需要安裝，可以參考[官網文件說明](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-manual-agent-install.html)

3. 確認 EC2 instance 有配置足夠權限的 IAM role 來與 SSM 溝通並執行相關操作(可直接利用 AWS Managed IAM Role `AmazonEC2RoleforSSM`)；而地端就必須要有足夠權限的 AWS access/secret key


Tags & Resource Group
======================

了解了 SSM 是透過 Agent 形式與 AWS 溝通，接著思考一下，若是有很多台機器需要管理時，要如何有效的進行分類以方便後續處理? 而 AWS 提供了 [`Tag`](https://docs.aws.amazon.com/general/latest/gr/aws_tagging.html) & [`Resource Group`](https://docs.aws.amazon.com/ARG/latest/userguide/resource-groups.html) 兩項功能來滿足這個需求。

## Tag

- 許多 AWS resource(非全部) 都可以用 key/value 的形式來設定 tag，其中 EC2 instance 很常被用來設定 tag

- 命名的規則是很自由的，可以隨意根據需求去設置，但建議還是參考一下 [AWS 提供的規範 & 建議]([`Tag`](https://docs.aws.amazon.com/general/latest/gr/aws_tagging.html))來進行

- tag 通常被用在作為 **資源分類**、**自動化**、**成本分類/分析** .... 等目的

- 大原則是：**有太多 tag 總比沒有好**

## Resource Group

- Resource Group 則是對每個 AWS resource 進行邏輯分組的服務，分組依據可來自 `tag` or `cloudformation stack`，屬於 regional service

- 為 AWS reousource 建立邏輯分組，通常會根據 **Application**、**Application stack 中不同的層級**、**環境**(production or development) ... 等等資訊


SSM Document & Run Command & Automation
=======================================

## SSM Document

- SSM Document 是種預先定義好的一連串執行命令，在 AWS console SSM 的 `Shared Resources` 可以找到，其中預設都是 AWS 所提供的，可以針對 AWS resource 進行一些特定操作，例如：重啟 EC2 instance(`AWS-RestartEC2Instance`)

- 除了 AWS 預設提供的之外，使用者也可以定義屬於自己的 Document

- Document 中可以定義 parameter & action 兩個部份，其中 action 中定義的就是指定要執行的特定操作

- Document 只是定義要執行的動作，但不會明確指定要用在哪個 AWS resource，何時進行，因此會與 `Run Command`、`State Manager`、`Patch Manager`、`Automation` ... 等服務搭配使用

## Run Command

![AWS SSM Run Command](/blog/images/aws/SSM/SSM-Run-Command.png)

- 顧名思義，Run Command 就是用來執行一個 Document(像是個 script 的概念)，或者單一指令

- 透過 `Tag` or `Resource Group`，可以一次對多個 AWS resource 執行 Run Command

- 提供 Rate Control & Error Control 的功能

- 透過 SSH Agent 進行，因此不需要 SSH

- Command output 可以顯示在 console，也可以送到 S3 bucket or CloudWatch Log

- 若要發送狀態通知(In Progress, Success, Failed)，可以送到 SNS

- EventBridge 也可以根據 event 來呼叫 Run Command

## Automation

![AWS SSM Automation](/blog/images/aws/SSM/SSM-Automation.png)

- 設計用來簡化常見針對 AWS resouece 的維護 & 佈署相關操作(例如：restart EC2 instance、建立 AMI、EBS snapshot ... 等等)

- 除了 Document 之外，Automation 還有更多 AWS 為其定義好的日常維運內容(稱為 `Runbook`)，例如：
  - Remediation
  - Patching
  - Security
  - Instance Management
  - Data Backup
  ...... 等等

- 除了 AWS 預先提供的日常維運操作 Runbook 之外，也可以自訂 Runbook

- 可在以下情境被觸發：
  - 手動透過 AWS console/CLI/SDK
  - 透過 EventBridge
  - 在 maintenance windows 時排定
  - AWS Config 中觸發的 rules remediation


Parameter Store
===============

## Overview

![AWS Parameter Store](/blog/images/aws/SSM/SSM-ParameterStore.png)

- Parameter Store 是個 serverless 服務，提供安全存放 configuration & secret 的地方

- 與 KMS 有完美的整合，協助用來加密機敏資訊

- 提供 version control 的功能

- 權限的管理方式是以 `by path` + `IAM` 的方式處理；基本上概念就是，來存取 Parameter Store 會檢查有無正確的 IAM 權限，若是有就提供，若是存取加密資訊，也同時需要有 KMS 的存取權限才行

- 可以搭配 EventBrige 送 configuration 調整通知

- 可與 CloudFormation 整合

## 存放結構

configuration 存放於 Parameter Store 的結構設計，可以 by dept，例如：

- /dept1/
  - app1/
    - dev/
      - db-url
      - db-pwd
    - prod/
      - db-url
      - db-pwd
  - app2/
- /dept2

若是以 reference 的概念設定，例如：
- /aws/reference/secretsmanager/secret_id_in_secret_manager
- /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2


SSM Inventory & State Manager
==============================

## Inventory

- 用來蒐集 EC2 instance & on-premise 機器上的 metadata (包含安裝了哪些軟體、OS drivers、更新、運行哪些服務 ... 等等)

- 這些資料可以在 AWS console 中查詢，也可以存在 S3 並搭配 Athena or QuickSight 這一類的服務進行查詢

- 可以指定資料蒐集的時間間隔

- 可跨多個 AWS account & region 查詢 (代表可以將所有 instance 資料整合在一起查詢)

- 可以針對 custom metadata 設定 custom inventory (例如：機器所在的機櫃位置)

## State Manager

如果需要管理的 EC2 instance 數量龐大，想要找茫茫大海中找出含有特定特徵的 instance，光用 tag or resource group 還是不夠的，舉個例來說，假設要尋找有安裝防毒軟體的 instance，若是 tag 忘了標上，可能就很難找到這樣的 instance，而這時 state manager 就可以派上用場。

透過從 inventory 中蒐集來的 instance metadata，state manager 可以定期(or 不定期)自動與這些 metadata 進行查詢關聯(association)；使用者可以自定義這些關聯的內容，由 state manager 協助找出對應的 instance 並建立關聯，方便後續進一步的管理工作。

而自動化關聯這件事情的實現，是透過 SSM Document 來完成的。


Patch Manager
=============

## Overview

![AWS SSM Patch Manager](/blog/images/aws/SSM/SSM-PatchManager.png)

- 用來自動對 managed instance 進行 patch 的服務，包含 OS、application、security ... 等等

- 同時支援 EC2 instance & on-premise 機器

- 作業系統的部份，支援 Linux、macOS, Windows

- 可以選擇立即執行 patch 工作，或是設定 Maintenance Window 的方式來設定排程進行

- 可以掃描 instance 並產生 patch 相關報告，並且將報告送到 S3

- 實際執行時，還是透過 Run command 來執行 Document 中的內容進行 patch 工作

另外，關於 patch 要執行到何種程度，以及要如何正確指定要 patch 的 instance，需要額外搭配 **`Patch Baseline`** & **`Patch Group`** 來完成。

## Patch Baseline

- 用來定義哪些 patch 要安裝，哪些不要安裝

- 可自訂 custom patch baseline，用來設定 approved/rejected patch

- patch 可以設定幾天後自動 approve

- 預設只安裝 critical & security 相關的 patch

## Patch Group

- 設定要 patch 的 instance 分類 (例如：dev / test / prod)

- 要指定 patch group，就必須為 instance 加上 tag key `Patch Group`；每個 instance 只能屬於一個 patch group

- 每個 patch group 僅能註冊一個 patch baseline

## Maintenance Windows

- 用來設定何時執行指定操作，這些操作可能包含更新 OS、更新驅動程式、安裝軟體 .... 等等

- 設定中主要包含以下內容：
  - schedule
  - duration
  - 包含哪些 instance
  - 包含哪些 task


Session Manager
===============

![AWS SSM Session Manager](/blog/images/aws/SSM/SSM-Session-Manager.png)

- 用來對 EC2 instance & on-premise 機器啟動一個 secure shell 並進行連線

- 可透過 AWS console、AWS CLI or Session Manager SDK 來建立連線

- 支援 Linux、macOS, Windows

- 會將連線的記錄 & 在 instance 上執行的指令都 log 下來(session log)，且可轉存到 S3 or CloudWatch Logs

- CloudTrail 可用來記錄 `StartSession` event


References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)