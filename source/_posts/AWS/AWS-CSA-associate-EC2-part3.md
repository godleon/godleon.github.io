---
layout: post
title:  "AWS CSA Associate 學習筆記 - EC2(Elastic Compute Cloud) Part 3"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為與 EC2 相關的服務，會分成三篇文章來記錄，此為第三部份，包含了 Bootstrap Scripts、Instance Metadata、Placement Group、WAF、EFS、FSx、FSx for Lustre ... 等內容"
date: 2020-06-02 05:20:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - EC2
  - EFS
  - FSx
  - FSx for Lustre
  - WAF
---



Bootstrap Scripts
=================

- EC2 instance 在設定時可以透過 `user-data` 欄位指定 bootstrap script，目的是讓 EC2 instance 在佈署完成後，進行指定的工作

- 這功能可以與其他服務搭配，讓一些特殊需求的自動化容易些
> 例如：EC2 instance 佈署完成後，呼叫 meta data server 取得 public ip，放到 S3；S3 event 觸發 Lambda function 將資料讀出後，寫入 RDS



Instance Meta Data
==================

- 每個 EC2 instance 可以透過一個特殊的 IP 位址 `169.254.169.254` 取得 instance meta data (其實就是關於此 instance 的資訊)

- 透過 `curl http://169.254.169.254/latest/meta-data/` 可以列出可取得哪些 meta data，大概包含 iam/instance-action/instance-id/local-hostname/local-ipv4/mac/metrics/network/public-hostname/public-ipv4 .... 等等，可取得不少資訊

- 透過 `curl http://169.254.169.254/latest/user-data` 就可以取得在 instance 設定時候填入的 user-data 資訊



EFS(Elastic File System)
========================

首先看看 AWS 對 EFS 的定義：

> Amazon Elastic File System (Amazon EFS) provides a simple, scalable, fully managed elastic NFS file system for use with AWS Cloud services and on-premises resources. It is built to scale on demand to petabytes without disrupting applications, growing and shrinking automatically as you add and remove files, eliminating the need to provision and manage capacity to accommodate growth.

> Amazon EFS offers two storage classes: the Standard storage class, and the Infrequent Access storage class (EFS IA). EFS IA provides price/performance that's cost-optimized for files not accessed every day. By simply enabling EFS Lifecycle Management on your file system, files not accessed according to the lifecycle policy you choose will be automatically and transparently moved into EFS IA.

> Amazon EFS is designed to provide massively parallel shared access to thousands of Amazon EC2 instances, enabling your applications to achieve high levels of aggregate throughput and IOPS with consistent low latencies.

> Amazon EFS is a regional service storing data within and across multiple Availability Zones (AZs) for high availability and durability. Amazon EC2 instances can access your file system across AZs, regions, and VPCs, while on-premises servers can access using AWS Direct Connect or AWS VPN.

大概可以整理出以下重點：

- 沒有預付費用， 只須根據儲存容量 & 傳輸費用付費

- 可以擴充到 PB 等級大小

- 跟 S3 相同，可協助將不常存取的檔案移到 IA tier 中

- 資料會在指定的 region 中分散不同 AZ 進行存放

其他重點資訊：

- EFS 目前支援 NFSv4

- 可以支援同時數千個 NFS 連線

- 提供 read-after-write consistent，表示寫入後馬上可以讀取到

- **EFS 也有提供 EFS-IA 用來存放不常存取的資料，藉此降低資料儲存成本(但 lifecycle policy 最長僅能設定為 90 天)**



FSX & FSx for Lustre
====================

因為對 Windows 不是很有興趣，這邊就重點整理一下吧!

## FSx (for Windows File Server)

- FSx 可以想像一下就是 Windows 中的 SMB 協定提供出來的檔案系統，所以其實跟 EFS 之於 Linux 是差不多相同意思

- FSx 同樣也是全託管(fully managed)服務，資料的 HA 也會自動保證(同 region 跨 AZ，也可以選擇 single AZ)

- **支援 Microsoft Distributed File System (DFS)**

## FSx for Lustre

- 這是強化版的共享檔案系統，可同時被 Windows & Linux 使用，總之就是**高效能 + 可擴展**

- 可以提供 low latency、high throughtput/IOPS 等特性，用在像是機器學習、HPC .... 這類資料存取繁重的應用是很合適的

- 可與 S3 進行整合，資料處理完就可以直接存放到 S3

- **不支援 Microsoft Distributed File System (DFS)**



Placement Group
===============

試想一下，如果有些應用需要 VM 之間網路延遲很低，在地端的 data center 會怎麼做?
> 很簡單囉，只要把 VM 起在同一台機器，或是接在同一台網路設備的不同機器，不要讓他們之間距離太遠，問題就搞定了!

那如果希望某台實體機器掛掉了，不會影響到太多的 VM 運作，要怎麼做呢?
> 這其實也不難，只要將 VM 起在不同的實體機器就可以了!

簡單來說，`Placement Group` 這個功能可以讓使用者根據需求決定"你想要把 EC2 instance 擺在哪裡?"這件事情

而 AWS 提供了三種 placement group，分別是 `cluster`、`spread`、`partition`，使用者可以根據需求自行選擇：

## Cluster

![AWS EC2 Placement Group - Cluster](/blog/images/aws/EC2_PlacementGroup-cluster.png)

- instance 一定會在同一個 AZ 內

- 會盡可能的將 instance 放在一起，甚至同一個機櫃上，以取得更低的網路延遲

- 可以提供 instance 之間的網路有更低的延遲 & 更高的吞吐量

- 承上三點，所以選擇 instance type 時，至少要選擇有 10Gb 以上網路的 instance type 才能享受到 placement group 的所帶來的優勢

- 通常用在需要快速完成的 big data 相關應用；或是需要極低的 network latency & high throughput 的其他應用

- **增加新的 EC2 intance 到 cluster placement group，可能會因為當下 VM 所在的硬體已經資源不足，出現 `insufficient capacity error`；此時的解決方法就是停止並重新啟動 VM，讓 VM 移動到不同的硬體上啟動，就可以解決此問題(新的 VM 就可以順利增加了)**

## Spread

![AWS EC2 Placement Group - Spread](/blog/images/aws/EC2_PlacementGroup-spread.png)

- instance 不一定會在同一個 AZ 內；在同一個 AZ 中的 insance 也不會在相同的實體機器上

- 不同 placement group 的 instance 絕對不會在同一個 rack 上，因此某個 rack 發生問題不會導致所有 instance 無法使用

- 若是需要在同一個 AZ 中的 HA，可以透過設定 spread placement group 達成

- 隔離的單位為單一 instance

- **每個 spread placement group 在每個 AZ 最多只能有 7 個 running instance** (因此若是 EC2 instance 太多，很容易碰到上限)

- 最大化 HA 的選項

## Partition

![AWS EC2 Placement Group - Partition](/blog/images/aws/EC2_PlacementGroup-partition.png)

- instance 不一定會在同一個 AZ 內

- 每個 AZ 中最多只能有 7 個 partition；最多總共可以有 100 個 EC2 instance

- 運作方式 & 效果 spread type 類似，只是單位是 instance group (多個 instance 形成一個名稱為 `partition` 的單位)

- 從 EC2 instance 的 metadata 中可以取得 partition 配置的相關資訊

- 不同的 partition 不會被安排在同一個 rack 上

- 隔離的單位為 partition(也就是一群 instance)

- 通常像是 HDFS、HBase、Cassandra、Kafka 可以利用 partition placement group 來額外設定 hardware-level HA

## 其他

- 同一個 AWS 帳號下的 placement group 名稱不能重複

- 只有某些類型的 instance 才可以放到 placement group 中(例如：compute optimized / GPU / memory optimized / storage optimizted ... 等等)

- 若要將 instance 放到 cluster placement group 中，建議是同質性的 instance (若是同樣的 instance type 更好)

- 不同的 placement group 無法進行合併

- 若是將 instance 移到 placement group 中，必須先切到 `stopped` 狀態；而且這件事情目前僅能透過 AWS CLI or SDK 完成，無法在 console 完成這件事情

- 若是遇到 `insufficient capacity error`，解決方法是停止所有在 placement group 中的 instance，再重新啟動



WAF(Web Application Firewall)
=============================

首先以下是 AWS 原廠對 WAF 服務的說明：

> AWS WAF is a web application firewall that helps protect your web applications or APIs against common web exploits that may affect availability, compromise security, or consume excessive resources. AWS WAF gives you control over how traffic reaches your applications by enabling you to create security rules that block common attack patterns, such as SQL injection or cross-site scripting, and rules that filter out specific traffic patterns you define. You can get started quickly using Managed Rules for AWS WAF, a pre-configured set of rules managed by AWS or AWS Marketplace Sellers. The Managed Rules for WAF address issues like the OWASP Top 10 security risks. These rules are regularly updated as new issues emerge. AWS WAF includes a full-featured API that you can use to automate the creation, deployment, and maintenance of security rules.

WAF 若要是在地端實現，要購買的設備可是相當昂貴的；此外，不論是地端 or 雲端，要善用 WAF 的功能，也要清楚 WAF 的原理 & 相關知識，使用起來才可以符合自己所需，以下是 AWS WAF 的特點：

- WAF 主要就是要監控被轉發到 CloudFront、ALB、API Gateway ... 等服務的 HTTP & HTTPS 的請求
> 這表示 AWS AWF 可以掛在 CloudFront、ALB、API Gateway 這幾個地方前面

- 可以根據以下條件設定讓某些 request 通過 or 被阻擋：

    - 來源 IP address

    - query string parameters (可搭配正規表示式)

    - 來源國家

    - 在 HTTP header 中的值

    - request 長度

    - SQL Injection (惡意的 SQL 內容)

    - Cross-Site Scripting (惡意的 script 內容)

- 上面三個在 WAF 後方的服務，可以自行決定要接受 or 拒絕 quest，跟 WAF 的設定是分開的


AWS WAS 在過濾流量上提供三種模式供使用者選擇：

- 除了指定的條件外，允許所有 request 通過 (黑名單設計)

- 除了指定的條件外，拒絕所有 request 通過 (白名單設計)

- 不阻止 request，僅針對指定的條件進行計數 (通常用於觀察之用)


## 考試重點

- WAF 可以阻擋 Layer 7 的攻擊，例如：`SQL Injection`、`Cross Site Scripting`(**可以設定 rate-based rule**)；而 AWS Shield 則是阻擋 Layer 3 & 4 的攻擊，例如：DDoS

- 若要需要阻止惡意的來源 IP 位址，可以從哪些服務著手：
  - Network ACLs
  - AWS WAF

- AWS Shield 是個預設會啟用的服務，有 Standard & Advanced 兩種等級



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Retrieving instance metadata - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html)

- [Amazon Elastic File System (EFS) | Cloud File Storage](https://aws.amazon.com/efs/)

- [Placement groups - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)

- [AWS WAF - Web Application Firewall - Amazon Web Services (AWS)](https://aws.amazon.com/waf/)