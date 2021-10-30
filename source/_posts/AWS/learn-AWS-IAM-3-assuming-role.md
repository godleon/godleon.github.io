---
layout: post
title:  "[AWS IAM] 學習重點節錄(3) - Assuming IAM Role"
description: "此篇文章是學習 Udemy 課程 \"AWS Identity Access Management (IAM) Practical Applications\" 時，將 IAM Assuming Role 的設計 & 作法詳細記錄下來，並以較為淺顯易懂的方式重新說明"
date: 2020-12-22 21:10:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - IAM
---

前言
===

在原本的權限設定部份，當我們新增了 IAM User 後，為了讓使用者有特定 AWS resource 的權限，此時我們會繼續設定 IAM Policy，接著再將 IAM policy attach 到 IAM User 中，此時每一個 User 就會有指定權限了。

但若是使用者越來越多怎麼辦? 一個一個設定 IAM User 然後再 attach policy 其實是很累的，因此就可以透過設定 `IAM Group`，讓這件事情很簡單就可以完成，以下圖為例，我們有兩個 IAM Group，分別是 **Engineers** & **Consultants**：

![IAM Group example](/blog/images/aws/IAM-Assume-Role-1-Group-Example.png)

另外一個問題來了：如果使用者權限很特殊，他**希望可以同時存取兩個 IAM Group 可存取的 AWS resource，但又希望可以在單一的時間點以特定 Group 的視角來操作 AWS resource**，這要怎麼做?

這時候就是 `IAM Role` 出場的時候了!

此時只要記得兩個重點：

1. **IAM User 可以屬於某個 IAM Group，甚至可以屬於多個 Group**

2. **IAM User 無法屬於某個 IAM Role，必須透過"切換"的方式，在 AWS 中稱為 "Assume Role"，而"切換"這個操作需要有權限才行**



IAM Role 如何運作?
=================

首先先看一下將原本 IAM Group 更換為 IAM Role 後，會變成以下概念：

![IAM Role example](/blog/images/aws/IAM-Assume-Role-2-Role-Example.png)

每個 Role 都有正確的 IAM Policy 設定，因此可以存取指定的 S3 bucket。

到這邊似乎看不出來跟原本的 IAM Group 有什麼差異，繼續往下看：

![IAM Role - Assume Engineers Role](/blog/images/aws/IAM-Assume-Role-3-Assume-Engineers-Role.png)

在上圖中，Bob 可以透過 Assume Role 的方式，將自己的權限切換到 Engineers Role，接著就可以存取 Engineers Role 中可存取的 AWS resource。

當然，若是有足夠的 Assume Role 權限，Bob 甚至還可以切換到 Consultants Role：

![IAM Role - Assume Consultants Role](/blog/images/aws/IAM-Assume-Role-4-Assume-Consultant-Role.png)

想當然爾，這時候 Bob 就擁有了 Constants Role 所對應到的權限了，自然也可以存取到對應的 AWS resource。



如何取得 Role 的身份?
==================

那接著 Bob 要如何具備 EngineerRole 的身份呢? 很簡單，只要將 Bob assume 成 Enginner Role 即可，但要做這件事情，必須要有 `sts:AssumeRole` 的 Action 權限。

這是什麼意思?

`Assume Role` 基本上是一種 Action(`"Action": "sts:AssumeRole"`)，因為 Assume Role 這個行為是從 **AWS Security Token Service** 中取得一個暫時的 token，藉此取得該 Role 所事先定義好的權限。(`sts:AssumeRole` Action & IAM Role 的對應關係可以從[此 AWS 官網文件](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awssecuritytokenservice.html#awssecuritytokenservice-actions-as-permissions)找到)

因此! 還要建立一個 policy，專門用來針對特定的 Resource(這裡當然指的就是 `IAM Role` 了)，允許 `"Action": "sts:AssumeRole"` 的操作，內容大致會如下所示：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAssumeRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": [
                "arn:aws:iam::777777777777:role/MyTestEngineerRole",
                "arn:aws:iam::777777777777:role/MyTestConsultantRole"
            ]
        }
    ]
}
```

接著再將這個 policy 套用在 IAM User 身上，此時 IAM User 就會擁有切換 Role 來取得不同 AWS resource 存取權限的能力。



如何切換成特定的 Role 取得權限?
===========================

假設使用者已經被賦予 AssumeRole 的權限，那就可以從 AWS console 右上角，點選自己的帳號，就可以看到一個 `Switch Roles` 的選項，進入後可以看到以下畫面：

![IAM Role - Switch Roles](/blog/images/aws/IAM-Switch-Role.png)

填入 `Account ID` & `Role`(IAM Role Name) 的欄位，就可以切換到特定的 Role 了；此外，還可以選填 `Display Name` 並選擇 `Color` 取得更清楚的識別。

## 注意事項

Switch Role 是個很好用的功能，但在使用上有以下兩點需要注意：

- 每個 IAM User 一次只能 assume 成一個 IAM Role

- 當 switch 成另外一個 IAM Role，**原有 attach 在 IAM User 身上的權限都會不見，只剩下該 IAM Role 的權限**

因此，若是切換成另外一個 IAM Role 後，發現怎麼原有的權限不見了，那就表示該 IAM Role 並沒有你原本所擁有的權限。


## Security

- 在上面的範例中，只要知道 `Account ID` & `IAM Role Name` 兩個資訊，就可以自由切換成不同的 Role，這樣其實在安全性上並不是太好，因此建議的方式是，針對每一個 IAM Role 都設定各自的 Assuming Policy。

因此以上面的例子為例，就會拆成以下兩個 IAM Policy：

```json
//套用這個 policy 的 IAM User 只能切換成 Engineer
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAssumeRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": [
                "arn:aws:iam::777777777777:role/MyTestEngineerRole"
            ]
        }
    ]
}
```

```json
//套用這個 policy 的 IAM User 只能切換成 Consultant
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAssumeRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": [
                "arn:aws:iam::777777777777:role/MyTestConsultantRole"
            ]
        }
    ]
}
```

如此一來，帳號管理的安全性上又更進一步的提昇了!



總結
===

Assuming Role 是 IAM 中很重要的一個功能 & 觀念，因為不只是 IAM User 可以切換成特定的 IAM Role，連 AWS service 本身也可以透過 Assuming Role 的功能切換成特定 IAM Role 來暫時取得存取 AWS resource 的權限。

因此這個觀念務必要熟悉，在進行 AWS 服務架構設計規劃時，才可以設計出正確且安全的權限管理規範 & 計畫。



References
==========

- [AWS Identity Access Management (IAM) Practical Applications | Udemy](https://www.udemy.com/course/aws-identity-access-management-practical-applications/)

- [Actions, resources, and condition keys for AWS Security Token Service - Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awssecuritytokenservice.html)