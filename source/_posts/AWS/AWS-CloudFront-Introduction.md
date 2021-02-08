---
layout: post
title:  "[AWS] CloudFront 簡介"
description: "此篇文章是學習 AWS CloudFront 服務過程中所節錄下來的重點整理"
date: 2021-02-07 10:00:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CloudFront
  - CDN
---


什麼是 AWS CloudFront ?
==================

CloudFront 其實就是 CDN 的服務，目的就是透過 edge server 來就近提供相關資源給使用者，讓使用者的體驗更好。

之前在準備 CSA 的時候有寫了關於 CloudFront 的介紹，這邊就不多做介紹了，可以[連結到這篇文章](https://godleon.github.io/blog/AWS/AWS-CSA-associate-Advanced-Networking/#CloudFront) 看一下關於 CloudFront 的簡介。

以下就針對上面文章沒提到，但是在閱讀官網文件時所看到的重點，把一些重要的資訊記錄下來，藉此來更了解 CloudFront 這個服務。


- 預設檔案 cache 在 edge 的時間為 24 hours(default expiration time)，超過就會過期，最短可以設定為 0 second，但沒有 maximum expiration time 可以設定


Use Cases
=========

通常 CloudFront 除了 cache 靜態資源之外，還可能會有以下幾個常見的使用案例：

- 提供 VOD(video on demand) 的串流服務，支援像是 MPEG DASH, Apple HLS, Microsoft Smooth Streaming, CMAF ... 等協定；並且可協助廣播串流資訊，大幅降低 origin server 的負擔

- CloudFront 可以與 origin 用 HTTPS 的方式回源，這樣 end-to-end 的傳輸就已經是加密的，此外還可以額外加上 `field-level encryption`，但需要先上傳 public key 到 CloudFront，並搭配 origin 的 private key 來解密，讓機敏資料保護可以作到更進一步的安全提昇

- 在 edge 端的客製化，例如：當 origin server 有問題時，直接從 edge 回應 HTTP error message；或是使用個 function 在 request 送回 origin server 之前，作為認證 & 授權存取的管理


Regional Edge Caches
====================

雖然 CloudFront 會 cache 靜態檔案，但其實不會放很久，若是資料有一段時間沒被存取，cache 很容易就會失效，因此除了我們一般認知的 CloudFront Edge server 之外，CloudFront 還多了一層稱為 `Regional Edge Caches` 的機制來提供更大的快取空間，首先看一下架構圖：

![CloudFront Regional Edge Caches](/blog/images/aws/CloudFront/aws-cloudfront-regional-edge-caches.jpg)

運作原理說明如下：

- 原本的架構中，所有 cache 都放在 edge 端；但 Regional Edge Caches 的出現，變成了 Edge Location 與 origin server 之間的 cache

- 使用者送出 HTTP reuqest 後，當 origin server 回應，會同時存在於 Regional Edge Caches & Edge Location 中

- Regional Edge Caches 的儲存容量更大，可以 cache 更多且更久的 object (等於是第二層 cache)
 
- 透過 Regional Edge Caches，可以提昇 Edge Location 運作的效率，回源時可以只回到 Regional Edge Caches 而非 origin server

但還是有以下幾點需要注意：

- HTTP method `PUT/POST/PATCH/OPTIONS/DELETE` 都會從 edge location 直接回到 origin server，不會經過 regional edge caches

- dynamic request 也是直接回到 origin server，不會經過 regional edge caches

## 有哪些 Edge Location? IP 是多少?

AWS 會主動更新 Edge Location 的 IP 清單，可以參考[此網頁](https://docs.aws.amazon.com/general/latest/gr/aws-ip-ranges.html)，可以取得一個[完整的檔案列表](https://ip-ranges.amazonaws.com/ip-ranges.json)，以 JSON 形式發布。

而這些 IP 可以在 ALB 設定 source IP whitelisting，在 TCP Layer 4 這一層做保護，比起設定 customized HTTP header，我覺得是更安全的作法。

若是希望可以主動收到 Edge Location IP 更新通知，可以訂閱 SNS Topic(`arn:aws:sns:us-east-1:806199016981:AmazonIpSpaceChanged`) 來取得主動通知，並搭配串接後續的自動化流程，更新 ALB 的 allowed source IP

收費
====

 基本上 CloudFront 就收三個費用：

 - 如果靜態資源有放在 S3，那就會收 S3 儲存空間的費用

 - 將資料從 Edge Location 傳送給 client 的傳輸費 (對應帳單中的費用名目為 `region -DataTransfer-Out-Bytes`)

 - Edge Location 回到 origin server 所花費的流量 (對應帳單中的費用名目為 `region -DataTransfer-Out-OBytes`)

> 如果有使用 `field-level encryption` 或是使用 [`Origin Shield`](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) 作為額外的一層 cache layer，就還會有額外的費用產生 



實作範例
=======

## 以 S3 作為 Origin Server

### S3 設定須注意事項

- S3 bucket 預設是 private，對於要接受回源的物件檔案，需要修改提供 **public read** 的權限

- 可以只開放特定 object 的 public read 權限，也可以是特定 path or 整個 bucket

- 如果想要限制只有特定的人可以透過 CloudFront 存取 S3 的內容，可以參考[此篇文章(Serving private content with signed URLs and signed cookies)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html)


### CloudFront 設定須注意事項

- 在 **Origin Domain Name** 的部份，可以直接選擇的有 `S3 Bucket`、`ELB`、`MediaPackages`、`MediaStore`，而從這幾個選項就知道 CloudFront 跟這幾個服務的整合度是很好的 (例如：API Gateway 在畫面上就無法選擇到)

- **Default Cache Behavior Settings** 會有以下幾種行為：
  - 尚未 cache 的內容，就會把 request 轉送到 S3
  - 允許 client 透過 HTTP/HTTPS 協定存取資料
  - CloudFront edge location 會負責回應 client
  - 將資料 cache 在 edge location 24 小時
  - 僅會將基本的 header 資訊轉送回 origin server，而且 cache 時不會以這些 header value 為基礎進行 cache
  - 回源時不會包含 cookie & query string (**因此 origin server 收不到 cookie & query string 的資料**；但其實 S3 也不處理 cookie，但可能會處理某些 query string)
  - 所有人都可以透過 CloudFront 存取放在 S3 的內容
  - **不會自動對傳輸內容進行壓縮**

- **Distribution Settings** 中需要注意的部份：
  - **Price Class** 預設是 `Use All Edge Locations`，若是選擇其他選項，那就只有特定幾個區域的使用者可以享受到 CDN 的功能
  - **Default Root Object** 可以指定預設 object，例如：`index.html`，如此一來 client 連到 CloudFront domain 時，不須指定 object path 就會是預設的 object

### 問題排除

在一開始設定好 CloudFront & S3 後，瀏覽網頁竟然出現了 `HTTP 307 Temporary Redirect`，當下很疑惑，以為透過 CloudFront 取得靜態檔案，應該會由 CloudFront 負責回源到 S3 取得相關資源後，cache 並回應 client，但事實並非如此，因此上網查訊了一下，發現了以下文章：

[Troubleshoot HTTP 307 errors in Amazon S3](https://aws.amazon.com/premiumsupport/knowledge-center/s3-http-307-response/?nc1=h_ls)

原因的解釋為：

> After you create an Amazon S3 bucket, it can take up to 24 hours for the bucket name to propagate across all AWS Regions. During this time, you might receive the 307 Temporary Redirect response for requests to regional endpoints that aren't in the same Region as your bucket. For more information, see Temporary request redirection.

因此可以知道，在前 254 hours 內，透過 CloudFront 存取 S3 origin server 的資源，有可能會得到 HTTP 307 的訊息，但這問題會隨著 S3 資訊被同步到所有 region 後而消失。



參考資料
=======

- [What is Amazon CloudFront? - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)

- [【每周快報】1013-1021 AWS 服務更新 – eCloudture](https://www.ecloudture.com/weekly_news-1013-1021-aws-service-update/)

- [Troubleshoot HTTP 307 errors in Amazon S3](https://aws.amazon.com/premiumsupport/knowledge-center/s3-http-307-response/?nc1=h_ls)