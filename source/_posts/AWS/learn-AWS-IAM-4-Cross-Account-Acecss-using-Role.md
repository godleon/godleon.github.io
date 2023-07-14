---
layout: post
title:  "[AWS IAM] 學習重點節錄(4) - Cross Account Access by IAM Role"
description: "此篇文章是學習 Udemy 課程 \"AWS Identity Access Management (IAM) Practical Applications\" 時所記錄下來的重點，主要說明如何透過 IAM Role 的機制，達成跨帳號(Cross Account)存取 AWS 資源的目的"
date: 2020-12-24 06:00:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - IAM
---


前言
====

前面討論的部份都屬於在同一個 root account 下，要如何進行權限的管理 & 設定。

但是在某些使用場景下，要存取 AWS resource 來源並非在 root account 下，而是另外一個帳號：

![IAM Cross Account - Case 1](/blog/images/aws/IAM/IAM-Cross-Account-Case-1-1.png)
> 來自於另外一個帳號的 IAM User Jenny，要存取我們的 S3 bucket

也可能是外部的 SAAS 服務：(例如：New Relic, Data Dog ... etc)

![IAM Cross Account - Case 2](/blog/images/aws/IAM/IAM-Cross-Account-Case-1-2.png)

這時候，透過 IAM Role 來做管理設定會是個相對方便 & 有效的方法，若是透過給予 Access Key 的方式，後續可能會造成很多問題。



Access/Secret Key 有什麼問題?
===========================

若是使用提供 Access Key & Secret Key 的方式，會有以下幾個隱憂：

- 你沒辦法確保 access key 提供出去後，對方不會把 Key 再提供給別人

- 若是對方傻呼呼的將 Key 送到 public Git repository，那全世界都會知道你的 access key

- 若因為安全規範，需要 rotate access key，這件事情會相當難完成，這樣每一個曾經給過 access key 的對象又要再來一遍

- 如果對方帳號中需要存取我們 AWS resource 的人因為某些原因，時常變更，或是來來去去，這樣我們就必須要常常調整 IAM User 的設定，並給出新的 access，這對管理上來說會是個惡夢

綜合以上問題，為了讓權限管理這件事情的複雜度可以收斂在一定的範圍內，改用 IAM Role 來管理會是較好的方式。



Cross Account Role for IAM User 設定範例
=======================================

## 使用場景

在[上一篇文章](learn-AWS-IAM-3-assuming-role.md)中所提到的是在同一個帳號下的 IAM Role，那若是 Cross Account Role 是什麼樣一個場景呢? 以下舉個範例：  

![IAM Cross Account -  purpose](/blog/images/aws/IAM/IAM-Cross-Account-access-resources.png)

在這邊要達成的目的是：讓帳號 ACME 的 IAM User Troy，可以存取帳號 Stuzio 底下的 S3 bucket

## 設計原理 & 規範

了解了上面的跨帳號的資源存取需求後，那接下來要怎麼完成這件事情呢? 假設有 A & B 兩個帳號，而 B 帳號的 IAM User 想要存取 A 帳號的 S3 Bucket，那會需要進行以下幾個步驟：

1. 在 A 帳號中設定一個 IAM Role，並設定 Trust RelationShip 為 B 帳號，允許其 Assume Role

2. 在 A 帳號中設定 IAM Policy，將 S3 Bucket 存取權限社繫結到 IAM Role 上

3. 在 B 帳號中設定 IAM Policy，允許 Assume Role 成 A 帳號的 IAM Role，並繫結到 B 帳號的 IAM User 上


### 設定 IAM Role & 設定 Trust Relationships

首先，**必須先設定一個 IAM Role，設定 Trust Relationships，信任來自於另外一個帳號**，要完成這件事情，首先要建立一個 IAM Role(按照上圖，名稱為 `StuzioAcmeConsultantRole`)，並設定 Trust Relationships 內容如下：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            //表示信任來自於 AWS root account "arn:aws:iam::183438944504:root"
            //底下所有的 IAM User 送過來的 request
            "Principal": {
                "AWS": "arn:aws:iam::183438944504:root"
            }, 
            //允許執行 "sts:AssumeRole" 的操作
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

上面這個設定，**表示信任 & 允許另外一個帳號(Account ID = 183438944504)底下的 IAM User，透過 Assuming Role 的方式，切換成此帳號(Account ID = 081278057606)的 IAM Role `StuzioAcmeConsultantRole`**。

### 設定 IAM Policy

接著為了可以存取 S3 Bucket，那就要設定正確的 IAM Policy，以下是設定內容：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "FullAccessToProjectX",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::com.stuzio.project-x",
                "arn:aws:s3:::com.stuzio.project-x/*"
            ]
        },
        {
            "Sid": "ListAllBuckets",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        }
    ]
}
```

![IAM Policy](/blog/images/aws/IAM/IAM-Cross-Account-role-and-policy.png)

為這個 Policy 取個名字(如上圖的 `StuzioProjectXPolicy`)，然後再將此 policy attach 到上一個步驟建立的 IAM Role(`StuzioAcmeConsultantRole`) 上，那 IAM Role 就會有權限存取 S3 Bucket 了。


### 允許 Assuming Role

完成了上面的步驟，目前 A 帳號的 IAM Role 已經有權限可以存取 A 帳號的 S3 Bucket，接下來的問題是：**如何讓 B 帳號的 IAM User 可以切換成 A 帳號的 IAM Role ?**


設定流程如下圖：

1. 建立 IAM Policy，提供 Allow "sts:AssumeRole" 的權限，對象(Resource)是 A 帳號的 IAM Role

2. 將 IAM Policy 繫結到 B 帳號的 IAM User

![IAM Policy - Assuming Role](/blog/images/aws/IAM/IAM-Cross-Account-assume-role.png)

IAM Policy 的內容如下：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "StuzioClientAssumeRolePolicy",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::081278057606:role/StuzioAcmeConsultantRole"
        }
    ]
}
```

完成後，將此 IAM Policy 繫結到 Troy 身上即可囉!



Cross Account Role for SAAS Service 設定範例
===========================================

假設需要存取我們 AWS resource 的對象是外部的 SAAS 服務(以 DataDog 為例)時，那應該怎麼設定呢?

其實概念上跟另外一個 AWS 帳號是差不多的，其實 SAAS 服務也會提供它們自己的 AWS Account ID，此外，還會包含一個 External ID 作為進一步的識別之用，因此 SAAS(以 DataDog 為例)，會提供兩個資訊：

- AWS Account ID

- External ID

因此，我們這邊要做的就是新增一個 IAM Role，設定 Trust Relationships 為 SAAS 提供的 IAM Account & External ID，

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::464622535012:root"
            }, 
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "[PUT EXTERNAL ID HERE]"
                }
            }
        }
    ]
}
```

結果如下圖所示：

![IAM Cross Account - SAAS 1](/blog/images/aws/IAM/IAM-Cross-Account-Case-SAAS-1.png)


接著就是設定 IAM Policy，給入 SAAS 所需要的權限就可以了!

至於 SAAS 服務端需要建立用來 Assuming Role 的 IAM Policy，SAAS 服務會自己處理，就不需要我們擔心了，最後會變成以下這樣：

![IAM Cross Account - SAAS 2](/blog/images/aws/IAM/IAM-Cross-Account-Case-SAAS-2.png)

SAAS 服務中的 AWS IAM User 就可以切換成成我們提供的 IAM Role 來存取其所需要的資源囉!


然而，若是我們有多個環境，每個環境屬於不同的 AWS Account 時，那要讓 SAAS 服務都可以接入，也變得簡單許多，只要重複上面設定 IAM Role + Trust Relationships & IAM Policy 即可，那這樣多環境的情境就換變成下面這樣：

![IAM Cross Account - SAAS 3](/blog/images/aws/IAM/IAM-Cross-Account-Case-SAAS-3.png)



為什麼要用 IAM Role?
===================

為什麼相較於使用 IAM User + Group，改成使用 IAM Role 會是比較好的管理策略呢? 

以下舉幾個理由來說明：

![IAM - Why IAM Role? - 1](/blog/images/aws/IAM/IAM-Why-IAM-Role-1.png)

- 若是另外一個帳號的管理者希望改由另外一個 IAM User 來存取我們的 AWS resource，只要該管理者自己將 AssumeRole 的 IAM policy 綁到新的 IAM User 身上即可，我們可以完全不用調整

![IAM - Why IAM Role? - 2](/blog/images/aws/IAM/IAM-Why-IAM-Role-2.png)

- 可以不用給出 Access/Secret Key，因為無法確保對方會不會安全的存放我們提供出去的 Access/Secret Key

![IAM - Why IAM Role? - 3](/blog/images/aws/IAM/IAM-Why-IAM-Role-3.png)

- 若是需要存取 AWS resource 的 entity 數量很大，這樣的管理負擔會很大，且很沒彈性

![IAM - Why IAM Role? - 4](/blog/images/aws/IAM/IAM-Why-IAM-Role-4.png)

- 若是有多個環境，那管理的複雜度又會再提昇一個量級上去

![IAM - Why IAM Role? - 5](/blog/images/aws/IAM/IAM-Why-IAM-Role-5.png)

- 若是外部的 entity 同時要存取我們多個環境中的資源，使用 IAM User & Group，就表示要管理多組的 Access/Secret Key，同時也造成了對方的不便

![IAM - Why IAM Role? - 6](/blog/images/aws/IAM/IAM-Why-IAM-Role-6.png)

- 不需要為每個需要來存取 AWS resource 的 entity 建立 IAM User 了，只要設定 IAM Role 並指定 Trust Relationship 即可



References
==========

- [AWS Identity Access Management (IAM) Practical Applications | Udemy](https://www.udemy.com/course/aws-identity-access-management-practical-applications/)

- [Providing access to an IAM user in another AWS account that you own - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios_aws-accounts.html)

- [Providing access to AWS accounts owned by third parties - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios_third-party.html)