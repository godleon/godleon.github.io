---
layout: post
title:  "[AWS IAM] 學習重點節錄(5) - IAM Role Federated Access"
description: "此篇文章是學習 Udemy 課程 \"AWS Identity Access Management (IAM) Practical Applications\" 時所記錄下來的重點，主要說明如何透過 IAM Role Federated Access，讓外部的認證機制可以與 AWS IAM 串接，取得使用 AWS resource 的權限"
date: 2020-12-25 06:00:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - IAM
---


概觀
====

前面提到的 IAM Role 可以透過 Assuming Role 的方式，讓同一個帳號中的 IAM User，甚至是 cross account 的 IAM User，都可以透過 Assuming Role 的方式來取得對應的 IAM Role 的權限。

其實 Federated Access 也是相同的概念，差別在於，前面提到的部份都是在 AWS 服務的範圍內(即使是 cross account，甚至某些 SAAS 服務)，而 Federated Access 則是來自於外部的服務，例如：Facabook。

想像一個狀況，假設公司開發了一個 Mobile App，而這個 App 需要存取到特定的 AWS resource，有沒有安全又方便的方式可以達成呢?

這個需求就是需要透過 Federated Access 來完成了，大方向的概念大概是這樣的：

1. 我們允許來自 Facebook 的使用者存取特定的 AWS resource，透過 Federated Account 的方式確認是來自 Facebook 的特定 App 

2. **使用者認證**這件事情由 Facebook 負責處理

如此一來，使用者就可以透過 Facebook 認證後，就可以透過 Assuming Role 的方式來存取我們的 AWS resource 了。



使用 Access/Secret Key 有什麼問題?
================================

若是將 Access/Secret Key 放入 App 中，作為存取 AWS resource 的認證方式，將會是下面這個模樣：

![IAM - App Access by Access Key](/blog/images/aws/IAM/IAM-App-Access-By-Key.png)

隨著 App 下載數量越來越多，Access Key 就會被到處散佈，那可能會造成以下兩個隱憂：

1. 隨著應用程式越來越多人安裝，那就表示這把 access key 存在於越來越多人的手機上，安全性堪慮，且這把 access key 就會變成完全無法隨意更動的狀態

2. 若是有惡意的駭客從手機中取出這把 access key，他也等同擁有了對應的 IAM policy 的權限

所以使用 Federated Access 是個相對彈性 & 安全的方式



Federated Access 有那幾種?
=========================

Federation Access Type 有兩種：

- Web Identity Federation

![IAM Federation Access Type - Web Identity Federation](/blog/images/aws/IAM/IAM-Web-Identity-Federation.png)

基本上就像是透過 FB、Google .... 認證後，拿著 FB or Google 的臨時憑證來 Assuming Role，藉此取得權限。

- Enterprise Identity Federation

![IAM Federation Access Type - Enterprise Identity Federation](/blog/images/aws/IAM/IAM-Enterprise-Identity-Federation.png)

這則是透過像是 Microsoft Active Directory 的方式，使用者透過 AD 認證後，拿著 AD 的臨時憑證來 Assuming Role，藉此取得權限。

而 Federated Access 的設定方式，不外乎就是透過兩種：

- `SAML`

- `OpenID Connect`

以下看一下目前建立 IAM Role 時，Trust Entity Type 有哪些可以選：

![IAM Role - Trust Entity Type](/blog/images/aws/IAM/IAM-Trust-Entity-Types.png)

- 其實 Web Identity 就是透過 Cognito or OpenID Provider

- 而 SAML 2.0 federation 就直接告訴我們是用 SAML



設定方式
=======

以下使用串接 Facecbook 認證為例，假想一個情境：我要希望讓某個 Facecbook user 可以存取特定的 S3 bucket，那要怎麼做?

## AWS 端設定工作

基本上，Facebook 那一段的認證就由 Facebook 自己去處理，AWS 要處理的就是以下幾件事情：

- 設定好 IAM Role，綁定正確的 IAM Policy

- 設定 IAM Role Trust Relationship，指定來自於 Facebook 相關的訊息，而 Trust RelationShip 的設定大概如下所示：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            //表示信任來自於外部 Federated Account "graph.facebook.com" 
            //所送過來的 request
            "Principal": {
                "Federated": "graph.facebook.com"
            }, 
            //允許執行 "sts:AssumeRoleWithWebIdentity" 的操作
            "Action": "sts:AssumeRoleWithWebIdentity",
            //僅信任特定的 Facebook App ID
            "Condition": {
                "StringEquals": {
                    "graph.facebook.com:app_id"： "[YOUR_FACEBOOK_APP_ID]"
                }
            }
        }
    ]
}
```

透過以上的設定，就可以看得出來，其實只要從 FB 中取得一個資訊就好了，那就是 **Facebook App ID**，而這個 App ID 可以在到 [Facebook for Developer 網站](https://developers.facebook.com/)中，建立一個新的 App 並取得。

## 認證流程

基本上整體的認證流程是這樣：

1. 使用者透過 Facebook 登入，取得一組臨時的憑證(Access Token, UserID ... etc)

2. 程式可以使用該臨時憑證，對 AWS IAM 執行 Assuming Role 的操作

3. AWS IAM 會回應一組臨時用的憑證(Access/Secret Key & Token)

4. 使用者就可以使用 AWS 給出的臨時憑證來存取 AWS resource

了解以下正確的認證流程後，在開發相關程式就會比較得心應手了。



References
==========

- [Identity providers and federation - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers.html)