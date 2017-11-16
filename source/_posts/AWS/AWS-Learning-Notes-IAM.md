---
layout: post
title:  "AWS 學習筆記 - IAM(Identity and Access Management)"
description: "This article is a memo recorded when learning AWS IAM"
date: 2017-05-03 04:00:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - IAM
---


What is IAM?
============

AWS Identity and Access Management (IAM) enables you to securely control access to AWS services and resources for your users. Using IAM, you can create and manage AWS users and groups, and use permissions to allow and deny their access to AWS resources. 

IAM is a feature of your AWS account offered at no additional charge. You will be charged only for use of other AWS services by your users.



What does IAM give you?
=========

- Centralized control of your AWS account

- Shared Access to your AWS account
> 可以分享權限到其他帳號去

- Granular Permissions
> 可以針對每個帳號可存取的資源權限進行很細部的控制，例如**限制某人只能對 DynamoDB 進行唯讀的存取**

- Identity Federation(including Active Directory, Facebook, Linkedin etc)
> 可以透過其他服務(AD, Facebook, Linkedin etc)的帳號透過 SSO 登入 AWS

- Multifactor Authentication
> AWS 建議為每個帳號都設定 multifactor authentication

- Provide temporary access for users/devices and services where necessary

- Allows you to set up your own password rotation policy

- Integrates with many different AWS services

- Supports PCI DSS(Payment Card Industry Data Security Standards) Compliance
> PCI DSS(支付卡產業資料安全標準)是在整合外部付費服務之用，為了提升線上支付的安全性



Critical Terms
==============

- **Users** - End Users(think people)

- **Group** - A collection of userss under one set of permissions.
> A way to group our users and apply policies to them collectively

- **Roles** - You create roles and can then assign them to AWS resources

- **Policies** - A document that defined one(or more permission)

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


學習筆記
======

## IAM is global(universal)

從 management console 進入 IAM 的功能頁面後，Region 的部份會變成 `global`，表示 IAM 只需要設定一次，這個設定就可以用來套用到使用者在全球所有 region 中的 resource


### IAM users sign-in link

![IAM users sign-in link](http://etutorialsworld.com/wp-content/uploads/2016/05/72.22BAWS2BIAM2BDashboard-1.png)

1. 這是用來提供給其他使用者存取 AWS resource 之用，並非 root account，需要注意一下!

2. 網址是動態產生的，可以透過 **Customize** 的 link 設定別名以方便記憶


## Security Status

### (1) Delete your root access keys

AWS 建議儘量不要用 root account 進行資源的存取；正確的作法應該新增使用者，並為使用者設定所需要的資源存取權限。

例如：一開始我在 root account 加了一把 access key，在這個部份就無法 pass 檢查了

### (2) Activate MFA on your root account

選擇 `A virtual MFA device` 作為 MFA device type(Hardware 是要花錢買的)，接著可以選用 `Google Authenticator` 作為接收驗證碼之用。

在 Google Authenticator 中可以設定很多個要用來作 MFA 的帳號，當然也就可以把 AWS IAM 設定進來。

透過掃描 barcode 的方式，手機上會一直出現 random 的啟用碼(要等一下)，輸入兩個就可以用來啟用 AWS IAM MFA 了。

### (3) Create individual IAM users

在建立 user 時，有幾點需要注意一下：

- **Access type**：勾選 `Programmatic access` 才可以用這個帳號搭配 AWS API, CLI, SDK .... 等其他開發工具來存取 AWS 的資源；勾選 `AWS Management Console access` 才可以透過密碼登入的方式進入 AWS Management console

- 可在此時順便建立 or 指定特定的 group，也可以順便指定 policy 來設定權限
> policy 並非 group 專屬，也可以 attach 到單一 user

- 建立完成的 user(若有勾選 **Programmatic access**) 會得到 `Access key ID` & `Secret access key`，這是開發用來存取 AWS 資源的程式需要的資訊


### (4) Use groups to assign permissions

目前 AWS 已經提供了很多內建的權限清單可以用，例如：S3 唯讀, Glacier 唯讀....等等，但目前還不確定能不能自訂權限的選項。。

### (5) Apply an IAM password policy

這沒什麼特別，就是設定密碼的規則....(長度, rotation period...等等)


## Add Role

### (1) Role Type

- **AWS Service Role**: 用來指定 AWS 上面的特定 service
> 這可以設定的非常細，例如：只讓 EC2 對 S3 完全存取，無其他 service 的存取權限

- **Role for Cross-Account Access**: 可以用來讓特定帳號去存取其他帳號的 management console

- **Role for Identity Provider Access**: 作 SSO, 整合 FB, Linkedin 帳號時才會用到的部份

### (2) Attach Policy



Summary
=======

- IAM is universal. It does not apply to regions at this time.

- The **root account** is simplely the account created when first setup your AWS account. It has complete Admin access.

- New Users have **NO** permissions when first created.

- New Users are assigned **Access Key ID** & **Secrect Access Keys** when first created.

- These are not the same as a password, and you cannot use the Access Key ID & Secret Access Key to Login in to the console. You can use this to access AWS via the APIs, SDK and Command Line however.

- You only get to view these once. If you lose them, you have to regenerate them. So save them in a secure location.

- Always setup Multifactor Authentication on your root account.

- You can create and customize your own password rotation policies.