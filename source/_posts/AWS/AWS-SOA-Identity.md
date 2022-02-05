---
layout: post
title:  "AWS SOA 學習筆記 - Identity"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 Identity 相關服務(Cognito、SSO)在實際操作維運上需要注意的事項"
date: 2022-01-23 17:25:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - SOA
  - IAM
  - Cognito
  - SSO
---

IAM Security Tools
==================

IAM 提供了兩個方便檢視使用者權限範圍 & 服務使用狀況的工具，分別是：

1. IAM Credential Report (account-level)

2. IAM Access Advisor (user-level)

其中 IAM Credential Report 屬於 AWS Account level 的資訊(所以應該一般是由 root user 來使用)，會匯出一個 csv 每一個 IAM user 的帳號使用狀況，可以用來協助評估是否已經是一個不需要使用 or 安全設定沒有設定好(例如：MFA 沒開啟)的帳號

而 IAM Access Advisor 屬於 user-level 的資訊，會顯示用來存取每個服務的權限，以及最後存取服務的時間，方便管理者用來進行最小權限原則部份的規劃。


IAM Access Analyzer
===================

![AWS IAM Access Analyzer](/blog/images/aws/Identity/IAM-Access-Analyzer.png)

- IAM Access Analyzer 可以協助你找出有哪些服務(例如：S3 Bucket、IAM Roles、KMS Key、Lambda Function、SQS)提供給超出預期的範圍進行存取

- 透過設定 **Zone of Trust**，可以指定服務應該限定在什麼範圍內存取，超過範圍的應該是不合法的(例如上圖中的 Account & External Client)

- 分析完後，超過 **Zone of Trust** 的存取範圍，會被判定為 `findings`，這時候有兩個處理方式：
  1. 修正服務存取權限
  2. 將其設定為 archive(表示忽略)，讓其不會再出現在 finds 中 (另外還可以設定 archive rule 來針對符合特定條件的 findings 自動 archive)


Identity Federation
===================

## Overview

![AWS Identity Federation](/blog/images/aws/Identity/Identify-Federation.png)

Identity Federation 是個宏觀的概念,以上圖來說明，重點如下：

- 這個功能目的就是搭配外部的 Identity Provider，讓外部的使用者可以 assume role(暫時) 來存取 AWS 資源

- 使用者僅可以 assume 成特定的 role

- 可與外部多種 identity provider 整合，例如：
  - LDAP
  - Microsoft Active Directory (其實就是 via SAML)
  - Single Sign On
  - OpenID
  - Cognito

- 不需要新增額外的 IAM User，使用者帳號在外部管理

## SAML Federation for Enterprise

- 這主要目的是要讓使用者可以透過內部的 Microsoft AD or LDAP server(或是任何符合 SAML 2.0 標準的 Identity Provider) 進行認證後，並可暫時的存取 AWS service API or management console/CLI

- 若是存取 AWS service API，那 client 會直接與 STS(Security Token Service) 服務連接，取得臨時的 credential (如下圖)

![AWS SAML-based Federation for API access](/blog/images/aws/Identity/SAML-based-federation-for-api-access.png)

- 若是存取 AWS console or CLI，那 client 則會需要與 AWS SSO endpoint 通訊，由 SSO 服務與 STS 連接，取得臨時的 token (如下圖)

![AWS SAML-based Federation for Management Console/CLI access](/blog/images/aws/Identity/SAML-based-federation-for-management-console-access.png)

- 若是認證機制不相容 SAML 2.0，還是可以串接，只是對 STS 串接的部份就需要自行開發 (如下圖)

![AWS SAML-based Federation for Custom Identity Broker](/blog/images/aws/Identity/SAML-based-custom-identity-broker.png)

- 不論如何串接，優點都是不需要再建立 IAM User，使用者帳號由外部服務管理

## Cognito - Federation Identity Pools for Public Applications

![AWS Cognito - Federation Identity Pools for Public Applications](/blog/images/aws/Identity/Cognito-Federation-Identity-for-Public-Applications.png)

- Cognito 的目標是要讓 Application User(client 數量更多) 可以直接存取 AWS resource(例如：將 Facebook user access log 寫入 S3 bucket)

- 整個身份驗證流程如下：
  1. 登入到提供身份證認的服務(例如：Google、Facebook、Twitter、OpenID、SAML .... 等等)
  2. 從 Federated Identity Pool(這裡指的就是 Cognito) 取得臨時的 AWS credential 
  3. 透過 臨時的 AWS credential 來存取 AWS resource (搭配預先設定好的 IAM policy 限定可存取範圍)

- Web Identity Federation 可以達到與 Cognito 相同效果，但此服務已經被 Cognito 取代


STS (Security Token Service)
============================

## Overview

- STS 是串接外部認證服務與 AWS 內部權限的關鍵，用來提供有限度且暫時的權限來存取 AWS resource

- STS 核發的 token 有效期限最長是一個小時

- 提供了幾個主要的 API，分別是 `AssumeRole`、`AssumeRoleWithSAML`、`AssumeRoleWithWebIdentity`、`GetSessionToken`

- **`AssumeRole`** 可發生在相同的 AWS account，也可以 cross AWS account，但要有對應的 Allow IAM policy 才行

- **`AssumeRoleWithSAML`** 回傳 credential 給經過 SAML 認證的使用者

- **`AssumeRoleWithWebIdentity`** 回傳 credential 給其他的 IdP(例如：Google、FB、OIDC provider .... 等等)，AWS 建議這個搭配 Cognito 來設定

- **`GetSessionToken`** 用來給 AWS IAM User & 其他 AWS account 使用，目的就是要取得臨時的 credential 來獲取臨時存取 AWS resource 的權限

## Assume Role

![AWS STS - Assume Role](/blog/images/aws/Identity/STS-Assume-Role.png)

Assume Role 的重點在於：

- 需要在 account or cross account 中先定義一個 IAM Role

- 接著需要定義哪些 principal 可以 assume 成這個 IAM role

- 對 STS 呼叫 AssumeRole API(如上圖) 來切換 IAM role 取得權限，流程如下：
  1. 使用者對 STS 呼叫 AssumeRole API (帶著 identity provider 給的 token/credential)
  2. STS 呼叫 IAM 服務，檢查是否有權限執行 AssumeRole API
  3. 檢查通過，STS 回傳一個臨時的 credential 給使用者
  4. 使用者拿到 credential 來切換成 IAM role 取得存取 AWS resource 權限

## Cross Account Assume Role

![AWS STS - Cross Account Assume Role](/blog/images/aws/Identity/STS-Cross-Account-Assume-Role.png)

上圖是個 cross account assume role 的設定情境：

1. 管理者在 production account 中建立一個可以用來存取 production bucket 的 IAM role `UpdateApp`

2. 管理者在設定允許 Development Account 中在 Developer Group 的使用者，可以 assume 成 production account 中的 `UpdateApp` IAM role
> 在 Developer Account 中的 Developer Group 在這個案例中就是上面說的 `Principal`

3. developer 呼叫 assume role API

4. STS 服務提供暫時性的 credential 並回傳

5. developer 成功 assume role 後並更新資料到 produciton bucket


Cognito User Pools
==================

## Overview

Cognito User Pools 服務本身有以下幾個特性 & 功能：

- 可做為 web or mobile apps 的 user database，可啟用 MFA

- 提供簡單的登入機制(透過 username & password) & 重置密碼的功能

- 提供 EMail & 電話認證

- 可與外部的 Identity Provider(例如：FB、Google、SAML) 整合作為身份認證來源

- 使用者登入後，會回傳 JWT(JSON Web Token)

> **需要注意的是，Cognito User Pools 是用來作為 Application User Database，而非用來存取 AWS resource 用**


## 與外部 Identity Provider 整合

下圖介紹了與外部的 Identity Provider 整合時的架構：

![AWS Cognito User Pools](/blog/images/aws/Identity/Cognito-User-Pools.png)

以下簡單說明一下登入的流程：

1. Web/Mobile app 透過 Cognito User Pools 執行 login 操作

2. Cognito User Pools 向內部的 Database or 外部的 Identity Provider 進行認證

3. 認證成功，Cognito User Pools 會回傳一個 JWT 給 Web/Mobile app 作為後續使用


## 與內部服務整合

下圖介紹與 API Gateway & Application Load Balancer 整合時的架構：

![AWS Cognito User Pools integrations](/blog/images/aws/Identity/Cognito-User-Pools_integrations.png)

在 API Gateway 的部份，使用者可以在 API call 中加入從 Cognito User Pools 拿到的 JWT，API Gateway 則會向 Cognito User Pools 驗證 token 的有效性，進而決定要不要往後方的 backend service 送。

而 Application Load Balancer 的部份，則是透過 OIDC 的方式，將 Cognito User Pools 作為 OIDC provider 來進行身份認證，認證完後就會將 request 往後送到 backend service。


Cognito Identity Pools
======================

## Overview

- 首先要弄清楚 Cognito Identity Pools 服務的目的，是要**協助使用者取得存取 AWS resource 用的臨時 credential**

- 可包含多個 Identity Source 並與其整合，例如：
  - public provider：例如：Amazon、Facebook、Google、Apple ... 等等
  - **存在於 Cognito User Pool 中的 user**
  - OpenID Connect Provider
  - SAML Identity Provider
  - Custom Login Service

- 同時提供 unauthenticated access (guest) 的機制

- 通過認證的使用者可以直接存取 AWS service 或是透過 API Gateway(有與 Cognito Identity Pool 整合)
  - 可透過 user_id 進行細部權限控管
  - 需要在 IAM policy 中以 policy variable 的方式讓透過 cognito identity pool 傳過來的 credential 取得對應權限


## 單獨使用 Cognito Identity Pools 服務

既然 Cognito Identity Pools 可以與眾多 IdP 串接，當單獨使用此服務時，整體認證的架構就會如下圖：

![AWS Cognito Identity Pools](/blog/images/aws/Identity/Cognito-Identity-Pools.png)

認證流程如下：

1. Web/Mobile Application 會先登入 IdP 的服務

2. 向 Cognito Identity Pools 傳送 IdP 提供的 token 並要求臨時的 AWS credential

3. Cognito Identity Pools 會拿 token 並向 IdP 驗證 token 是否合法

4. token 合法，就會向 STS 服務取得一個臨時的 AWS credential，並回傳給 Web/Mobile Application

5. Web/Mobile Application 取得權限存取 AWS resource


## 搭配 CUP(Cognito User Pool)

![AWS Cognito Identity Pools with CUP](/blog/images/aws/Identity/Cognito-Identity-Pools_with-CUP.png)

另外一種模式則是搭配 CUP(Cognito User Pool) 服務，讓所有 user data 統一存在 CUP；在此種模式下，只要向 CUP(Cognito User Pool) 驗證使用者身份即可。

## IAM Role

關於搭配 Cognito Identity Pools 服務時，如何撰寫 IAM policy 讓使用者可以正確的 assume role，可以參考以下 AWS 的官方文件：

- [IAM roles - Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html)


Single Sign-On (SSO)
====================

## Overview

- SSO 服務的目的是要讓多個 AWS account & 其他外部服務(例如：DropBox、Office365、Slack .... 等等)等使用者可以同一個入口登入後，就可以存取所有串接好的服務

- 可與 AWS Organization 進行整合

- 支援 SAML 2.0

- 可與 on-premise Active Directory 進行整合

- 可以對權限進行集中管理

- 可以搭配 CloudTrail 作到集中稽核的目的

以下就是個範例示意圖：

![AWS Cognito Identity Pools with CUP](/blog/images/aws/Identity/SSO-with-AD.png)

## SSO v.s. AssumeRoleWithSAML

![AWS Cognito Identity Pools with CUP](/blog/images/aws/Identity/SSO-vs-AssumeRoleWithSAML.png)

上圖是 SSO 與 AssumeRoleWithSAML 的差異，重點是：

1. 使用者是從外部服務登入(AssumeRoleWithSAML) or AWS SSO Login Portal(SSO) 登入

2. 使用 AssumeRoleWithSAML 需要向 STS 換取一個臨時的 credential 存取 AWS resource；但透過 SSO 登入後，就會有合法的 credential 可以存取 AWS resource


References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)

- [About SAML 2.0-based federation - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_saml.html)

- [Enabling SAML 2.0 federated users to access the AWS Management Console - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-saml.html)

- [Providing access to externally authenticated users (identity federation) - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios_federated-users.html)

- [IAM roles - Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/iam-roles.html)