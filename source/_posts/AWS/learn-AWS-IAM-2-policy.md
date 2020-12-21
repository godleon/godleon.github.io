---
layout: post
title:  "[AWS IAM] 學習重點節錄(2) - IAM Policy"
description: "此篇文章是學習 Udemy 課程 \"AWS Identity Access Management (IAM) Practical Applications\" 時，將學習 IAM Policy 相關的重要概念，例如：Policy Structure, Statement ID, Effect, Action, Resource, Condition, Principal ... 等等的過程中整理出來的重點知識 & 正確使用方式"
date: 2020-12-21 06:40:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - IAM
---


IAM Policy 簡介
===============

## Overview

簡單來說，IAM Policy 的目的就是要管控"**在你帳號(root account)底下的使用者能做些什麼事情**"，當然前面就需要包含 Authenticate & Authorization 的過程

透過 AWS console，可以看到很多 pre-defined 的 IAM policy，並且可以看到以 JSON format 輸出的格式，而要自定義 IAM policy，也同樣是要提供 JSON 的定義來完成 

## Policy Type

IAM Policy 有兩種類型：

1. 使用者自行開發的 policy，稱為 **Customer Managed Policy**，使用者可以自行建立、編輯、刪除

2. 另外一種則是 AWS 預先定義好提供給 user 的 policy，稱為 **AWS Managed Policy**，只有提供 read only 的權限

因此若是 AWS 有發佈新的服務，就自然會有相對應的 IAM policy 被定義出來，提供給使用者方便使用。



IAM Policy Structure
====================

以下透過一個範例來簡單介紹 IAM Policy structure：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MyListBucketStatement", 
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": [
        "arn:aws:s3:::com.test.demo.file"
        "arn:aws:s3:::com.test.demo.images"
        "arn:aws:s3:::com.test.demo.notes"
      ]
    }
  ]
}
```

其中有幾個重要部份：

- [`Version`](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_version.html)：用來告知 AWS 是使用哪個 Language Syntax rule (基本上使用新的版本)

- [`Statement`](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_statement.html)：主要用來定義允許(or 拒絕)對哪些 AWS resource 進行操作

還有一些重要事項需要提醒：

- 所有的關鍵字(Version, Statement, Sid ... etc)都是有分大小寫的，寫錯會不可用

- 仔細看一下 Statement 的結構，它是一個 array，因此可以包含多個 sub statement

此外，**Statement Element** 一共有九種，分別是：

1. [Sid](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_sid.html)

2. [Effect](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_effect.html)

3. [Principal](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html)

4. [NotPrincipal](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notprincipal.html)

5. [Action](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_action.html)

6. [NotAction](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notaction.html)

7. [Resource](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_resource.html)

8. [NotResource](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notresource.html)

9. [Condition](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition.html)



[Statement ID (Sid)](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_sid.html)
=====================

- Sid 是選填的(optional)，因此不填也是可以的

- Sid 若是填了，就必須要確定在同一個 JSON document 中的 Sid 是不能重複的

- Sid 其實可以拿來作為識別該 policy 的用途

- Case sensitive，因此只要大小寫不同就會被視為不同的 Sid



[Effect](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_effect.html)
========

- Effect 只會有兩個值，`Deny` & `Allow`

- AWS 預設的 Effect 是 `Deny`，除非明確指定 `Allow` 才會是 Allow 的效果

- policy 中的每個 statement 都必須要定義 `Effect`

- 如果同時有 Allow & Deny effect 套用在同一個 resource，那效果就會是 deny



[Action](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_action.html)
========

## 基本概念

Action 基本上就是要清楚說明：**在那一個 AWS resource 上，可以做哪些事情?**

![IAM Action](/blog/images/aws/IAM-Action.png)

從上圖來說明：

- `resource`：指定 AWS resource

- `verb`：可以做什麼事情

> 因此上面就表示，可以對 S3 服務進行 ListBucket 的操作

## 每個服務到底有哪些 Action 可用 ?

有了 resource & verb 的概念後，另外一個問題來了 => AWS 服務這個多，那每個服務可用的 action 有哪些呢?

這部份肯定沒有人會去背下來，因此可以查詢[官網文件 "Actions, resources, and condition keys for AWS services - Service Authorization Reference"](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html) 得到解答，例如剛剛 [S3:ListBucket](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazons3.html) 的例子：

- **resource**：出現在文件的第一行 "service prefix: `s3`"，因此可以知道關鍵字就是 `s3`

- **verb**：文件往下移動，就可以看到 [`Actions defined by Amazon S3`](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazons3.html#amazons3-actions-as-permissions)，就可以看到有哪些 verb 可以使用

因此了解如何查詢官方文件，可以少記很多其實沒必要的東西，只要記住重要的事情即可!

## 如何更方便撰寫 Action ?

如果我們要設定的 Action 很多怎麼辦? 一個一個寫要寫很久的! 那這個時候就可以透過 **wildcard** 來幫忙這件事情，以下用幾個範例來說明：

- `s3:*`：

- `s3:Get*`

- `s3:*Bucket*`

## 其他特性

除了上面可用 wildcard 之外，Action 在撰寫上還有其他特性，例如：

- not case sensive

- 可以在同一個 Action 定義中，定義多個且不同的 service + verb

因此我們可以這樣調整：

- `s3:Get*` 與 `s3:get*` 是相同的

- `[s3:Get*, ec2:Attach]`：一次可以定義多個 Action

- `*`: 允許在所有 AWS resource(service) 上進行所有操作(verb)



Resources
=========

以上面的 S3 為例，當你建立了一個 bucket 之後，這個 bucket 就是一個 resource；接著又上傳了幾個檔案到 bucket 中，這幾個檔案也是一種 resource。

因此另一個問題就來了：**到底我們可以對哪些 AWS resource 進行操作? 要如何明確的定位這些 resource 呢?**

於是就有了下面的篇幅來說明這些事情：

## Amazon Resource Name

為了明確的定位到特定的 resource，AWS 提出了 Amazon Resource Name(ARN) 用來為每一個 resource 建立一個獨一無二的名稱之用。

![IAM Resource - ARN](/blog/images/aws/IAM-Resource-ARN.png)

以上圖為例，就很明確的定義出，此 policy **允許在 "arn:aws:s3:::com.test.demo.file", "arn:aws:s3:::com.test.demo.images"、"arn:aws:s3:::com.test.demo.notes" 三個 bucket 上進行 ListBucket 的操作**。

此外，ARN 的內容設定上會有一些重點需要了解：

- ARN 的定義也可以跟 Action 一樣使用 wildcard(`*` & `?`) 來進行設定，例如：`"Resource": "*"` 表示目標是所有的 AWS resource

- Resource 的定義是 case sensitive，因此要注意大小寫是有差別的

- policy 調整後生效可能會需要一點時間，可能從幾秒鐘甚至到一分鐘不等

## ARN Format

ARN 的設計目的是要定位到任何特定的 resource，而 ARN 的命名是把不同的服務、不同的 region、不同的使用者帳號這些事情都考慮進來後所設計出來的，因此就需要針對很多面向拆分出對應的資訊，以下是一個標準的 ARN format：

> arn:partition:service:region:account-id:resource-id

- 識別資訊是透過 ":"(冒號) 分隔的

- `arn` & `partition`：所有的 ARN 都是以 `arn` 開頭，partition 都是 `aws`

- `service`：這部份則是來自於 service prefix，這個資訊在上面提供的 [AWS 官網文件](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html) 找到每一個 AWS service 所對應到的 service prefix 是什麼

- `region`：這是 region ID，這資訊同樣也是可以到 [AWS 官網文件](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html) 找到，以 **Asia Pacific (Tokyo)** 為例，region ID 就是 `ap-northeast-1`
> 如果是屬於 global service(例如：**S3**)，這個資訊在 ARN 中就不會需要，因此在兩個 `:` 之間就啥也都不要寫就行

- `account-id`：進入 AWS console，在畫面右上角點選自己的帳號，就會從下拉選單中看到自己的 Account ID
> 有些服務不需要這資訊，例如：**S3**，那表示方法也就是在兩個 `:` 之間就啥也都不要寫就行

- `resource-id`：這個就是指定 resource 本身了，同時也可以透過 `path` 的方式來指定更明確的 resource(以下圖為例，透過 path 的方式，可以設定細到 object level 的 ARN)

![IAM Resource ARN format](/blog/images/aws/IAM-Resource-ARN-format.png)

更細膩的 AWS resource 表示方法也是有的，搭配 wildcard(`*` & `?`) 可以有下面幾種範例：

- `"Resource": "arn:aws:s3:::com.my.bucket.*"`：指定以 `com.my.bucket.` 開頭的 bucket & object

- `"Resource": "arn:aws:s3:::com.my.bucket.*s"`：指定以 `com.my.bucket.` 開頭，`s` 字元結尾的 bucket

- `"Resource": "arn:aws:s3:::*"`：所有 S3 bucket & object

如果有學過 regular expression，就會知道 `?` 表示單一字元， 因此搭配 `?` 的範例就不特別再列了。

## 如何寫出完整 & 正確的 ARN ?

基本上在 AWS console 中，大多數 resource 的 ARN 都可以直接從網頁上複製下來；但如果需要額外加 path 來指定的(例如：S3 object)，那就要知道 object path 才可以指定到正確的 resource。

而若是要清楚了解到底不同的 AWS resource ARN 要怎麼撰寫，可以參考以下兩個文件：

- [Amazon Resource Names (ARNs) - AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html) 取得更多資訊。

- [Actions, resources, and condition keys for AWS services - Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)

上面兩個文件是作為 reference 用，只要了解裡面有什麼資訊，在什麼情況下會需要用到它們，有需要再去查詢即可。


## 如何搞清楚 Action & Resource 的對應關係 ?

了解 Action & Resource 之後，接下來的問題就是，**要怎麼搞清楚每個 resource 可以對應到哪些 action 呢?**

有些 Action 很容易猜的出來，例如：`s3:ListBucket`，這很明顯一看就知道這 action 所影響的對象是 S3 bucket，但有時候遇到不熟的 service or Action 名稱定義上不清楚時，就會需要查文件了，此時以下文件可以幫到不少忙：

> [Actions, resources, and condition keys for AWS services - Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)

看上面的文件要搞清楚三個重要的部份：

- Resource Types

- Actions

- Condition Keys

以下就以 S3 的參考文件為例：

### Resource Types

![IAM policy resource types - S3](/blog/images/aws/IAM-Policy-Resource-Type.png)

可以看到目前 S3 服務中一共有五種 resource type，其中 bucket & object 是最常見的兩種，但在其他情況下可能也會使用到另外三種。

此外，還可以從 `ARN` 欄位中看到要表達該 Resource Type，ARN format 應該會是長成什麼樣子。

### Actions

接下來的問題是：**每一個 resource type 可對應到的 Actions 有哪些呢?**

在同一份文件中，點選右邊的 `Actions`，可以找到每個 Action 對應到的是哪個 Resource Type 的資訊，以下圖為例：

![IAM policy mapping of Resource Types & Actions  - S3](/blog/images/aws/IAM-Resource-Types-Actions-mapping.png)

可以很明確的看出每一個 Action 所對應的 Resource Type 是那一個。(例如：Action `IAM-Resource-Types-Actions-mapping.png` => Resource Type `bucket`)

但若是 Resource Type 是空白的，那就表示在 policy 中的 resource 是不需要的指定的，但因為 Resources 是必要欄位，因此要定義成：`"Resource": "*"`。

### Condition Keys

condition 這個主題很大，所以移動到下一個章節說。



Conditions
==========

## 簡介

在前面的設定中，除了設定 Action & Resource 之外，還可以透過 `Condition` 作為一個強大的手段可以進行複雜條件的設定，這功能就幾乎等同在原本的情況下再多加上 `IF` 的判斷(而用來判斷的資料依據可以有很多種)，若再加上 Statement 本身可以設定多個 sub statement，各種組合也都有可能被搭配出來。

![IAM policy Condition example](/blog/images/aws/IAM-Policy-Condition-1.png)

以上面的範例為例，這個 policy 會被套用在 AWS 使用者名稱完全等於 BOB 的情況下，但**若是名稱等於 Bob 就不會相符了**。

因此這時候衍生出一個問題：在 AWS IAM 中，使用者名稱是沒有分大小寫了，那這樣的判斷明顯無法忠實呈現正確的樣貌，因此就必須要將 `StringEquals` 改成 `StringEqualsIgnoreCase`。

## 撰寫 Condition 的規範

要寫好 Condition，首先要了解撰寫 Condition 的規則，首先：

- **Condition 中是可以放多個條件的** (透過 Array 的設定)：每個條件都是以 `AND` 來相互關聯
![IAM policy Condition format 1](/blog/images/aws/IAM-Policy-Condition-1.png)

- **Condition 的宣告是不能重複的**：例如：`StringEquals` 的設定不能出現兩次，若有這樣的需求，就必須把過濾條件放在同一個 `StringEquals` 之下 

以下是錯誤的：

![IAM policy Condition format 2 - Error](/blog/images/aws/IAM-Policy-Condition-Format-2-error.png)

要改成下面這樣才是正確的：

![IAM policy Condition format 2 - correct](/blog/images/aws/IAM-Policy-Condition-Format-2-correct.png)


## Condition Format

![IAM policy Condition Element - 1](/blog/images/aws/IAM-Policy-Condition-Elements-1.png)

接著來了解完整的 Condition 的正確結構，如上圖，會有三個部份：

- **Condition Operator**

- **Condition Key**

- condition value (這個就忽略不提了，就是條件值而已)

### Condition Operator

第一次看到 condition format，可能會想，到底有哪些 Condition Operator 可以用呢? 

這個答案可以直接參考 [AWS 官方文件(IAM JSON policy elements: Condition operators - AWS Identity and Access Management)](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html) 得到答案，在文件中，有根據資料來源的型態(string, numberic, date, boolean .... etc) 提供可用的 Condition Operator 的資訊。

以上面的 `aws:username`(String) 為例，就會有 `StringEquals`、`StringEqualsIgnoreCase`、`StringLike` .... 等等可用。 (可參考[此文件連結](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html#Conditions_String))

### Condition Key

![IAM policy Condition Key - 1](/blog/images/aws/IAM-Policy-Condition-Key-1.png)

另外一個接著來的問題是：有哪些 Condition Key 可以用呢? 那跟上面的 Action 的對應關係又是什麼?

首先提到 Condition Key 共有兩種類型，分別是：

1. Global Key

2. Service Key

#### Global Key

其中 Global Keys 是不論對象是什麼 service or action 都可以使用的，詳細的項目可以參考 [AWS 官網文件(AWS global condition context keys - AWS Identity and Access Management)](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html) 得到更多資訊。

需要注意的是，並不是所有的 Global Key 都會在任何時候都可以用，有些隨時都可以用(例如：`aws:CurrentTime`, `aws:EpochTime`)，有些則是在特定的狀況下才可以使用(例如：`aws:MultiFactorAuthAge`)，可以參考一下文件中 `Availability` 的說明得到更詳細的資訊。

#### Service Key

這個就是跟 Service 直接相關了，每個 Service 有其對應的 Condition Key 可以使用，當然不同的 Service 也會有所不同，而這資訊可以透過 [AWS 官網文件(Actions, resources, and condition keys for AWS services - Service Authorization Reference)](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html) 進行查詢。

以下以 S3 為例，從 S3 的服務頁面中可以看到以下資訊：

![IAM policy Condition Key - 2 - S3](/blog/images/aws/IAM-Policy-Condition-Key-2-s3.png)

從上表中可以看出來每一個 Condition Key 所對應到的資料類型是什麼，有了正確的資料類型資訊後，就可以找出對應的 Condition Operator 可以用哪些。

接著，每個 Action 有其對應的 Condition Key 可以用，並非任何一個 Action 都可以與任一個 Condition Key 搭配，而這個部份，就要查詢 [S3 文件](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazons3.html)中 Actions 部份資訊：

![IAM policy - Action & Condition Key mapping](/blog/images/aws/IAM-Resource-Types-Actions-mapping.png)

從上表可以看得出來每個 Action 有其對應的 Condition Key，因此每個 Service 所可以用的 Condition Key 就是透過這樣的方式查詢出來的。

## 如何撰寫正確的 Condition 內容?

上面看了這麼多，了解了每個部份的相互關係之後，那要如何撰寫正確的 condition 呢?

其實在每個 Service 的文件中都會有一個 `Security` 的部份，而在裡面找一下關於 Policy and Permission 這樣的文件，裡面就會有 Condition 的設定說明，以 S3 為例，可以參考[此文件](https://docs.aws.amazon.com/AmazonS3/latest/dev/amazon-s3-policy-keys.html)，在裡面可以看到很多範例可以根據需求直接修改後使用。

## 範例

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": ["s3:ListBucket"],
      "Effect": "Allow",
      "Resource": ["arn:aws:s3:::mybucket"],
      "Condition": {"StringLike": {"s3:prefix": ["David/*"]}}
    },
    {
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Effect": "Allow",
      "Resource": ["arn:aws:s3:::mybucket/David/*"]
    }
  ]
}
```

效果：

- 可以對 **arn:aws:s3:::mybucket** bucket 中，prefix 為 David 的執行 List Bucket 的操作

- 可以對 **arn:aws:s3:::mybucket/David** 這個 bucket 的 object 進行 GET & PUT 的操作

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": ["s3:ListBucket"],
      "Effect": "Allow",
      "Resource": ["arn:aws:s3:::mybucket"],
      "Condition": {"StringLike": {"s3:prefix": ["${aws:username}/*"]}}
    },
    {
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Effect": "Allow",
      "Resource": ["arn:aws:s3:::mybucket/${aws:username}/*"]
    }
  ]
}
```
> 效果同上，只是上面這個 policy 只限定針對帶有自己帳號的 bucket 有效，而非 David



Principal
=========

- 首先搞清楚 policy 有分兩種：`User Based Policy`(這個是要套用在使用者身上) & `Resource Based Policy`(這是要套用在 AWS resource 身上)

- 在 user based policy 中，是無法設定 principal 的，因為這一類的 policy 要透過 attach 的方式發生效果，因此放入 principal 資訊自然也就不合理

- 正確的用法是要將 principal 套用在 `Resource Based Policy` 上 (例如：**S3 Bucket Policy**)

- 只要有任何一種 policy 跟特定使用者有關，就會對它生效(例如：Resource Based Policy 中允許 Bob 讀取 S3 資料，即使 Bob 沒有與任何 User Based Policy 關聯，還是有權限可以存取 S3)

- 甚至可以將 principal 設定成另外一個 Account ID 的使用者 (但對方對特定的 AWS resource 也有要相對應的權限才行)
> 例如：設定一的 S3 bucket policy，同時允許帳號 A 的 Bob & Alice 以及帳號 B 的 John 進行存取，但若是 John 本身沒有存取 S3 的權限也是不行，必須要確保 John 本身有 S3 的存取權限才可以使用該 S3 bucket




References
==========

- [AWS Identity Access Management (IAM) Practical Applications | Udemy](https://www.udemy.com/course/aws-identity-access-management-practical-applications/)

- [Amazon Resource Names (ARNs) - AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html) 取得更多資訊。

- [Actions, resources, and condition keys for AWS services - Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)


- [IAM JSON policy elements: Condition operators - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html)

- [AWS global condition context keys - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html)

- [AWS JSON policy elements: Principal - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html)