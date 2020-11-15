---
layout: post
title:  "AWS 學習筆記 - IAM(Identity and Access Management)"
description: "此篇文章是學習 AWS IAM(Identity and Access Management) 時所留下的學習筆記"
date: 2020-04-24 05:35:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - IAM
---


What is IAM?
============

首先來看看 IAM(**AWS Identity and Access Management**) 在 AWS 原廠網站上的定義：

> AWS Identity and Access Management (IAM) enables you to manage access to AWS services and resources securely. Using IAM, you can create and manage AWS users and groups, and use permissions to allow and deny their access to AWS resources.

IAM 的主要目的如下：

- 用來管理使用者的權限等級，限制使用者可以存取的 AWS 資源範圍

- IAM 是以目前流行的 RBAC(Role-Based Access Control) 設計的，可以建立 User/Group，並將權限給予到 Group 等級

- IAM 服務是免費的


IAM 所提供的功能
==============

IAM 提供以下眾多的功能：

- 統一管理 AWS 帳號

- 可以將 AWS resource 使用權限分享給其他使用者

- 可以針對每個帳號可存取的資源權限進行很細部的控制，例如**限制某人只能對 DynamoDB 進行唯讀的存取**

- 提供 Identity Federation 功能，可與其他服務(例如：Active Directory, Facebook, Linkedin etc)整合，方便帳號管理

- 多因素認證(Multifactor Authentication)，為帳號密碼之外，額外增加一個隨時變動的密碼
> AWS 建議為每個帳號都設定 multifactor authentication

- 可為 user/device/service(例如：手機裝置) ...等對象提供暫時的存取權限，一段時間後權限即關閉

- 可自訂 password rotation policy，強制密碼一段時間後要變更

- 為了確保使用者可以安全使用 AWS 服務，IAM 與 AWS 眾多服務都有良好的整合

- 支援 PCI DSS(Payment Card Industry Data Security Standards) Compliance，方便整合外部支付的服務，提昇線上支付的安全性
> PCI DSS(支付卡產業資料安全標準)是在整合外部付費服務之用，為了提升線上支付的安全性


IAM 的核心概念 
============

IAM 的核心概念包含以下四項：

- **Users**：指的就是使用者，也可以泛指使用資源的人 or 對象

- **Group**：一群 `Users` 的集合，在 Group 中的 User 都會繼承 Group 所擁有的權限

- **Roles**：這個概念就是用來與實際的權限綁定所設計出來的，例如：`UpdateApp`，並指定 RDS & S3 的讀寫權限

- **Policies**：實際將權限綁定到 User/Group/Role(統稱為 `principal`) 的關鍵就是 Policy 了(**指定哪些操作(PUT/DELETE/UPDATE...etc)可以被使用在哪些 AWS resource 上**)，這是一個使用 JSON 格式所定義的文件，裡面會清楚描述可使用哪些 AWS resource & 可使用的權限，以下是一個例子：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```


IAM Policy
==========

- 清楚定義的 deny policy 的效果會蓋掉所有清楚定義的 allow policy

- 可以透過下列 policy document 快速封鎖特定的 IAM user 所有權限

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

- AWS 有三個重要且預先建立好的 policy template：
  - `Administrator access`：可以存取所有 AWS resource
  - `Power user access`：有 admin 存取權限，但無法做 user/group 管理
  - `Read only access`：只能檢視 AWS resource

- 一個 IAM user 可以同時與多個 IAM policy 綁定

- **IAM policy 是無法與 AWS resource/service 綁定的**，它是用來指定給 IAM user/group 用的


IAM User/Group
==============

- 每個 IAM User 都可以給予不同的權限

- IAM Group 可以一次賦予多個使用者相同的權限(`設定權限` -> `指定加入到 Group 的 User`)



IAM Role
========

- IAM Role 是為了讓 IAM User/Group、AWS Resource(例如：EC2 instance)、臨時需要存取 AWS 資源的外部帳號 ... 等取得權限用的機制

- IAM Policy 無法直接套用在 AWS service 上 (因此若是要賦予 EC2 instance 權限，那只能使用 IAM Role)

- **以上述範例來說，也可以使用 AWS credential(Access ID & Secret Key) 來達成相同效果，但完全不建議這麼做，會造成管理上很大負擔 & 安全性的隱憂**

- **每一個 EC2 instance 只能與一個 IAM Role 的設定綁定**；因此在複雜環境下，Role 權限設定要謹慎評估

- 外部帳號(不是 IAM User)需要存取權限，就必須透過整合 SAML provider(例如：AD) 的方式，搭配 IAM Role 來取得所需要的權限

## Role Type

`Role Type` 的概念很重要，這是用來決定要將權限賦予到那一種 entity(實體)上，首先要先有以下概念：

- IAM Role 會與 IAM Policy 綁定，透過此方式來讓 Role 有特定的 AWS resource 存取權限

- IAM Role 使用在特定的 entity 上，並非一般的 IAM User & Group

有了以上概念後，接著繼續介紹 IAM Role Type 有以下三種：

1. `AWS Service Role`：AWS 相關服務，例如：EC2、Lambda、Redshift ... etc

2. `Role for Cross-Account Access`：用來設定跨帳號存取權限用

3. `Role for Identity Provider Access`：若是帳號與外部系統(例如：透過 SAML)整合，就可以使用此種 role type 設定外部帳號權限


IAM Security Token Service (STS)
================================

- 用來取得臨時存取 AWS resource 用的權限(以 token 的形式提供)的服務

- 這個暫存的 credential 是有效期的，可以根據需求設定從幾分鐘到幾個小時，過期了就會自動失效

- credential 只能透過 STS API call 取得(無法從 AWS console 設定取得)

- 取得的 credential 包含 `session token`、`access key ID`、`secret access key` 三個部份

- 可與 Identity Federation 搭配；也可以與用在設定 Cross-Account Access & AWS service 權限時的 IAM Role 搭配

## 使用 STS 的好處

- 對於暫時需要 AWS resource 存取權限的應用程式，不用特定產生 credential

- 不用先建立一個 IAM identity(例如：IAM User/Group)，因為此服務是基於 **IAM Role** & **Identity Federation** 所搭配而成的

- 過期自動廢棄，不需要人工作業



IAM API Keys
============

- 若是有透過程式(非 AWS console)存取 AWS resource 的需求，就需要 `API Access Key`，例如：
  - AWS CLI
  - Windows PowerShell
  - AWS SDKs
  - 直接送到 AWS resource 的 HTTP request

- 產生後只會在一開始顯示一次，後來再也看不到了，沒紀錄到就要重新產生

- 因為 API Access Key 必須與 IAM User 綁定，來取得對應的存取權限(就端看該 user 與什麼 IAM Policy 綁定)

- 要是產生新的 API Access Key，就建議把舊的廢止

- 千萬不要把 API Access Key & Secret Access Key 放到 EC2 instance 中，可能會造成未來安全性上的漏洞(建議改用指定 IAM Role 的方式)



實作筆記
=======

## IAM is global(universal)

從 management console 進入 IAM 的功能頁面後，Region 的部份會變成 `global`，表示 IAM 只需要設定一次，這個設定就可以用來套用到使用者在全球所有 region 中的 resource


## 啟動 MFA

![IAM MFA options](/blog/images/aws/IAM_MFA-options.png)

- 建議啟動 MFA(Multi-Factor Authentication)，用來增加 root account 的安全性
> MFA 選項有三個，選擇 `Virtual MFA device`(Hardware 是要花錢買的) 可以與常見的 `Google Authenticator` or `Authy` app 搭配使用，透過掃描 barcode 的方式，手機上會一直出現 random 的啟用碼(要等一下)，輸入兩個就可以用來啟用 AWS IAM MFA 了

- root account 是一開始建立 AWS 所使用的帳號，擁有存取所有 AWS resource 的權限

- 使用 root account 建立其他擁有較小權限的帳號，並用其他帳號登入，會是相對較為安全的作法


## IAM users sign-in link

![IAM users sign-in link](http://etutorialsworld.com/wp-content/uploads/2016/05/72.22BAWS2BIAM2BDashboard-1.png)

1. 這是用來提供給其他使用者存取 AWS resource 之用，並非 root account，需要注意一下!

2. 網址是動態產生的，可以透過 **Customize** 的 link 設定別名以方便記憶

3. IAM user 的登入入口跟 root account 是不一樣的


## 新增 IAM Users

- 所有帳號在 AWS 的有效範圍都是 Global 的，沒辦法為特定的 Region 開啟帳號

- 建立 IAM user 時，可以根據需求建立；若是透過程式(API/SDK/CLI)存取 AWS，勾選 `Programmaric access`(需要 "**access key ID**" & "**secret access key**")；如果是要透過 AWS console 存取 AWS，勾選 `AWS Management Console access`(需要密碼)
![IAM User access types](/blog/images/aws/IAM_User-access-types.png)

- 在建立 Group 頁面中，指定權限的部份，若是看到有橘色方塊的項目，就表示此權限為 AWS 預先定義好的權限(AWS managed policy)；而 **Type** 為 `Job Function` 的部份，其實就是 AWS 預先為各種不同的管理職能，整理好的 AWS managed policy 的集合(減輕管理者設定權限上的負擔)，因此可將 Job Function 視為 AWS managed policy colleciton
![IAM Create Group - AWS Managed policy](/blog/images/aws/IAM_CreateGroup_AWS-managed-policy.png)

- policy 並非 group 專屬，也可以 attach 到單一 user

- 每個權限有其對應的 JSON 格式設定，若是未來要使用程式化的方式定義 IAM role 的權限，可以透過此方式很方便的取得正確的定義
![IAM Create Group - Permission JSON payload example](/blog/images/aws/IAM_CreateGroup_permission-json.png)


## 新增 Role

- 目前支援的 Role Type 有四種，分別是 `AWS services`, `Another AWS account`, `Web Identity`, `SAML 2.0 federation`，可能之後還會增加

- 以 `AWS service` type 為例，可用來指定 AWS 上面的特定 service 為 trusted entity(也可以視為用來存取其他 AWS resource 的來源端，例如：EC2)，並指定 trusted entity 可以擁有其他特定資源的存取權限
> 這可以設定的非常細，例如：只讓 EC2 對 S3 完全存取，無其他 service 的存取權限

其他的部份可以參考[官網文件(IAM Roles - AWS Identity and Access Management)](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)，有非常詳細的說明。


其他考試重點
==========

- 若是使用者權限給到 AdministratorAccess，卻想要限制該使用者權限只能在特定範圍內，可透過設定 `IAM permission boundary` 來達成，**但此功能只能套用在 IAM role/user 上，無法用在 IAM group 上**

-  **透過 Web Identity Federation，使用者可以透過相容於 OpenID Connect(OIDC) 的帳號提供者(Google, Facebook, Amazon ... etc)進行認證，取得臨時的 token 作為存取 AWS resource 之用**



Summary
=======

- IAM 有效範圍是 Global，目前還不支援 for 特定的 Region

- `root account` 是建立 AWS 帳號時所用的帳號，擁有存取所有 AWS resource 的權限

- 新建立的使用者預設是沒有任何權限的，都需要額外添加

- 要透過程式 or CLI 存取 AWS 的使用者必須要有 `Access Key ID` & `Secret Access Key` (只能看一次，因此產生的當下要妥善保存)

- root account 一定要啟用 MFA 以提高帳號安全性

- 可建立客製化的 password rotation policy


References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [AWS Identity and Access Management Documentation](https://docs.aws.amazon.com/iam/index.html)

- [IAM Roles - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)