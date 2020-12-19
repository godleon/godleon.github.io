---
layout: post
title:  "[AWS IAM] 學習重點節錄(1) - IAM Uer & Group、Credential"
description: "此篇文章是學習 Udemy 課程 \"AWS Identity Access Management (IAM) Practical Applications\" 時，將學習 IAM User & Group、Credential ...等內容的過程中整理出來的重要觀念 & 使用方式"
date: 2020-12-18 09:40:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - IAM
---


IAM(Identity & Access Management)
=================================

IAM 是什麼? 有以下幾個重點：

1. IAM 是 AWS 核心的服務

2. 用來控制一件事情 => **誰可以做什麼?**

3. 幾乎跟所有其他的 AWS service 都有深度的整合，作為其權限控管的機制

4. IAM 服務是屬於 global service，因此在 IAM 上建立的帳號，可以用在所有的 region 上

因此若是要善用 AWS 上的資源，IAM 是一個必須要熟悉的服務



IAM User
========

- 基本上 IAM User 會是一個使用者(or something)，被設定可用來存取 AWS resource

- 第一個建立的 account 稱為 `root user`，擁有最大的權限

- 建立帳號時，帳號是不分大小寫的，因此像是 `Bob` 與 `bob` 是相同的

- 同樣的帳號只能在一個 root account 中出現，不能有重複帳號

- 在 AWS 中做任何事情都要有權限的，而**權限來自於 policy**，即使要變更自己的帳號，也是要有 `IAMUserChangePassword` policy 權限的

- 一個 root account 可以建立的 user 數量上限是 5,000 (無法更多，AWS 不會提供)


## 在不同 root account 之間如何識別同名的使用者

![IAM User Account ID](/blog/images/aws/IAM_AccountID.png)

其中有一個比較重要的是，每一個 IAM User 要登入之前，需要知道三項資訊：

1. Username

2. Password

3. Account ID

因為 username 是可以隨意定的，不同的使用者可能會有相同的 username，而為了區分差別，此時就需要 `Account ID` 來識別了；而為了方便使用者登入，AWS 提供了 `https://[ACCOUNT_ID].signin.aws.amazon.com/console` 這樣的 url 並提供在管理網頁上(Security credentials 頁面中)


Credential
==========

要使用 AWS resource 需要 credential，用來驗證來訪者的身份；而 AWS 不會管你是哪位使用者，只要擁有合法有權限的 credential 就可以存取 AWS resource，因此其實妥善保護自己的 credential 不要被盜取是相當重要的。

此外，由於要服務各種不同的 entity 來使用 AWS resource，credential 大致上可分為以下幾種：

![IAM credential](/blog/images/aws/IAM_Credential.png)

1. `Email address + Password`：這是是給 root account 用的

2. `Account ID(Number) + User Name + Password`：這個是給一般的 user 用的

3. `Access Key + Secret Key`：這是給透過 API 存取 AWS resource 的時候使用的



真實使用者的類型
==============

現實世界中，有很多種不同的使用者類型，大致上可以分為以下七種：

1. Root User
2. Team Members
3. Federated Users
4. Application
5. External Users
6. Cross Account Users
7. Customers

以上每一種類型的使用者在某些場景下，可能都會有存取 AWS resource 的需求。

## Root User

基本上這就是最大權限的使用者了，會有以下幾種特性：

- 屬於該 account 的擁有者

- 每個 account 只能有一個 root user

- 有權限把整個 account 刪除

- 擁有所有的權限，而唯一可以限制 root user 的只有 `AWS Organization` 的機制

- Best practice => 不要使用 root user

## Team Members

![IAM team members](/blog/images/aws/IAM_team-member.png)

- 這是很常見的狀況，通常會在新創或小公司出現

- 需要存取 AWS resource 都用有各自的帳號，而這些帳號是透過 root account 開出來的

- 這類帳號的組合就是 `Account ID + User Name + Password`

## Federated Users

![IAM Federated Users](/blog/images/aws/IAM_federal-users.png)

- 適用在大型公司，擁有很多組織，並且可能有成百上千的使用者需要存取 AWS resource

- 由於一般大型公司都會有 AD 的存在，因此 IAM 也支援透過 AD 認證後，就可以進行 AWS resource 的存取，這樣的方式稱為 single sign on

## Application

![IAM Application](/blog/images/aws/IAM_application.png)

- 很多時候會有 Application 需要直接存取 AWS resource 的需求，此時使用的就不會是 EMail + Password

- 取而代之的是 `Access Key + Secret Key`，因為 AWS API 僅支援使用這類資訊作為 credential

## External Users

![IAM external users](/blog/images/aws/IAM_external-users.png)

- 存取 AWS resource 的人可能並非來自於組織內部，可能是外部的顧問、供應商、合作夥伴之類的

- 這些使用者(or 程式)通常是透過我們無法驗證的其他 identity provider 所驗證過

## Cross Account Users

![IAM cross account users](/blog/images/aws/IAM_cross-account-user.png.png)

- 這發生在其他 account 的 AWS user 想要存取你的 AWS resource 時，這就屬於 cross account user

- 兩個不同的 account 可能屬於同一家公司，也可能屬於不同公司

## Customers

- 這通常就是使用服務的用戶，可能透過手機 or 網頁連接到在 AWS 上的服務

- 可能也會透過 Facebook 作為 identity provider 來作為認證身份機制，來登入運行在 AWS 上的服務


Access Key
==========

- Access Key 會與某個 IAM user 有關連

- 每個 IAM user 最多可以有兩組 Access Key

- 通常 Access Key 是用來作為操作 AWS CLI 或是開發程式呼叫 AWS API 時使用

- 要正確的使用 AWS CLI 的功能，就需要把 Access Key 放到 `${HOME}/.aws/credentials` 中，格式如下：

```txt
[default]
aws_access_key_id = [YOUR_ACCESS_KEY_ID]
aws_secret_access_key = [YOUR_SECRET_ACCESS_KEY]
```

- `Access Key ID + Secret Access Key` 的組合在 AWS 是獨一無二的，因此可以用來識別特定使用者之用，並確認其有沒有足夠的權限存取 AWS resource


IAM Group
=========

其實 Group 概念就很清楚了，基本上就是將同性質的帳號可以群組起來，這樣對群組設定權限就會快多了，然而 IAM Group 有以下特性需要注意：

- 一個 root account 下最多只能設定 300 個 IAM Group；而每個 IAM User 最多只能加入 10 個 IAM Group 中 (這兩個限制無法請 AWS 提高)

- IAM Group 只有一層，因此表示 Group 不能在 Group 內，只有 User 可以在 Grouip 內

- IAM Group 沒有 credential



References
==========

- [AWS Identity Access Management (IAM) Practical Applications | Udemy](https://www.udemy.com/course/aws-identity-access-management-practical-applications/)