---
layout: post
title:  "AWS SOA 學習筆記 - 模擬考筆記"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容後，進行模擬考時所留下的重點內容，有些內容在學習過程中並沒有學到但有出現在模擬考中，因此就把這一類的資訊整理起來放在同一篇方便閱讀"
date: 2022-03-26 03:26:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - SOA
---


CloudFront + S3
===============

- S3 bucket 設定成 website 之後，僅能透過 HTTP 提供檔案，且無法設定 OAI(Origin Access Identity)

- 必須要透過搭配 CloudFront，並限定使用者使用 HTTPS 協定，回源設定使用 HTTP， 才可以讓使用者存取 S3 的流量通通加密

- 若是 S3 已經設定為 website，還要作到僅能限制透過 CloudFront 存取，那就只能用 custom header 的方式來達成這個目標


IAM + CloudFormation
====================

- CloudFormation StackSet 可以在多個 AWS Account & Region 佈署資源

- 通常使用 CloudFormation StackSet 產生帳號都是為了將帳單分群組的目的而使用

- 若用在 AWS Organization level，就可以根據業務需求統整多個 AWS Account 

> 搭配 Microsoft AD(or AWS Directory Service) 是用來在不同 AWS Account 建立 IAM Role 的方式

- 若是使用 CloudFormation 時沒有完全足夠的權限，就會完全無法建立 AWS resource，而不會只建立一部分，另一部份無法建立


IAM
===

- cross-account 的存取權限設定，可以選擇搭配(or 不搭配) IAM Role 來實現，但不使用 IAM Role 有以下優點：
  - principal 依然運行在 trusted account 中
  - 不需要放棄目前的 IAM User permission 來切換成 IAM Role
> 通常這樣的設定場景常用於在其他 AWS Account 中複製共享資源的時候使用

- `Permissions Boundary` 是用來設定 identity-based policy 可以設定的權限範圍為何，並不是設定在 IAM policy 中(不論是 identity-based or resource-based)

- policy 的 statement 中，除了 `Resource`，可能也會有 `NotResource` 的字眼出現，需要注意一下



EC2 + CloudFormation
====================

- 傳送 `cfn-signal` 到 CloudFormation 不需要設定 IAM Role 相關權限，只要確認連外網路會通且 routing 有設定正確即可



RDS + CloudWatch
================

- RDS 啟用 Enhanced Monitoring 可以觀測到 OS level metric (預設存放 30 天)
> 提供 Free Memory、Active Memory、Swap Free、Processes Running、File System Used 等五項 metric 數據

- 上面的 metric 適合用來觀測 RDS 中不同 process 使用 CPU 的狀況

- 若是要搭配自己的 monitoring system，可以從 CloudWatch Log 中取得 Enhanced Monitoring 送出的 JSON output

- Performance Insight 僅用來收集 DB engine 相關資訊


Elastic Beanstalk
=================

- 提供 Blue/Green Deployment 的方式將環境分開 or 佈署測試環境用


Trusted Advisor
================

- 在 S3 的部份，主要是用來檢查是否有設定公開存取權限、logging & versioning ... 等設定，無法用來產生費用報表
> 費用報表需要透過設置 Cost Allocation Tag 並搭配 Cost Explorer 來看


VPC
===

- AWS Direct Connect 的網路需求條件：
  - 若是 1 Gb 線路，需要 single-mode fibre with a 1000BASE-LX (1310nm) transceiver；10 Gb 線路則需要 10GBASE-LR
  - 必須支援 BGP & BGP MD5 認證
  - 必須支援 802.1Q VLAN
  - port 上的 auto negotiation 必須關閉

- VPC 內部的 subnet 預設都是可以互通的 (不考慮服務需要使用 security group 開通的前提，因為這不屬於 subnet-level 的 ACL 設定)

- **VPC endpoint connection 是無法在 VPC 之外的地方生效**，因此即使透過 Direct Connect 連到 VPC，也無法透過 VPC endpoint 以 private 的方式連到 S3

- Direct Connect 之間的傳輸沒有加密，若需要加密的話，可以搭配 VPN

- 預設 Network ACL 若是不加任何 rule，則會允許所有 inbound/outbound 流量

- VPC subnet 若是沒指定 network ACL，就會套用 default Network ACL 的設定


EC2
===

## general

- `DisableApiTermination` 屬性無法阻止 ASG or 使用者移除 instance

- Enhanced Networking 使用 SR-IOV 技術，可將 PPS(packet per second) 在 VIF drvier 上提升到 20K 以上的水準

- `DiskReadBytes` 是屬於 instance store metric，若 EC2 instance 中使用的不是 instance store，這個 metric 就會一直是 0

- ASG 中可以在 step scale policy 中設定 new instance 的 warm up 時間，在 warm up 時間還沒結束前，instace metric 不會被收到 ASG group metric

- 以下的原因可能會造成 EC2 啟動後馬上被 terminate：
  - 到達 EBS volume limit
  - EBS snapshot 有問題
  - EBS volume 被加密但沒有對應的 key 可以解密
  - region level 中達到 EC2 limit

- t2 Unlimited 選項可以提供持續 burst 的能力，但超出的 credit 需要另外收費，若是面臨偶而的峰值運算能力需求，是個划算選擇
> burst performance 是無法調整的

- **RI(Reserved Instace)** 才是有折扣的選項；**Capacity Reservation** 只是確保有 VM capacity 可用而已，沒折扣 (兩者不同需要搞清楚)

- EC2 instance 偶而會有 AWS 安排的 event(可能重啟之類的)，這部份沒有辦法控制，但會預先被通知

- 關於 public IP on EC2 instance：
  - 最多只有一個 public IP address，即使有多個 network interface
  - 無法手動取消與 public IP 的綁定，因為 public IP 不屬於 AWS account (須改用 EIP)
  - 指定了 EIP，原有的 public IP 就會被釋放，兩者不會同時存在
  - 只有 default VPC 會預設派發 public IP，非 default VPC 預設不會派發 public IP

## AMI

- 被移除的 AMI 無法恢復 or 還原，僅能透過 EBS snapshot or EC2 instance 重建新的

- 要分享 AMI 到另一個 region 不需要分享 EBS snapshot(沒任何關係)，就複製 AMI 過去即可
> EBS snapshot 也可以被複製到另一個 region，但不與 AMI cross-region sharing 有直接關係

- 若是要分享加密的 EBS volume 到另一個 region，只有透過 customer-managed CMK 

- 若 EC2 instance 啟動時出現 `This AMI was copied from an AMI with a kernel that is unavailable in the destination Region` 錯誤，表示所在的 AWS region 不支援 Linux PV(paravirtual) AMI

## ASG

- 在 ASG 中可以透過設定 `instance protection`，避免 instance 在 scale in 階段時被移除

- `DisableApiTermination` 無法阻止 ASG 移除 EC2 instance

## Spot Instance

- 無法對 spot instance 設定 `termination protection`，但可以針對 spot interruption 做一些應對


## 常見錯誤

- `InsufficientInstanceCapacity`：表示該 AZ 已經沒有足夠容量可產生新的 EC2 instance；可以考慮換個 AZ or 更換 instance type

- `InstanceLimitExceeded`：已經超過使用額度(per region)，需要找 AWS support 開 ticket 提高額度


ELB
===

- ALB access log 會包含所有沒有轉發到 target 的 request，表示即使沒有 healthy target 也會有記錄；但不會包含定期發送的 health check request 相關的資訊

- 預設僅會將 request 送給 healthy target，但若是 ELB 後方 target group 中的 instance 全部都是 unhealthy 狀態，就還是會把 request 送過去給 unhealthy target

- metric `ActiveConnectionCount` 可用來表示目前連進來的 client 數量有多少

- 讓 ALB 可以正確運作的流程：
  - 設定 target group，指定 target 為 EC2 instance or Lambda Function ... 等
  - 使用 ALB 前，至少需要註冊一個(or 多個) listener 才可以開始使用，否則 request 會沒地方去；可以自訂各種不同的 rule 來決定 request 如何被轉送給 listener，而 Listener 就是要從已經設定好的 target group 中選擇
> default rule 沒有任何條件，若是 request 沒有符合任何條件，那就會套用 default rule 的設定

- metric `HTTPCode_ELB_4XX_Count` 是由 ELB 產生的，並非 ELB targets(源站)

- 當 NLB 建立後，無法關閉使用中的任何一個 AZ


EBS
===

- 在希望增加 gp2 效能但又維持相同成本情況下，可以使用 RAID 0

- 透過 AWS Data Lifecycle Manager 可以自動建立 EBS snapshot，也可以自訂保留 & 移除的時間

- 掛載 EBS volume 時若是指定錯誤的 device name，就會卡在 attaching 的狀態，需要換個 device name or reboot instance 才能解決此問題

- 若 EBS volume 顯示 `error` status，表示底層硬體出問題了，只能透過從 EBS volume 的備份還原來救回資料

- 移除 EC2 instance 時，若是啟用 `DeleteOnTermination`，會連同 EBS volume 一起移除，但這行為不會呼叫 `DeleteVolume` API，因此 AWS Config 在一段時間內還是會認定 EBS volume 還是存在的


S3
===

- 無法將 Snowball Edge 的資料直接存進 Glacier；必須搭配 Lifecycle policy 逐漸轉移才行
> 透過 CLI 就可以直接將檔案上傳到 Glacier

- 啟用 versioning 功能除了可以避免意外刪除之外，也可以防止惡意程式的竄改 or 移除(有原本的版本可恢復)

- 若是要使用 S3 作為 web site，S3 bucket 的名稱必須要跟 domain name 完全一樣，不論有無 subdomain 都必須完全一樣

- 可以直接將檔案傳到 S3 IA

- 透過 policy 中 condition `StringLike` 搭配 condition key `aws:Referer`，可以限定特定的 website 來存取

- S3 Access Log 可以記錄到 object-level 的存取歷史狀況，但 CloudTrail 僅能記錄到 bucket-level 的 API call，無法記錄到 object level 的部份；因此若是有法規稽核的需求，需要確認稽核的範圍來決定使用哪種 solution

- `Retain Until Date` 屬於 object-level 的設定，並非 bucket-level 設定(在 bucket 中設定的是 duration，生效範圍是所有 object)

- 同一個 object 的多個不同版本，可以對應不同的 retension 設定

- 關於檔案加密所需要加上的 header 資訊：
  - `'x-amz-server-side-encryption': 'aws:kms'`：搭配 AWS KMS 服務進行加密
  - `'x-amz-server-side-encryption': 'AES256'`：搭配 SSE-S3 進行加密 (這是唯一可以透過 AES256 加密的選項)

- 存取 S3 上加密後的檔案，若是沒有對應的 key 存取權限，會出現 4xx Error

- S3 偶而會出現 500 Internal Error，通常 retry 就會好，因此若是程式中存取 S3 遇到這類型的錯誤，可以設計 retry 機制來解決 (透過 CloudWatch 也可以監控此類型錯誤)

- Inventory Report 可以列出在 S3 bucket 的 replication failed object

- S3 RTC(Replication Time Control) 可提供 S3 replication 相關的 metric & 發送相關的事件通知；可用來監控超過 15 分鐘還無法完成複製的物件

- S3 replication events 只有在啟動 RTC(Replication Time Control) 的前提下才會送出

- bucket owner 對於其他 AWS account 傳到 bucket 的 object 是沒有權限的，建立 object 的 AWS account 必須提供權限才可以讓其他帳號存取 object 或是進一步設定權限

- Storage Gateway VM 不支援透過 VM snapshot or AMI 還原，只能重建

- 若要在 File Gateway 與 S3 建立一個 private connection，需要使用 VPC endpoint

- `S3 Access Points` 可用來簡化管理分享資料給眾多 application 同時使用的任務 (產生唯一的 hostname & 獨立的權限管理機制)

- 要調整 Storage Gateway 的 cache disk space，需要重建 Storage Gateway


EFS
===

- 要強制使用 EFS 儲存資料要加密，有以下方法：
  - 在 AWS Organization 中定義 SCP(Service Control Policy)
  - 在 IAM Identity-based policy 中，使用 IAM condition key `elasticfilesystem:Encrypted`，限定使用者僅能建立加密的 EFS filesystem


DB
===

## RDS

- 即使是 Aurora，最多也只能支援 5 個 `cross-region` read replica

- 若是要加密 application & RDS instance 的連線(encrypted at rest)，需要搭配 RDS 所提供的 SSL certificate 來完成，並非用 KMS

- 若是直接寫資料到 replica table，那就會破壞原本運行中的 replication 機制

- read replica 的 `max_allowed_packet` 參數設定不可以小於 source DB，否則會產生錯誤導致 replication 無法正確執行

- read replica 只能用在 transactional storage engine 上(例如：InnoDB)，nontransactional storage engine(例如：MyISAM) 是不支援的

- Multi-AZ 之間的資料複製是同步的，而 read replica 的資料複製是非同步的

- RDS 要提供 public access，需要啟用 VPC 中的 `DNS hostnames` & `DNS resolution` 兩個功能

- 設定 RDS 時無法直接指定使用哪個 subnet，而是要透過 subnet group 來設定

## DynamoDB

- DynamoDB 中可利用 DynamoDB Streams 搭配 AWS Lambda 與傳統的關聯式系統整合

- DynamoDB export function 無法在匯出資料到 S3 時順便帶上 `bucket-owner-full-control` ACL，但可以包含 `PutObjectAcl` 的權限讓 bucket owner 取得存取資料的權限


Elasticache
===========

- redis replication 是非同步的

- 只有在 multi-AZ 啟用且 automatic failover 關閉的情況下，才可以手動 promote read replica

- 當某個 redis read replica 要被 promote 成 primary 時，會選擇 replication lag 最少的節點來做這件事



CloudFront
==========

- 要限定使用者透過 CloudFront 存取內容，可透過 CloudFront Signed Cookie/URL 進行存取(可避免盜連)


Redshift
========

- 若要讓 Redshift cluster 的資料在不同的 region 被存取，可以透過將 cluster snapshot 複製到另一個 region 來達成 (Redshift 不支援 read replica 來達成這個目的)


CloudWatch
==========

- CloudWatch Agent 設定檔若是使用相同檔名，即使在不同 path，稍後啟動的 agent 會再啟動時將原有已經運行的 agent 設定取代；若要避免此問題，就必須使用不同的設定檔名

- 若要同時對 EC2 instance/system check 都發告警，須使用 metric `StatusCheckFailed`(**0** or **1**)；若是要判斷單獨情況(instance or system)，則是可以改用 `StatusCheckFailed_Instance` or `StatusCheckFailed_System`

- 若是要整合來自多個 EC2 instance 的監控數據，需要啟用 detailed monitoring(basic 不支援)

- 啟用 detailed monitoring 會同時在 dashboard 中加上一個 widget，並將每個 region 的所有 EC2 instance 的 CPU utilization metric 聚合起來顯示
> 每一個 region 一個 widget，並非所有 region 一個 widget

- 在 CloudWatch 中可以取得的 EC2 instance metric 有 CPU 使用率、EBS Read/Write Ops/Bytes、Network throughput、Network In/Out ... 等，但沒有 disk 剩餘多少容量的 metric

- Unified CloudWatch agent 可同時支援透過 StatsD & collectd 蒐集資料；其中 StatsD 同時支援 Linux & Windows 作業系統，但 collectd 僅支援 Linux

- CloudWatch EventBridge 是針對 AWS event 進一步的進行自動化操作，而非用來安排特定時間發生 event

- 沒有 `multi-region graph` 這一類型的 widget 可用


CloudFormation
==============

- 無法移除啟用 `termination protection` 的 CloudFormation stack，移除會失敗且狀態完全不會改變

- CloudFormation change sets 是用來檢視有哪些 AWS resource 會變更，實際執行後就會消失
> rollback 跟 change sets 完全沒關係，因為根本還沒有任何 AWS resource 有變更

- 在 CloudFormation Template 中要取得某個 resource 的屬性，要用 `!GetAtt` function

- `retain stacks` 是屬於 CloudFormation level 的設定，並非屬於 StackSets level 的設定；因此移除 StackSets 無法自行決定要不要保留 resource

- CloudFormation StackSets 設定後，必須要設定 trust relationship，才可以在 target account 上使用 StackSets 來建立 AWS resource

- 要將設定好的 Service Catalog 分享給其他帳號有以下幾種方式：(只有前面兩個可以用 share reference 的方式)
  - account-to-account sharing
  - organizational sharing
  - 透過 stack sets 將 catalog 佈署到其他帳號


Serverless
==========

- ALB 不會傳資料給 X-Ray；但會為每個 request 加上 `X-Amzn-Trace-Id` header 方便後續追蹤

- X-Ray 可與 S3、API Gateway、Lambda Function 整合


KMS
===

- 若要整合 on-premise KMS 與 AWS，可以使用 Cloud HSM，可同時支援 symmetric(同步) & asymmetric(非同步) 兩種加密方式


SSM (System Manager)
====================

- 要在 Hybrid 環境中設定 SSM 的流程如下：
  1. 設定 IAM Role
  2. 建立 managed-instance activation
  3. 在 on-premise VM(or server) 中安裝 TLS certicate
  4. 安裝 SSM agent

- Patch Manager 是 SSM 的其中一個功能，並非 AWS 獨立服務

- 移除 secret manager 中的 secret 會有 7 天的 pending deletion 設計，這段時間內無法新增同名的 secret；但若是透過 CLI 加上 `DeleteSecret` + `ForceDeleteWithoutRecovery` API，就可以跳過 pending deletion 直接完全移除 secret，就可以馬上重建一個同名稱的 secret

- 執行 `AWSSupport-TroubleshootS3PublicRead` Run Command 可用來協助偵測存取 public S3 bucket 時無法正常存取的詳細原因


ACM
===

- ACM 無法用來加密 EMail


Security
========

- 若是要追蹤 security group 任何的變更並得到自動通知，可以透過 CloudWatch EventBrige(Event) 搭配 AWS Inspector 來做網路設定規範檢查

- AWS MFA 認證不支援使用已經用過的硬體裝置，若使用硬體裝置就需要使用新的

- IAM identity-based policy 是用來設定在 IAM User/Group/Role 上，來決定誰可以有哪些 AWS API 存取的權限；而 resource-based policy 則是設定在 AWS resource 上，來決定這個 resource 可被誰存取

- 若是在同一個 AWS account 中，存取權限的評估是，只有沒有明確的 deny，且在 identity-based policy or resource-based policy 有其中一個設定 allow，那就可以存取

- 但若是 cross account request 發生時，identity-based policy 就必需要設定，僅有 resource-based policy 是不行的
> 應該是 AWS 預設不允許以 cross account 的方式存取 AWS resource API


Identity
========

- 將 AWS 帳號與外部認證服務(例如：AD)透過 SAML 進行整合時，需要 `Role`、`Audience`、`RoleSessionName` 三個屬性設定在 SAML Assertion 中

- 使用 AWS Organization with OU 可帶來的優點：
  - 集中管理帳號
  - 方便進行存取控制、合規、安全相關的管理
  - 在相同 OU 下，方便共享 AWS Account 之間的資源
  - 可根據企業需求，自動化 AWS Account/Group 的建立，並套用對應的 policy


Compliance
==========

- 使用 CloudTrail log file Integraity Validation 可以驗證 log file 在 CloudTrail 送出後是否有被變更 or 刪除，可透過 AWS CLI 進行檢查 

- AWS Config 可協助用來評估 AWS resource 的調整是否合規(規範自訂)；若是要自動確認 AWS resource 變更都有符合安全要求，需要借助 AWS Config & CloudTrail 兩個服務的協助

- AWS Config 僅會在 compliance status 變更時發送通知，發送通之後若是 compliance status 沒有任何變更(即使一直不符合規範)，也不會再發送新的通知

- 預設 Organization Member Account 可以讀取 organization trail，但無法修改 or 刪除其內容


Billing
=======

- Billing Alert 的選項位於 Account Preference，並非在 CloudWatch


其他
====

- AWS Health API 可提供在 Personal Health Dashboard 上提供的資訊，方便使用者可以透過程式根據需求進行自動化


推薦影片
=======

- [AWS re:Invent 2018: Your Virtual Data Center: VPC Fundamentals an d Connectivity Options (NET201) - YouTube](https://www.youtube.com/watch?v=jZAvKgqlrjY)

- [AWS ANZ Webinar Series - An Introduction to the Well Architected Framework - YouTube](https://www.youtube.com/watch?v=CeceqWuZ0Cg)

- [AWS re:Inforce 2019: Security Best Practices the Well-Architected Way (SDD318) - YouTube](https://www.youtube.com/watch?v=u6BCVkXkPnM)

- [AWS re:Invent 2018: [REPEAT 1] Become an IAM Policy Master in 60 Minutes or Less (SEC316-R1) - YouTube](https://www.youtube.com/watch?v=YQsK4MtsELU)

- [AWS re:Invent 2018: Monitor All Your Things: Amazon CloudWatch in Action with BBC (DEV302) - YouTube](https://www.youtube.com/watch?v=uuBuc6OAcVY)