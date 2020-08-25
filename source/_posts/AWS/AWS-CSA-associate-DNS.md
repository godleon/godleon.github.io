---
layout: post
title:  "AWS CSA Associate 學習筆記 - Route 53"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 Route 53 服務所可以提供的功能(主要為 Routing Policy 的介紹)"
date: 2020-06-03 06:10:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - Route 53
---


DNS 101
=======

- **Route53 預設有 50 個 domain 的限制，但可以與 AWS support 聯繫取消此限制**

## 常見的 DNS Resource Record

常見的 DNS Resource Record 有 SOA/NS/A/CNAME/MX/PTR ... 等等，可參考以下文章的說明：

- [DNS資源紀錄(Resource Record)介紹](http://dns-learning.twnic.net.tw/bind/intro6.html)

- [支援的 DNS 記錄類型 - Amazon Route 53](https://docs.aws.amazon.com/zh_tw/Route53/latest/DeveloperGuide/ResourceRecordTypes.html)


## cname & alias 的差異

這個部份可以參考以下連結：

- [What is the difference between CNAME and ALIAS records? – NS1 Help Center](https://help.ns1.com/hc/en-us/articles/360017511293-What-is-the-difference-between-CNAME-and-ALIAS-records-)

- [Differences Among A, CNAME, ALIAS, and URL records - DNSimple Help](https://support.dnsimple.com/articles/differences-between-a-cname-alias-url/)



Route53 Routing Policy
======================

目前 Route53 支援的 routing policy 有以下幾種：

- Simple Routing

- Weighted Routing

- Latency-based Routing

- Failover Routing

- Geolocation Routing

- Geoproximity Routing (Traffic Flow only)

- Multivalue Answer Routing


## Simple Routing

![Route53 - Simeple Routing](/blog/images/aws/DNS_Simple-Routing.png)

- Simple Routing 可以讓使用者對單一筆 record 設定多個 IP，並以 random 的方式回應結果(還要另外考慮 TTL 的影響)

- 無法搭配 health check 機制


## Weighted Routing

![Route53 - Weighted Routing](/blog/images/aws/DNS_Weighted-Routing.png)

- 可以自訂比例，設定某百分比的 DNS 查詢流量回應某個 record

- 百分比跟 record 並非一對一的，在同一個百分比設定下可以有多筆對應的 record (random 回應)

- 可以設定為每一筆 record 設定 health check 並搭配 EC2 instance 的 health check 功能

- 若 health check 失敗，Route53 就不會回應該筆 record

- health check 失敗可以搭配 SNS 通知讓負責人員知道


## Latency-based Routing

![Route53 - Latency-based Routing](/blog/images/aws/DNS_Latency-Based-Routing.png)

- 設定時，每一筆 record 加入時，AWS 會自動偵測合適的 region 回應(也可以自己選)；完成後，同樣的 domain name 在不同的 region 會回應不同的 record

- 使用者查詢時，AWS 會在設定的 region 中對使用者延遲最低的 region 進行回應

- 以上圖為例，來自南非的使用者就會由 eu-west-2 來回應，因為延遲比 ap-southeast-2 小多了


## Failover Routing

![Route53 - Failover Routing](/blog/images/aws/DNS_Failover-Routing.png)

- Failover Routing 用來支援 active(primary) / passive 的設計；當 active site 出問題，Route53 可以回應 passive site record

- 需要搭配 health check 的設定才可以使用

- 當 active(primary) site 的 health check 失敗時，Route53 就會改成回應 passive site record


## Geolocation Routing

![Route53 - Geolocation Routing](/blog/images/aws/DNS_Geolocation-Routing.png)

- 可以根據使用者所在的位置回應指定的 record

- 設定時可以指定使用者所在的五大洲 or 國家


## Geoproximity Routing (Traffic Flow only)

- 這是進階版的 DNS traffic policy 設定，因為實在太複雜，認證考試基本上不會出現

- policy 設定時，除了可以使用使用者所在位置作為依據，連 resource 所在的位置也可以拿來作為判斷條件

- 透過 Route53 Traffic Flow 的設定(也必須搭配 traffic flow)，可以將各種條件結合起來成為複合型的判斷，因此可以作到很細膩的控制


## Multivalue Answer Routing

其實就是 Simple Routing，只是可以加上 health check 的功能



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Amazon Route 53 - Amazon Web Services](https://aws.amazon.com/route53/)

- [AWS route53 multivalue 筆記 - Kakashi's Blog](https://kkc.github.io/2017/07/23/aws-route-53-multivalue/)