---
layout: post
title:  "AWS CSA Associate 學習筆記 - Advanced IAM"
description: "此篇文章是學習 AWS IAM 的進階相關概念時所留下的學習筆記，包含 Directory Service、IAM Policy、Resource Access Manager、Single Sign-On ... 等內容"
date: 2020-09-03 05:35:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - IAM
---


AWS Directory Service
=====================

## 什麼是 AWS Directory Service ?

其實這就是 AWS 提供的 AD(Active Directory) 的服務(技術的基礎為 `LDAP` + `DNS`)，使用 Microsoft 產品的管理人員幾乎都會用到的一個服務，以下是 AWS 對此服務的介紹：

> AWS Directory Service for Microsoft Active Directory, also known as AWS Managed Microsoft AD, enables your directory-aware workloads and AWS resources to use managed Active Directory in the AWS Cloud. AWS Managed Microsoft AD is built on actual Microsoft Active Directory and does not require you to synchronize or replicate data from your existing Active Directory to the cloud. You can use standard Active Directory administration tools and take advantage of built-in Active Directory features, such as Group Policy and single sign-on (SSO). With AWS Managed Microsoft AD, you can easily join Amazon EC2 and Amazon RDS for SQL Server instances to your domain, and use AWS Enterprise IT applications such as Amazon WorkSpaces with Active Directory users and groups.

可以歸納出以下重點：

- 其實這就是讓原本地端的 Microsoft Active Directory 功能可以延伸到 AWS cloud 的一個服務 (不需要將地端資料同步到雲端)

- AWS resource 的存取認證 & 權限可由在地端(on-premises) 的 AD 來設定 (可以使用現存於地端 AD 的 credential)

- 只要加入 AD domain 的 EC2 instance(甚至是 RDS for SQL Server 也行)，就支援 SSO (single sign-on)，強大的 Group Policy 也同時可以使用

- 可使用原本管理 AD 的工具進行管理，也可以使用 AWS 提供的工具(例如：[AWS WorkSpaces](https://aws.amazon.com/workspaces))


其他的重點觀念：

- 是一個獨立在 cloud 的 directory 服務

- AD 支援 Kerberos、LDAP、[NTLM(NT LAN Manager)](https://en.wikipedia.org/wiki/NT_LAN_Manager) 認證

- 服務本身有 HA 的設計

- AWS Directory Service 是由好幾個 managed service 所組合而成的一個 service family，包含：
  - AWS Managed Microsoft AD
  - Simple AD
  - AD Connector
  - [Cloud Directory](https://aws.amazon.com/cloud-directory)
  - [Cognito User Pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html)

- **上面五個服務中，前三個是 AD compatible，後面兩個則並非 AD compatible**


## AWS Managed Microsoft AD

![AWS Managed Microsoft AD](/blog/images/aws/Managed-Microsoft-AD.png)

- 提供運行在 windows server 上的 DC(domain controller)

- 為了確保服務的 HA，會提供兩個 DC，運行在不同的 AZ 上；若是要增加 HA 或是效能，可以增加更多的 DC

- 在同一個 VPC 中的 application 可以連到 DC

- 這些 DC 都完全屬於自己的，不會與其他 AWS 使用者共用

- 可以透過 [AD Trust](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_connect_existing_infrastructure.html) 連線到地端已經存在的 AD

### AD 管理權責劃分

AWS Directory Service 服務看似美好，但其實不少事情是使用者自己要完成的，把這些事情搞清楚，就不會覺得服務好像少了些什麼方便性之類的：

#### AWS 負責項目

- multiple-AZ deployment

- patch、monitor, recover

- instance rotation

- snapshot、restore

#### Customer 負責項目

- users, groups, GPOs 相關設定

- 慣用工具

- scale out DCs (自行根據需求決定)

- 是否要設定 AD Trust 功能

- 設定更安全的認證授權機制(例如：LDAPS)

- 設定 AD Federation


## Simple AD

- 是一個獨立、全託管的 directoy 服務

- 擁有基本的 AD 功能

- 支援兩種佈署規格，`Small` 適用於 500 人以下，`Large` 適用於 5000 人以下

- 可用來很方便管理 EC2 instance 的帳號 (for Windows workload)

- 若是需要 LDAP 服務的 Linux workload，也可以使用這個選項

- **不支援 AD Trust (表示無法讓 Simple AD 加入已經存在於地端的 AD，若有這需求需要使用 AD Connector)**


## AD Connector

- 當要利用已經存在於地端的 AD 資料時，`AD Connector` 是最佳選擇

- 以 `Directory Gateway(Proxy)` 的方式運行，銜接地端 AD 資料

- 可讓地端使用者使用原本的登入資訊登入 AWS

- 可將 EC2 instance 加入現有的 AD domain

- 若未來有成長的需求，可以 scale out 成多個 AD connector


## Cloud Directory

- 專門提供給開發者，一種以 directory 結構儲存資料的服務

- 可以使用多階層式的方式儲存數以億計的資料 (例如：組織結構圖、課程目錄、設備註冊資訊 .... 等等)

- 全託管服務

- 跟 Microsoft AD 沒任何關係


##  Cognito User Pools

- 是種全託管的 user directory，可用來與 SaaS 應用整合，提供帳號服務

- 可提供 web or mobile 應用註冊、登入 ... 等功能

- 可與其他社群服務(例如：FB)的帳號系統進行整合

- 跟 Microsoft AD 沒任何關係



IAM Policies
============

要了解 IAM policy 之前，首先要了解**在 AWS 中，是如何識別一種資源的**，答案是透過一個稱為 `ARN(Amazon Resource Names)` 的概念

## ARN (Amazon Resource Names)

![AWS ARN](/blog/images/aws/IAM_ARN.png)

首先要搞清楚 ARN 的命名格式如下：
> arn:partition:service:region:account_id

但上面那一段只有到 account level，如果要往下到 resource level，則必須再補上下面這一段：
- `resource`
- `resource_type/resource`
- `resource_type/resource/qualifier`
- `resource_type/resource:qualifier`
- `resource_type:resource`
- `resource_type:resource:qualifier`

透過上面 ARN 的命名方式，我們可以很精確的定位到特定的資源，以下是幾個例子：

- **arn:aws:iam::123456789012:user/mark**：由於 IAM 屬於 global 範圍，因此 region 欄位就保留空白

- **arn:aws:s3:::my_awesome_bucket/image.png**：每個 s3 object 都是全球唯一，無關 region & account，因此這兩個欄位保留空白

- **arn:aws:dynamodb:us-east-1:123456789012:table/order**：可以細到特定 table 層級

- **arn:aws:ec2:us-east-1:123456789012:instance/\***：也可以使用 `*` 來表示特定種類資源的所有項目


## 什麼是 IAM Policy?

IAM policy 是個用來定義 AWS 資源使用權限的 JSON 文件，有包含以下兩種類型：

- **Identity policy**：用來與 IAM user/group/role 綁定的權限設定

- **Resource policy**：用來與 AWS resource(例如：S3 bucket, SQS queue, KMS 加密用金鑰) 綁定的權限設定，會明確的定義出 **誰可以存取資源 & 可以存取的 API 有哪些**

需要注意的是，IAM policy 不是設定好就會有效的，必須要與帳號 or 資源進行綁定(attach)才會有效。

### IAM Policy 設定格式

![AWS policy format](/blog/images/aws/IAM_Policy_Foramt.png)

上圖展示了一個 IAM policy 設定的標準結構，大概有幾個重點：

- 會帶有 `Version` 的定義，用來識別這個文件的結構，例如：`"version": "2012-10-17"`

- 由一連串的 `statement` 定義所組成，每個 statement 會用大括號(`{}`)包起來

- 每個 statement 都會符合一組 AWS API request 的定義，指定可以對 AWS resource 執行哪些動作

以下是一個實際範例：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "SpecificTable",
            "Effect": "Allow",
            "Action": [
                "dynamodb:BatchGet*",
                "dynamodb:DescribeStream",
                "dynamodb:DescribeTable",
                "dynamodb:Get*",
                "dynamodb:Query",
                "dynamodb:Scan",
                "dynamodb:BatchWrite*",
                "dynamodb:CreateTable",
                "dynamodb:Delete*",
                "dynamodb:Update*",
                "dynamodb:PutItem"
            ],
            "Resource": "arn:aws:dynamodb:*:*:table/MyTable"
        }
    ]
}
```

有幾項重點需要知道：

- `Effect` 可以是 **Allow** or **Deny**

- API 存取的限制範圍就是定義在 `Action` 中，可以用 wildcard(`*`) 來表示具有同樣 prefix 的 API

- `Resource` 欄位則是指定這個 statement 會對哪個 ARN 生效


## Permission Boundry

Permission Boundry 是用來限制使用者存取權限的範圍在那，舉例來說，假設希望達成以下目的：

- 管理者賦予 Leon `AdministratorAccess` 的權限

- 但卻希望這個權限僅對 DynamoDB 有效

透過設定 Permission Boundry，加入 `AmazonDynamoDBFullAccess`，就可以限定 Leon 的權限範圍只會在 DynamoDB 中，而不會有管理者的完整權限。

> 需要注意的是，Permission Boundry 並不是拿來設定特定 role 允許或拒絕存取特定 resource 之用


## 實作注意事項

- policy type 有 `AWS managed`(前面會有一個橘色盒子圖案，是 AWS 預先設定好一些常用的 policy 方便使用者直接取用) & `Customer managed` 兩種

- 在 statement 中，不同的 API 會對應到不同的 ARN 設定，例如：`s3:ListBucket` 就會用在 bucket level ARN；`s3:GetObject` 就會用在 object level ARN

- 要將 policy 跟某個 entity(例如：Role) 綁定才會有效；還需要將這樣的綁定組合指定給特定的 AWS resource(例如：EC2 instance)

- 透過 **inline policy**，可以讓 policy 只對特定 role 有效；無法用在其他的 role 上


## 考試重點

- IAM policy 中，對任何 resoure 的存取預設都是拒絕，若要開放，就必須明確設定

- **如果明確的設定拒絕存取，那就是完全拒絕存取，優先權大於任何開放存取的相關權限，即使有綁定開放權限也沒用**

- policy 要生效，就需要做好正確的綁定(with user/group/role)

- 有多個 policy 同時與特定的 entity 或是 resource 綁定時，結果會是 join 後的效果



Resource Access Manager (RAM)
=============================

- 在 AWS multiple account 的管理中，一般會將管理、帳務、功能性 .... 等帳號根據不同職能拆分開，但若是有多個帳號需要共享 resource 的存取權限時，就正好是 RAM 可以協助的地方

- 不論是獨立的帳號，或是 organization，都可以透過 RAM 的設定，在不同的帳號間設定資源共享
> 試想看看如果你要將同樣的 resource 權限設定給 100 個不同的帳號，沒有個簡單的方式可能會很辛苦阿.....

- **並非所有 resource 的權限都可以透過 RAM 在不同的帳號間共享**，目前支援的有 `App Mesh`、`Aurora`、`CodeBuild`、`EC2`、`EC2 Image Builder`、`License Manager`、`Resource Group`、`Route 53`


## 實作重點

- 當 account1 分享資源給 account2 之後，account2 需要到 RAM 管理畫面中 approve 來自於 account1 的分享，才可以在 console 畫面中看到被分享出來的資源

![AWS RAM share resource between accounts](/blog/images/aws/RAM_share-resource-between-accounts.png)

- 範例中 account1 將 private subnet 分享給 account2

- account2 可以在 private subnet 中建立 EC2 instance，藉此取得與內部其他資源的聯繫

- account2 無法修改 account1 分享出來的 subnet，只能加 tag



AWS Single Sign-On
==================

![AWS SSO SAML integration](/blog/images/aws/SSO_SAML-integration.png)

- AWS SSO 可以整合很多外部服務的現存帳號，提供使用者 SSO 的功能

- 成功連結到 AWS SSO 的帳號，就可以給定不同資源的存取權限

- 只要是支援 SAML 2.0 的協定，都可以與 AWS SSO 進行整合來提供帳號認證的服務



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Amazon Cloud Directory | Amazon Web Services (AWS)](https://aws.amazon.com/cloud-directory)

- [Amazon Cognito User Pools - Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html)

- [AWS Identity and Access Management Documentation](https://docs.aws.amazon.com/iam/index.html)

- [AWS Resource Access Manager](https://aws.amazon.com/ram/)

- [AWS Single Sign-On | Cloud SSO Service | AWS](https://aws.amazon.com/single-sign-on/)