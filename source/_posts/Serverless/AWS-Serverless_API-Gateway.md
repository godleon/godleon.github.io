---
layout: post
title:  "AWS Serverless 學習筆記 - API Gateway"
description: "此篇文章介紹 AWS API Gateway 的概念與特性，以及使用上所需要注意的操作細節"
date: 2022-04-16 11:15:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - Serverless
  - API Gateway
  - Lambda
---


什麼是 API Gateway ?
===================

API Gateway 其實就是接收所有 API 的第一個入口，並協助分發到後端的程式進行處理的一個角色，假設以下圖的餐廳為例：

![What is API Gateway](/blog/images/serverless/AWS-serverless_what-is-api-gateway.png)

在此場景中，所有的客人都是發送 API 的 client 端，然後右邊像是 Dessert/Main Course/Drinks ... 等幾個 station 則是幾個作為 backend service 的部份，最後則是中間的 Orchestrator 的角色，負責接收客人的需求，並將需求正確的轉發給對應的 station 處理。

因此在這個場景中，orchestrator 就是屬於 API Gateway 的角色。

而實際中，AWS API Gateway 提供的重要功能包含以下幾個(但不限於)：

1. 讓開發人員可以根據業務需求，建立、發佈、維護、監控和保護 API service

2. 提供 API 認證(authentication) & 授權(authorization) 的功能

3. 可對 API request 提供 tracing、caching、甚至是 throttling 的功能

4. 提供 staged deployment、canary release ... 等功能

以下是 AWS API 的架構示意圖：

![AWS API Gateway](/blog/images/serverless/AWS-serverless_AWS-API-Gateway-Arch.png)

可以看出，API Gateway 背後可支援的 backend service 有很多個，Lambda 只是其中一個，也可以是 EC2、Kinesis、DynamoDB、甚至是其他 public endpoint 都可以。


API Gateway Components
======================

以下介紹在 AWS Gateway 的設定中，比較重要的幾個 component：

## Resources

`Resources` 中是用來定義所有的 API 清單，但若沒有進行佈署(deploy)，是不會在任何環境生效的
> 例如：每次開始學習都會新增的 `/helloworld`

而在每個 Resource 底下，都可以新增不同的 `Method`(目前支援 GET、POST、HEAD、DELETE、OPTIONS、PUT、ANY ... 等七種)，每個 method 都可以對應到不同的 backend service。


## Stages

提供 `Stages` 的設計，等同於 dev/staging/production 這樣的設計，可以讓開發者決定將 API 佈署(deploy)到哪個環境

## Models

`Models` 用來定義 API 所使用的資料格式，透過 JSON schema lanaguage 來定義；舉例來說，可以檢查傳入的資料應該有哪些欄位，直接拒絕不符合傳入資料定義的 request，算是可以額外設定一層的自動檢查機制
> 這是選擇性設定，不設定也是沒關係的

## API Keys

可產生 API Key，用來作為呼叫 API 時的身份認證機制，可以防範未授權的呼叫 API

而 API Key 同時也可以額外綁定 Usage Plan 的設計來限制用量(例如：每秒不能呼叫超過 100 次 API)

## Usage Plan

這部份的設定可以讓 API client 對於存取特定的 API，進行 throttling & quota limits ... 等設定，而限定的依據會需要透過 client API key 來完成

## Custom Domain Names

若要使用自訂的 domain name 存取 API，那就需要透過 `Custom Domain Names` 的功能，需要指定以下內容：

- domain name

- 支援的最小 TLS version

- Endpoint Type (Regional or Edge-Optimized with CloudFront)

- ACM 憑證


API Gateway Endpoint Type
=========================

API Gateway Endpoint Type 有分為三種，分別是：

- edge optimized

- regional

- private

## Edge Optimized

![AWS API Gateway Endpoint Type - edge optimized](/blog/images/serverless/AWS-serverless_API-Gateway-Endpoint_edge-optimized.png)

- 利用了 CloudFront 散布在全世界的 edge 節點，最靠近使用者，可取得最低的 latency

- 但也有無法取得內部資料的問題，例如：DB 資料(in private subnet)

> 基本上，設計時需要考量與外部服務互動的需求，才能判斷這個是不是合適的 endpoint type

## Regional

![AWS API Gateway Endpoint Type - regional](/blog/images/serverless/AWS-serverless_API-Gateway-Endpoint_regional.png)

- 這個就很直觀，擺放的位置就是 region；基本上搭配 Route53 + CloudFront 是用來提供全球加速時很常見的組合設計

- 可像上圖搭配 multi-region 的設計，來提高服務可用性

## Private

![AWS API Gateway Endpoint Type - prirvate](/blog/images/serverless/AWS-serverless_API-Gateway-Endpoint_regional.png)

- 顧名思義，就是僅有內 VPC 內部可存取

- 這類型的 API Gateway，會提供一個 VPC endpoint 提供存取，也僅能透過 VPC endpoint 才能存取


## 其他注意事項

- API endpoint type 是可以變更的；但**無法將 private API 變更為 edge-optimized API**


Requqest/Response Cycle
========================

![AWS API Gateway - Requqest/Response Cycle](/blog/images/serverless/AWS_API-Gateway_Request-Response-Cycle.png)

當一個 request 進到 API Gateway 且 match 到某個 path & method 後(例如上圖的 `GET /first-api-test`)，那就會進入到 Request/Response Cycle，而在這個 cycle 中，共有四個階段，分別是：

1. Method Request

2. Integration Request

3. Integration Response

4. Method Response

在這四個階段，每一個階段都有其可以設定當 API 運行更為安全 & 有效率的設定可以套用，以下分成四個部份來繼續說明。

## Method Request

這部份的功能其實就像個 Gateway Keeper，作為 API request 進到 API Gateway 後的第一道的把關，所謂的把關，表示有些工作是可以在此階段完成的，例如：

- 設定 Query String Paramters & Request Headers

- 搭配 `Request Validator`，可設定要驗證 `Body` or `Query String Paramters` & `Request Headers`

透過這些驗證的設置，若是驗證不過的 API call，可以在 API Gateway 這邊就回應 Error，這個有問題的 request 就不需要傳送到 backend service 中，達到過濾有問題的 request 的效果。


## Integration Request

- Body Mapping Template 可用來檢查資料，甚至可以將 input data 先做好轉換(or 擷取 input data 中的一部分)再往後送；透過此機制，可以從 input data 只取得後面 Lambda Function 所需要的資料，將不需要的資料都過濾掉

- 主要用來觸發後面的 Mock Endpoint、Lambda Function、AWS Service、其他 HTTP endpoint .... 等等

以 Lambda 為例，其實 request 從 API Gateway 進入後，不論是帶 query string parameter & headers & body，經過了 `Method Request` & `Integration Request` 階段後，都只能透過 event 取值，而以 query string parameter 為例，實際上根本不會在 event 參數出現，因此就需要在 `Integration Request` 階段做個 mapping 的設計來將 query string parameter 的內容轉成 event 中的內容。


## Integration Response

- 這個階段可以將回傳的資料做轉換

- 用來設定不同的 status code 會需要 mapping 到的 header value 為何
> 有哪些 response header 需要定義在 `Method Response` 階段，這邊才會出現

- 這個階段的同時也有 Body Mapping Tempalte 可以設定，也可以用來做一層轉換，只回應給 client 需要的資料


## Method Response

- 用來定義 response 應該要有的樣子，一般不會設定 block，只是用來設定不同的 status code 應該要回應的資料模樣

- 可來設定當 status=200 時，需要回應哪些 header(實際 mapping 到什麼內容需要在第三個 `Integration Response` 階段設定)；也可以設定 response data 的格式驗證


如何在 API Gateway 中作到 Cross Account Call ?
============================================

若是目前有兩個 Account 分別是 prod & dev，希望從 prod Account 的 API Gateway 中，呼叫 dev Account 的 Lambda Function，應該要怎麼做?

其實概念很簡單，就是給權限而已，重點的部份有以下幾個：

- 在 dev Account Lambda Function 新增權限

- `source ARN` 設定 prod Account 的 API Gateway(要包含 Resource & Method 的完整 ARN)

- `Principal` 的部份需要設定為 `apigateway.amazonaws.com`

- `Allow Action` 設定為 `lambda:InvokeFunction`

當建立 method，指定的 Lambda Function 是來自另一個 AWS Account(需要使用完整的 Lambda Function ARN) 時，AWS console 就會跳出提示要使用者透過 CLI 來建立 permission，但其實重點就是上面的那幾個部份，所以要自己做也行。


Lambda Version & Alias
======================

Lambda 提供了 `Version` & `Alias` 的功能，一開始只有 Version 1，然後會有一個 `$LATEST` 的變數指到最新的 version，隨著 version 的更新，`$LATEST` 的指向也會跟著改指到最新的 version，這時候如果搭配 `Alias`，就可能變成下圖的樣子：

![AWS Lambda Version & Alias](/blog/images/serverless/AWS-serverless_Lambda-version-alias.png)

在上圖中的狀況是：

1. 目前這個 Lambda Function 共有四個版本，Version 1~4，因此 `$LATEST` 是指到 Version 4

2. 建立了一個 Alias，而這個 Alias 可以指向特定的 version，也可以同時指向兩個 version；以上圖的 `oldestNewest` 為例，就同時指向了 Version 1 & 4；此外還可以額外設定 traffic split，上圖會將流量以 50:50 的比例分散到同一個 Lambda Function 的不同 version

3. 有了 Alias 後，API Gateway 就可以指向 alias (在預設情況下，直接指定 Lambda 就表示指向其 `$LATEST` 所指向的 version)，接著透過特定 Resource + Method 進來的 request 就會以 50:50 的比例被分散流量

透過上圖的架構，就可以達到類似 Canary Deployment 的效果，讓一部分的流量可以先送到 Lambda `$LATEST` version，其他的留在原本的版本。

另外，若是希望可以動態的讓 API call 轉發到不同的 alias，可以透過 [Stage Variable](https://docs.aws.amazon.com/apigateway/latest/developerguide/stage-variables.html) 的方式，以類似 `versionTest:${stageVariables.lambdaAlias}` 的方式來設定 method 對應的 Lambda Function。


Canaray Deployment
==================

上面介紹透過 Alias 的方式可以達到類似 Canary Deployment 的效果，但在 API Gateway 中，Canary Deployment 就是其中一個功能，只是達成的方式比較不太一樣，以下簡單介紹如何使用 API Gateway 提供的 Canary Deployment 功能。

完整 Canary Deployment 的流程如下：

1. 在指定的 Stage 中(例如：`dev`)啟用 Canary 功能 (此時可以看到 **Stage’s Request Distribution** 的設定中，流向 canary 的比例是 0%)

2. 在 `Resource -> Method` 的 `Integration Request` 的設定中，調整 Lambda Function 的設定(改成新的 Lambda Function)

3. 接著執行 Deploy API，此時畫面會出現像是 `dev (Canary Enabled)` 的選項(如下圖)，表示這個修改會被視為 canary deployment
![AWS API Gateway - canary enabled](/blog/images/serverless/AWS-serverless_API-Gateway-canary-enabled.png)

4. 此時流量還不會切到 canary version 上，還是需要到 Stages 設定中，切到 `Canary` tab 去調整 `Stage’s Request Distribution` 的比例才行

5. 確認 canary version 沒問題，就可以透過 **Promote Canary** 的功能，將所有流量都切到 canary version 上

> 需要注意的是，Canary Deployment 在 API Gateway 中是設定在 `Stage` level 的，因此當啟用了 Canary 功能之後，指定的 stage 就會啟用 canary 功能，而不是只有特定的 Resource & Method 有啟用 canary 


API Caching
===========

API Gateway 可以提供 cache 的功能，但有以下幾點需要注意：

- 這功能不包含在 free tier 中，啟用就需要付費

- 可以設定 cache capaciry、是否要加密、TTL .... 等選項

- 僅支援 GET method


CORS(Cross Orgin Resource Sharing)
==================================

關於 CORS 的部份，目前已經有很多文章寫這個部份，可以參考以下連結了解：

- [[Day 27] Cross-Origin Resource Sharing (CORS) - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](https://ithelp.ithome.com.tw/articles/10251693)

[DAY05 - API串接常見問題 - CORS - 概念篇 (2) - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](https://ithelp.ithome.com.tw/articles/10267489)

API Gateway 若要處理 CORS 的問題，只要點選 `Enable CORS`，就會出現以下設定畫面：

![API Gateway - Enable CORS](/blog/images/serverless/AWS-serverless_API-Gateway-Enable-CORS.png)

相關的設定給進去，就會自動將所有設定都完成，包含連 `OPTIONS` 的部份都會處理好(透過 **MOCK integration** 完成的)，因此 pre-flight 的問題就可以一併解決了。


REST API v.s. HTTP API
======================

這邊提供 AWS 官網文件中，兩者的說明 & 差異部份，可以根據自己的需求選擇合適的項目：

- [API Gateway use cases - Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-overview-developer-experience.html)

- [Amazon API Gateway concepts - Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-basic-concept.html)

- [Choosing between HTTP APIs and REST APIs - Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html)

基本上，REST API 提供較為完整的功能(細部比較可以參考上面連結中的比較表)，但價格比較貴；HTTP API 則反之，不過因為功能沒這麼多的關係，latency 表線上也會相對好一些。


References
==========

- [Amazon API Gateway > Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)

- [Setting up REST API integrations - Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-integration-settings.html)

- [Setting up stage variables for a REST API deployment - Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/stage-variables.html)

- [Change a public or private API endpoint type in API Gateway - Amazon API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-api-migration.html)