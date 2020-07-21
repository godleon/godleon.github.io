---
layout: post
title:  "AWS CSA Associate 學習筆記 - DB"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 Application 相關服務，包含 SQS、SNS、SWF、Elastic Transcoder、API Gateway、Kinesis、Cognito ... 等等內容"
date: 2020-05-25 06:20:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - SQS
  - SNS
  - SWF
  - Elastic Transcoder
  - API Gateway
  - Kinesis
  - Cognito
---


SQS(Simple Queue Service)
=========================

## 什麼是 Message Queue ?

![Message Queue](/blog/images/aws/SQS_Message-Queue.png)

Message Queue 這個概念被提出來，有兩個很重要的原因：

1. 將不同服務之間的通訊進行封裝

2. 將各個服務進行解耦(decouple)

將服務之間的通訊進行封裝，並帶入了 producer & consumer 的概念，讓原本複雜的訊息傳遞，變成了相對容易的概念，大幅減少程式設計師在處理訊息通訊這件事情所花費的精力，可以把時間花在真正重要的事情上。

而解耦服務這件事情，就是在微服務時代必備的一項功能了，整個概念大概就是將大型服務拆分成多個小型服務，並透過 messsage queue 來互相交換訊息，藉此讓開發人員可以專心在單一功能，降低錯誤的發生，也可以單獨對服務進行升級。

關於更多 Message Queue 的知識，Google 找一下就一堆了，這邊就不繼續說下去......


## 什麼是 SQS ?

SQS 是 AWS 提供的全託管 message queue 服務，這在目前現在化的系統架構中，是個很重要的一個部份，以下先來看看 SQS 的官方介紹：

> Amazon Simple Queue Service (SQS) is a fully managed message queuing service that enables you to decouple and scale microservices, distributed systems, and serverless applications. SQS eliminates the complexity and overhead associated with managing and operating message oriented middleware, and empowers developers to focus on differentiating work. Using SQS, you can send, store, and receive messages between software components at any volume, without losing messages or requiring other services to be available. Get started with SQS in minutes using the AWS console, Command Line Interface or SDK of your choice, and three simple commands.

從上述的介紹中，可以看出幾個 SQS 服務的重點：

- 全託管(fully managed)的 message queue 服務

- message queue 的用途是在微服務、分散式、Serverless 架構中，將多個功能進行解耦(decouple)與擴展(scale)

- 大幅降低了處理訊息傳遞工作的複雜度，讓開發者可以專住在業務相關的開發上

- 可同時承受極大量的訊息，並確保訊息不會遺失


## SQS 的特性

要使用 SQS 之前，要先搞清楚它本身的特性，才不會犯下沒有必要發生的錯誤......

AWS SQS 提供了兩種 Queue Type，分別是：

### Standard Queue (default)

![SQS](/blog/images/aws/SQS_Standard-Queue.png)

- 這是預設的 Queue Type

- 每秒可以承受的訊息量極大(基本上可以當作是無限制)

- 可以確保每個訊息都至少被傳送一次 (**有可能會超過一次....**)

- 若是訊息量真的太大，會有偶發性訊息發送順序跟原有傳送順序不一致的情況發生 (但會盡可能的達到順序一致)

因此可以看出 standard queue 偶而會發生失誤，如果 application level 可以 cover 這個問題，或是服務本身性質對這個問題無感，那 standard queue 絕對是最好的選擇。

### FIFO Queue

![SQS](/blog/images/aws/SQS_FIFO-Queue.png)

- 基本上就是改良 standard 版本的缺點，包含了以下的改良：
    - 順序一定會保證一致一定會確保訊息只送一次，不會重複發送
    - 保證訊息在 consumer 取走並刪除前都可以取得

- 支援 message group，在同一個 Queue 中，每個 group 可以有自己的排序設定

- TPS(Transaction Per Second) 最大只有 300

- 包含 standard queue 中所有的功能


## 考試重點

- SQS 是種 message queue 服務，這類服務被設計出來的很重要的其中一個目的，就是要 **decouple** service，讓不同的 service 可以各自專心處理與自己相關的細節

- SQS 是 pull-based 類型的服務，非 pushed-based；這表示 SQS 不會主動送訊息出來，需要有個服務主動去拉(例如：EC2 instance)

- Message 的大小是可以從 1 byte ~ 256 KB (可參考[官方文件](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-quotas.html))

- 訊息可以保留在 Queue 中的時間範圍為 1 分鐘 ~ 14 天，可以自由設定 (預設是 4 天)

- Visibility Timeout 是指當 consumer 拿走特定 message 之後，該 message 就會進入 invisible 的狀態，其他 consumer 自然就無法取來處理；一旦超過 timeout 設定，同一個 message 就可以被其他 consumer 取走並處理
> **若是 visibility timeout 設定太小，可能工作還沒處理結束，其他 consumer 又從 Queue 上面取下來作(等於傳送兩次)**，這樣就會發生預期之外的結果，因此在設定 visibility timeout 必須搞清楚每個 job 會需要的執行時間才能做正確的設定

- Visibility Timeout 的設定最長是 12 小時
> 因此若是 job 會執行超過 12 小時，可能 SQS 就不是適合拿來搭配的服務.....

- SQS 會保證訊息至少會送出一次

- SQS 支援 long polling 的連接方式，因此即使是 pull-based，還是可以藉此減少建立連線的成本
> 持續的向 SQS 建立連線是需要額外花費的，使用 long polling 可以省下不少費用



SWF(Simple Workflow Service)
============================

> 參考課程中的講者有提到，SWF 幾乎沒在考試中出現，所以大概了解一下應該就可以了...

{% youtube dFD7CEM1vbc %}

- 主要目的在協助開發人員建立、執行、調整一連串執行的背景工作(可能某些工作是平行處理的)

- 可將特定的服務/程式與人工動作結合

- 可以協助追蹤每個階段的工作執行狀態


## SWF Actor (重要組成元素)

SWF Actor 共有以下三類：

- `Workflow Starter`：用來啟動 workflow 的 application (例如：網站上的下單行為)

- `Decider`：根據前一個 task 的 input(或是遇到失敗了) 來決定下一個 task 是那一個，或是結束

- `Activity Worker`：就是實際執行每個 task 內容的角色


## SWF 與 SQS 比較

- SQS 的訊息最多保留 14 天，但在 SWF 中的工作最長可以執行一年

- SWF 提供了 task-oriented API；SQS 提供了 message-oriented API

- SWF 會確保每個 task 只會被分派一次，不會重複；但 SQS 有可能會重複(standard queue + 大量訊息)

- SWF 會追蹤應用程式中所有的 task & event；但 SQS 不支援 application-level tracking，必須得自己實作(如果使用多個 queue 時)



SNS(Simple Notification Service)
================================

## 什麼是 SNS ?

SNS 是 AWS 提供作為訊息通知的服務，但其實不僅僅是簡單的訊息通知而已，其實 SNS 的服務在設計上，所涵蓋的範圍是很廣的，可以滿足大多數訊息發送的要求；以下是 AWS 對 SNS 的介紹：

> Amazon Simple Notification Service (SNS) is a highly available, durable, secure, fully managed pub/sub messaging service that enables you to decouple microservices, distributed systems, and serverless applications. Amazon SNS provides topics for high-throughput, push-based, many-to-many messaging. Using Amazon SNS topics, your publisher systems can fan out messages to a large number of subscriber endpoints for parallel processing, including Amazon SQS queues, AWS Lambda functions, and HTTP/S webhooks. Additionally, SNS can be used to fan out notifications to end users using mobile push, SMS, and email.

從上面的描述可以看出以下重點：

- **push-based**：這跟 SQS 不一樣，SNS 會主動將訊息發送出去

- **高可用**：當要轉發的訊息送到 SNS 後，會自動被複製到多個 AZ 上，確保訊息不會遺失

- **訂閱機制**：SNS 本身就是個 pub/sub 的設計，搭配 `topic`，可以讓使用者(or 服務)進行訂閱，當有訊息傳到 topic 後，SNS 就會主動將訊息傳送給訂閱者

- **支援各式 endpoint**：訊息不僅可以傳給一般的使用者，也可以傳給其他服務；此外，傳遞的方式也有多種，可以是 SMS, email，甚至是直接呼叫 Lambda function 或是將資料再丟進 SQS 中都可以
> 藉由 SNS 支援的多種 endpoint type，開發者就可以很輕易的根據需求將訊息傳遞到不同的使用者、device，甚至是 service 了


## SNS 的優點

- push based，因此在費用上就會相對比 pull-based 省不少

- 提供簡單的 API，並容易與其他的應用進行整合

- 彈性的訊息傳遞設定(也支援多種不同的傳送協定)


## SNS 與 SQS 比較

- 同樣都是 messaging service

- SNS 為 push-based

- SQS 為 pull-based


Elastic Transcoder
==================

首先看一下 AWS 官網對 Elastic Transcoder 的介紹：

> Amazon Elastic Transcoder is media transcoding in the cloud. It is designed to be a highly scalable, easy to use and a cost effective way for developers and businesses to convert (or “transcode”) media files from their source format into versions that will playback on devices like smartphones, tablets and PCs.

從上面的描述可以看出來 Elastic Transcoder 有幾個特點：

- highly scalable & cost effective 就不用說了，AWS 每個服務都是這麼介紹自己的....

- 此服務是作為多媒體檔案的轉檔服務(media transcoding)用

- 來源為 file(並非 streaming)，因此很直覺就可以聯想到一定可以跟 S3 串接

- 可以輕易的將來源檔案轉換成不同設備可以看的版本 & 解析度

有了以上的說明之後，以下這個簡單的應用範例看起來就很直覺了：

![Elastic Transcoder Example](/blog/images/aws/ElasticTranscoder_Example.png)

上面的服務串接做了以下事情：

1. 將錄製好的影像檔案上傳 S3

2. 透過 Lambda function 拉取 S3 中的影像檔案，並丟給 Elastic Transcoder，根據設定轉成符合各種設備 & 不同解析度的檔案

3. 將轉檔後的結果上傳到 S3


API Gateway
===========

## 什麼是 API Gateway ?

首先，AWS 原廠對 API Gateway 的解釋有兩段，第一段大致說明了 API Gateway 的特色：

> Amazon API Gateway is a fully managed service that makes it easy for developers to create, publish, maintain, monitor, and secure APIs at any scale. APIs act as the "front door" for applications to access data, business logic, or functionality from your backend services. Using API Gateway, you can create RESTful APIs and WebSocket APIs that enable real-time two-way communication applications. API Gateway supports containerized and serverless workloads, as well as web applications.

- 全託管服務，代表你不用自己建立一台 EC2 instance 來提供服務

- 讓開發者可以更容易建立、發佈、維護、監控並安全地提供 API 服務

- API Gateway 可以提供 RESTful & WebSocket API，作為存取內部服務的大門

- 後端的服務可以以 Container(由 EKS or ECS 提供)、Serverless(由 Lambda 提供) 或是一般的 web application(由 EC2 提供)，也可以是類似 DynamoDB 這樣的服務

![API Gateway Example](/blog/images/aws/APIGateway_Example.png)


## API Gateway 的細部功能

第二段官方說明中就比較著重在細部的功能：

API Gateway handles all the tasks involved in accepting and processing up to hundreds of thousands of concurrent API calls, including traffic management, CORS support, authorization and access control, throttling, monitoring, and API version management. API Gateway has no minimum fees or startup costs. You pay for the API calls you receive and the amount of data transferred out and, with the API Gateway tiered pricing model, you can reduce your cost as your API usage scales.

- API Gateway 可以根據 request 數量 auto scale，因此同時承受數以十萬計的 API call 是沒有問題的

- 若要防止 DDOS，也可以進行流量的管理

- 支援 CORS，可以讓不同 domain 的 web site 之間可以互相存取 API

- 可搭配其他安全機制進行授權存取的管理

- 可搭配 CloudWatch 進行監控

- 版本管理

- 提供多種不同成本控制 & 管理的方式

## 如何設定 & 佈署 API Gateway

### 設定 API Gateway 

設定 API Gateway 大致上的流程如下：

1. 定義一個 `API`

2. 定義 `Resource` & 其他的 nested Resource(以 `URL path` 的形式)

3. 對每一個 Resource 進行設定：
    - 選擇使用的 `HTTP method` (GET, POST .... etc)
    - 進行安全性相關設定
    - 選擇 `Target` (EC2, Lambda, DynamoDB .... etc)
    - 設定 request & response 的轉換格式

### 佈署 API Gateway

佈署 API Gateway 的流程則是如下：

- 將 API Gateway 佈署到指定的 `stage`
    - 預設使用的是 API Gateway 服務提供的 domain
    - 也可以使用自訂的 domain
    - 可搭配 AWS Certificate Manager 取得免費的 SSL/TLS 憑證

## Cache(快取)功能

API Gateway 還提供了一個很重要的 cache 功能，說明如下：

- 透過 cache，實際上到達後端的 request 會減少，有些 API call 可以在 API Gateway 這一層就直接回應了

- 可以降低 Latency

- 若要開啟 cache 功能，則需要到 `stage` 的設定中開啟

- 可透過設定 cache TTL，決定 cache 要存活的時間長短 (在設定的 TTL 時限內，API Gateway 就會直接使用 cache 回應，不會將 API call 轉到內部服務去)


## 考試重點

- API Gateway 運作的類似像 load balancer，但是個全託管服務

- 對外提供 HTTPS endpoint，對內可指向多種內部服務

- 提供 cache 機制來提昇服務效能

- 低成本高效能，可自動擴展

- 可透過限流的方式來阻止攻擊行為

- 可搭配 CloudWatch 服務，存放 log 相關資訊

- 若使用類似 AJAX 的功能存取位於不同 domain 的 API gateway，則需要開啟 CORS 的功能
> CORS 本身在 browser 中是預設啟用的


Kinesis
=======

## 什麼是 Kinesis?

以下是 AWS 官網對 Kinesis 的介紹：

> Amazon Kinesis makes it easy to collect, process, and analyze real-time, streaming data so you can get timely insights and react quickly to new information. Amazon Kinesis offers key capabilities to cost-effectively process streaming data at any scale, along with the flexibility to choose the tools that best suit the requirements of your application. With Amazon Kinesis, you can ingest real-time data such as video, audio, application logs, website clickstreams, and IoT telemetry data for machine learning, analytics, and other applications. Amazon Kinesis enables you to process and analyze data as it arrives and respond instantly instead of having to wait until all your data is collected before the processing can begin.

從上面的官方介紹可以整理出幾個 Kinesis 的服務重點：

- 處理 streaming data 用

- 可針對資料進行即時分析處理，不需要等待資料全部到齊

- 資料來源可以從 video, audio, application log, 網站點擊所產生的資料、IoT 遙測資料 ... 等等

- 可針對資料進行分析，甚至可以搭配其他服務進行 machine learning 相關的工作

## Kinesis 可以提供哪些類型的服務?

Kinesis 根據功能共分為以下四類：

- Kinesis Video Streams

- Kinesis Data Streams

- Kinesis Data Firehose

- Kinesis Data Analytics

### Kinesis Video Streams

![Kinesis Video Streams Example](/blog/images/aws/KinesisVideoStreams_Example.png)

可針對視訊的串流資料進行回放、安全監控、臉部辨識、機器學習，或是其他分析....等工作。

### Kinesis Data Streams

![Kinesis Data Streams Example](/blog/images/aws/KinesisDataStreams_Example.png)

- 資料在 Kinesis 中會以 shard 的形式存在，保留在 persistent storage 中

- 搭配後方的服務(例如：EC2, Lambda, Kinesis Data Analytics) 進行後續的分析處理工作

- 處理完的資料可以存到 S3, EMR, Redshift, DynamoDB .... 等地方做後續運用

### Kinesis Data Firehose

![Kinesis Data Firehose Example](/blog/images/aws/KinesisDataFirehose_Example.png)

- 沒有 persistent storage，資料進入 Kinesis 後要立即處理

- 可選擇搭配 Lambda or Kinesis Data Analytics 來進行資料處理

- 處理完後的資料可以轉存入 S3 or Elasticsearch 服務中

### Kinesis Data Analytics

![Kinesis Data Analytics Example](/blog/images/aws/KinesisDataAnalytics_Example.png)

- 此類型的 Kinesis 則是將查詢和分析直接設定在 Kinesis 服務本身

- 輸出的結果可以存到 S3, Redshift, Elasticsearch ... 等服務

## 考試重點

- 了解 `Kinesis Data Streams` & `Kinesis Data Firehose` 的差異

- 了解 Kinesis Data Analytics 可以如何被使用


Cognito
=======

## 什麼是 Cognito?

AWS Cognito 是個用來解決"使用者認證"這件事情的服務。

在網路上提供服務，使用者認證這件事情總是讓人覺得很麻煩，但不做又不行，到底是要自己建立帳號管理系統，還是直接去接 Google or FB 的認證就好，開發也其實並非很簡單，因此 AWS Cognito 這項服務就是要讓這件事情變得容易。

先看看 AWS 官網對 Cognito 的說明：

> Amazon Cognito lets you add user sign-up, sign-in, and access control to your web and mobile apps quickly and easily. Amazon Cognito scales to millions of users and supports sign-in with social identity providers, such as Facebook, Google, and Amazon, and enterprise identity providers via SAML 2.0.

以下整理出幾個重點：

- Cognito 是用來讓開發者很方便的可以在 web or mobile app 上提供使用者登入、登出、存取控制...等功能

- 可與 FB, Google, Amazon, Microsoft Azure Directory ....等服務的認證進行整合(透過 SAML 2.0)

## 服務特色

- 不用擔心可服務的使用者數量，AWS Cognito 可以全部扛起來....

- 可與常見的 social identity provider(FB, Google .... etc) 進行整合

- 支援各種帳號整合標準，例如：Oauth 2.0, SAML 2.0, OpenID Connect ... 等等

- 支援 multi-factor authentication 以及多項資安標準

- 可與 AWS IAM 搭配，對 AWS 其他服務 & 資源進行存取控制，例如下圖所示：

![Cognito Example 1](/blog/images/aws/Cognito_Example.png)

- 使用內建 UI，搭配一些客製化，很容易就可以整合到現有的服務中，不需要撰寫任何程式碼

- 可將使用者變更後的資料同步到多個裝置中，並發送通知，例如下圖所示：

![Cognito Example 3](/blog/images/aws/Cognito_Example3.png)

## User Pool & Identity Pool

在 Cognito 服務中，有兩個部份需要特別注意，分別是 User Pool & Identity Pool

### User Pool

- User Pool 是 Cognito 服務中實際存放使用者資訊的地方

- 使用者不一定要透過 FB or Google 認證進行登入，也可以使用 User Pool 中的資訊進行登入

- Cognito 就只是一個 broker，將認認需求傳送到 User Pool or FB or Google

- 成功認證後的使用者可以取得一個 JWT(Json Web Token)

### Identity Pool

Identity Pool 則是提供臨時的權限，用來存取 AWS 資源 or 服務

### 搭配使用的場景

以下是一個 User Pool 搭配 Identity Pool 的場景：

![Cognito Example 2](/blog/images/aws/Cognito_Example2.png)

- 使用者透過 User Pool 進行認證，認證資料可能來自於 User Pool 本身或是 FB or Google

- 認證成功後取得 JWT

- 使用者拿著 JWT 去向 Identity Pool 換取一個臨時用的 AWS credential

- 使用者有了 AWS credential 後就可以存取特定的 AWS 資源或服務

## 考試重點

- Cognito 可與 Google, FB, Amazon 等 social identity provider 進行認證

- Cognito 只扮演 broker 的角色，協助向不同 provider 進行認證

- 認證成功後會取得 token，可用來交換成 AWS credential，進而具有特定的 IAM role 的身份

- User Pool 負責使用者註冊、認證、帳號恢復...等工作

- Identity Pool 則是用來賦予 AWS 資源與服務的存取權限


References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Amazon Simple Queue Service (SQS) | Message Queuing for Messaging Applications | AWS](https://aws.amazon.com/sqs/)

- [高可靠度的微服務通訊 - Message Queue — 安德魯的部落格](https://columns.chicken-house.net/2019/01/01/microservice12-mqrpc/)

- [Amazon Simple Queue Service (SQS) | Message Queuing for Messaging Applications | AWS](https://aws.amazon.com/sqs/)

- [Amazon Simple Workflow Service - Cloud Workflow Development - AWS](https://aws.amazon.com/swf/)

- [Amazon Simple Notification Service (SNS) | AWS](https://aws.amazon.com/sns)

- [AWS | Amazon Elastic Transcoder - Media & Video Transcoding in the Cloud](https://aws.amazon.com/elastictranscoder)

- [Amazon API Gateway - Amazon Web Services](https://aws.amazon.com/api-gateway)

- [Amazon Kinesis - Process & Analyze Streaming Data - Amazon Web Services](https://aws.amazon.com/kinesis)

- [Amazon Cognito - Simple and Secure User Sign Up & Sign In | Amazon Web Services (AWS)](https://aws.amazon.com/cognito)