---
layout: post
title:  "AWS CSA Associate 學習筆記 - Advanced Networking"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 Route 53、CloudFront ... 等網路相關進階服務介紹 & 說明"
date: 2020-06-03 06:10:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - Route 53
  - CloudFront
---


Route 53
========

## DNS 101

- **Route53 預設有 50 個 domain 的限制，但可以與 AWS support 聯繫取消此限制**

### 常見的 DNS Resource Record

常見的 DNS Resource Record 有 SOA/NS/A/CNAME/MX/PTR ... 等等，可參考以下文章的說明：

- [DNS資源紀錄(Resource Record)介紹](http://dns-learning.twnic.net.tw/bind/intro6.html)

- [支援的 DNS 記錄類型 - Amazon Route 53](https://docs.aws.amazon.com/zh_tw/Route53/latest/DeveloperGuide/ResourceRecordTypes.html)


### cname & alias 的差異

這個部份可以參考以下連結：

- [What is the difference between CNAME and ALIAS records? – NS1 Help Center](https://help.ns1.com/hc/en-us/articles/360017511293-What-is-the-difference-between-CNAME-and-ALIAS-records-)

- [Differences Among A, CNAME, ALIAS, and URL records - DNSimple Help](https://support.dnsimple.com/articles/differences-between-a-cname-alias-url/)


## Hosted Zones

- 每個 zone 所代表的就是一個 domain

- 每個 zone 裡面包含的就是那個 zone 所擁有的 DNS record 設定

- zone 有分為 `public` & `private` 兩種：
  - **pulbic hosted zone**：面向 Internet 的，別人只要有使用正確的 DNS server 設定就可以查詢的到
  - **private hosted zone**：只有在 VPC 中有效(跨 VPC 也可以)，在 VPC 中的 instance 可以正確的查詢到 DNS record 設定
> **private hosted zone 不會回應來自 VPC 之外的 DNS 查詢**

- 每個 zone 建立後都會預帶 `NS(name server)` & `SOA(start of authority)` 的設定


## Record Sets

- Record Set 就是將 domain 轉換為指定 IP address 的設定

- 設定中包含了多個選項：`Record Type`(例如：A/AAAA/CNAME/MX ... etc)、`Standard/Alias`、`Routing Policy`、`Evaluate target health`(可用來監控 application 的健康狀況，並指定觸發特定動作)

- **Alias Record Set** 所設定的並非 IP address，而是指向特定 AWS resource 的 alias 設定


## Route53 Routing Policy

目前 Route53 支援的 routing policy 有以下幾種：

- Simple Routing

- Weighted Routing

- Latency-based Routing

- Failover Routing

- Geolocation Routing

- Geoproximity Routing (Traffic Flow only)

- Multivalue Answer Routing


### Simple Routing

![Route53 - Simeple Routing](/blog/images/aws/DNS_Simple-Routing.png)

- Simple Routing 可以讓使用者對單一筆 record 設定多個 IP，並以 random 的方式回應結果(還要另外考慮 TTL 的影響)

- 無法搭配 health check 機制

### Weighted Routing

![Route53 - Weighted Routing](/blog/images/aws/DNS_Weighted-Routing.png)

- 可以自訂比例，設定某百分比的 DNS 查詢流量回應某個 record

- 百分比跟 record 並非一對一的，在同一個百分比設定下可以有多筆對應的 record (random 回應)

- 可以設定為每一筆 record 設定 health check 並搭配 EC2 instance 的 health check 功能

- 若 health check 失敗，Route53 就不會回應該筆 record

- health check 失敗可以搭配 SNS 通知讓負責人員知道

### Latency-based Routing

![Route53 - Latency-based Routing](/blog/images/aws/DNS_Latency-Based-Routing.png)

- 設定時，每一筆 record 加入時，AWS 會自動偵測合適的 region 回應(也可以自己選)；完成後，同樣的 domain name 在不同的 region 會回應不同的 record

- 使用者查詢時，AWS 會在設定的 region 中對使用者延遲最低的 region 進行回應

- 以上圖為例，來自南非的使用者就會由 eu-west-2 來回應，因為延遲比 ap-southeast-2 小多了

### Failover Routing

![Route53 - Failover Routing](/blog/images/aws/DNS_Failover-Routing.png)

- Failover Routing 用來支援 active(primary) / passive 的設計；當 active site 出問題，Route53 可以回應 passive site record

- 需要搭配 health check 的設定才可以使用

- 當 active(primary) site 的 health check 失敗時，Route53 就會改成回應 passive site record

### Geolocation Routing

![Route53 - Geolocation Routing](/blog/images/aws/DNS_Geolocation-Routing.png)

- 可以根據使用者所在的位置回應指定的 record

- 設定時可以指定使用者所在的五大洲 or 國家

### Geoproximity Routing (Traffic Flow only)

- 這是進階版的 DNS traffic policy 設定，因為實在太複雜，認證考試基本上不會出現

- policy 設定時，除了可以使用使用者所在位置作為依據，連 resource 所在的位置也可以拿來作為判斷條件

- 透過 Route53 Traffic Flow 的設定(也必須搭配 traffic flow)，可以將各種條件結合起來成為複合型的判斷，因此可以作到很細膩的控制

### Multivalue Answer Routing

其實就是 Simple Routing，只是可以加上 health check 的功能



CloudFront
==========

CloudFront 就是 AWS 透過佈建在全世界各地的 edge server 所提供的 CDN 服務，有以下幾個重點知識需要了解：

- AWS 在全世界各地建立了 edge server，確保大多數的使用者都有一個相對源頭較近的地方可以取得資料

- **Edge Location**：實際就是快取靜態資源的地方(edge server)，跟 Region & AZ 兩個是不同的概念

- **Origin**：這就是靜態資源的原始儲存位置，當使用者對 edge server 要求檔案，但 edge server 沒有時，edge server 就會回源到此處取得所需的檔案(可以是 S3、EC2 instance、ELB + EC2 instance)
> Edge server 會將回源取到的資料進行 cache，下一位使用者來存取時，就不需要再度回源了
![S3 CloudFront](/blog/images/aws/S3_CloudFront.png)

- **Distribution**：這其實就是 CDN 服務本身所提供的 endpoint，搭配其智慧化的 DNS 解析能力，可以讓使用者取得離自己最近的 edge server 進行存取，因此可以視為 edge location collection；而 AWS CloudFront 目前支援兩種資源的發佈
  
  - **Web Distribution**: 通常是一般網站使用的靜態資源，像是 image, css, javascript, music ... 都屬於此類；也包含透過 HTTP 協議傳輸的視訊協定，例如：HTTP Live Streaming(HLS), Dynamic Adaptive Streaming over HTTP (DASH), Microsoft Smooth Streaming (MSS), or HTTP Dynamic Streaming (HDS) .... 等等。
  
  - **RTMP**: 用於視訊串流類的服務中(但這通常會根據服務型態不同，還需要額外考慮視訊延遲的問題)
  > CloudFront 將會在 2020 年底停止 RTMP 的支援

- Edge Locations 並非只是用來拉取資料，也會有寫入資料的應用場景(例如：**S3 Transger Acceleration**)

- 快取在 edge location 的靜態資源，會有 TTL(Time to Live) 的屬性，不會永久存在

- 為了讓使用者可以很快取得最新的資料，可以執行 CDN 清理的工作或是讓快取失效(invalidate)，但因為需要重新回源，因此會有額外費用產生

## 優點

- 使用者可以從最近的 edge server 拉資料，速度快多了，體驗自然大幅上升

- 從 origin service 傳出的資料變少了，成本下降(CDN 傳輸單位成本比 EC2 instance 低)


## 進階設定

- `Restrict Viewer Access(Use Signed URLs or Signed Cookies)`：開啟此功能，可以限定使用者必須要登入後取得特定域名的 cookie 才可以存取 CDN 上的快取資源；或是讓資源在特定時間(例如：60 mins)過後就失效


## 可能發生的效能問題

- CDN 的運作基礎在於提供使用者一個最近的 edge server 進行存取，但要提供出這樣的資訊，就必須要進行完整的 DNS 查詢流程，因此是 DNS 查詢很慢，也是會影響到使用 CDN 的效能的

- 若是在 url 帶上 query string，那 edge server 就不會回應了，而這樣的 request 就會回到 origin server，當然效能自然就會差很多

- 若要要提昇 CDN 效能，比較直接的作法就是拉長 cache 的過期時間




References
==========

## Route 53

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Amazon Route 53 - Amazon Web Services](https://aws.amazon.com/route53/)

- [AWS route53 multivalue 筆記 - Kakashi's Blog](https://kkc.github.io/2017/07/23/aws-route-53-multivalue/)


## CloudFront

- [以 Amazon CloudFront 加速您的網站 - Amazon Simple Storage Service](https://docs.aws.amazon.com/zh_tw/AmazonS3/latest/dev/website-hosting-cloudfront-walkthrough.html)

- [FAQs | CDN, Zone Apex, Edge Cache | Amazon CloudFront](https://aws.amazon.com/cloudfront/faqs/?nc=sn&loc=6)