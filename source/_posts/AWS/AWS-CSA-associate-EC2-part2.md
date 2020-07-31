---
layout: post
title:  "AWS CSA Associate 學習筆記 - EC2(Elastic Compute Cloud) Part 2"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 EC2 服務，會分成三篇文章來記錄，此為第二部份，包含了 EBS、Snapshot、Volume Encryption、AMI、ENA、EGA、CloudWatch、CloudTrail、IAM Role ... 等內容"
date: 2020-05-30 09:10:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSAA
  - EC2
  - EBS
  - AMI
  - CloudWatch
  - CloudTrail
  - IAM
---


EBS(Elastic Block Store)
========================

## EBS 是什麼?

EBS 是 AWS 提供的 block storage 服務，可提供以下服務：

- 與 EC2 instance 搭配使用的 block storage service

- EBS 會自動在 volume 所在的 region 產生複本備份，確保資料安全 & 可用性

- 可根據效能需求 or 預算考量，選擇合適的 volume type

- 可使用 EBS snapshot 將資料備份到 S3，並搭配 S3 lifecycle 管理機制來提高資料的安全性


## EBS Volume Type

EBS volume type 有四種(官方宣稱)，直接使用下表來進行比較：
> 第五種為前一代的 `EBS Magnetic`，但應該之後會消失

![EBS storage type comparison](/blog/images/aws/EBS_Storage-Type-Comparison.png)

- 不同的應用其實都有合適的 volume type 可以對應

- 每個 volume type 對應的 API Name 需要稍微記一下

## Volumes & Snapshots

- EBS volume 與 EC2 instance 會在同一個 AZ 中

- volume 存在於 EBS 中；snapshot 則是存在於 S3 中

- snapshot 是 volume 在特定時間點的複本

- snapshot 的特性是 incremental，因此對 volume 進行 snapshot 只會針對有變動的 block 進行處理

- 第一次的 snapshot 需要花費較長的時間完成

- 不建議在 EC2 instance 運行中建立 snapshot，會影響效能；但還是可以這麼做

- EBS volume 可以直接線上擴展容量 or 更改 volume type，不會造成資料毀損 (但需要花幾分鐘的時間完成)

- EC2 instance 中的 root disk 是其實來自於 AWS 準備的 EBS volume snapshot

- 當 EC2 instance 被移除時，root disk 會跟著被移除，只有額外新增的 volume 保留下來


## 如何將 EC2 instance 或 EBS volume 移到不同的 AZ 中?

1. 對 EBS volume 進行 snapshot

2. 透過上一個步驟產生的 snapshot 建立一個 AMI(Amazon Machine Image)
> 從 EBS snapshot 建立 Image 時，**Virtualization Type** 建議選用 `Hardware-assisted viirtualization`，之後使用 image 時可以支援較多種 EC2 instance type；若選擇 `paravirtual` 會導致可以選擇的 EC2 instance type 變得很少

3. 使用 AMI 在不同的 AZ 中建立 EC2 instance

上面的範例是說明如何將 EBS volume 移到不同的 AZ 上；但如果要移到另外一個 region 呢?

1. 執行上面的前兩個步驟

2. 執行 **Copy AMI** 操作，將 AMI 複製到不同的 region 中
> 若是要實現跨 region 移動 EBS volume，就要使用 **Copy AMI** 的方式


## 磁碟加密(Encrypt)功能

- 若是 EBS volume 有加密，則對其進行的 snapshot 也都會被自動加密

- 從加密後的 snapshot 還原成 EBS volume，也會自動被加密

- snapshot 可以在同一個帳號或是跨帳號，甚至公開進行分享，但必須是沒有加密的狀態下

- 目前在建立 EC2 instance 時，支援將 root device 進行加密

### 將 EC2 Instance 未加密的 root device 進行加密的流程

若是有某台帶有未 root device 的 EC2 instance，因為某些原因需要將其加密，可透過以下流程進行：

1. 對未加密的 root device volume 進行 snapshot

2. **複製 snapshot，並選擇進行加密**

3. 使用上一個步驟已經加密後的 snapshot 建立一個 AMI

4. 透過新建立的 AMI 產生一個新的 EC2 instance (此時 root device 已經加密)



AMI Type
========

AMI 是產生 EC2 instance 的基礎，使用者在選擇 AMI 時，可基於以下幾個項目進行選擇：

- Region

- 作業系統

- 系統架構(32-bit 或是 64-bit)

- 啟動權限

- root device 所使用的 storage 類型，可能會是 `Instance Store` 或是 `EBS Backed Volume`

## AMI 類型

- 所有的 AMI 都是來自於兩個來源：**Amazon EBS** or **Instance Store**(Ephemeral Storage)

- EBS volume 的來源是透過 Amazon 預先準備好的 EBS snapshot 所產生的

- Instance Store volume 則是從預先儲存在 S3 的 template 中建立出來的 (Console 中的 **Community AMI** 頁面可看到)

## 考試重點

- 當啟動 EC2 instance 之前，可以新增額外的 instance store volume & EBS volume

- 但是當 EC2 instance 被啟動後，就無法新增額外的 instance store volume，只能新增 EBS volume

- Instance Store Volume 所建立的 instance 無法停止，因此一旦 instance 出問題，資料就會遺失了；但透過 EBS backed instance 是可以停止的

- 兩種類型的 instance 重新啟動都不會遺失資料

- 預設情況下，兩種類型的 instance 被移除後，root device 都會一併移除

- 可以設定當 instance 被移除後不移除 EBS backed volume，不過 instance store volume 無法做這樣的設定



ENI, ENA, EFA 的比較
===================

這個部份可能會需要曾經有硬體相關的背景可能會看比較懂....

總之有些時候可能會有一些特殊應用，會需要 VM 可以有接近 Bare Metal(實體機器) 的網路效能(包含延遲)，但又不想管理不甚彈性的實體機器，那 AWS 提供一些額外的加強型網路選項可以協助這個部份；若是真的無法達到需求，再考慮實體機器也可以....

## ENI (Elastic Network Interface)

- 就是一般啟動 EC2 instance 預設使用的虛擬網路卡

> 若是需要獨立網路(例如：logging network)，但不需要特別高效能，希望便宜些，就用此選項

## ENA (Elastic Network Adapter)

- 加強型的虛擬網路卡，使用 SR-IOV 功能，讓 VM 對硬體設備的存取可以繞過 Hypervisor 直接對硬體存取，大幅提昇網路效能、降低延遲 & CPU 使用率
> SR-IOV 技術不僅可用在網路設備上，儲存設備也可以....簡單來說就是支援 SR-IOV 的 PCIe 設備都行

- ENA 在網路部份提供了更高的頻寬、穩定且更低的延遲

- 啟用這個功能不會用任何無外費用產生，但僅有部份 Instance type 支援此功能

- ENA 可以支援到 100Gbps 的網路頻寬；而在以前的 instance type 有支援 10Gbps 頻寬的 VF(Intel 82599 Virtual Function) 選項，現在都以 ENA 為主

- 若是要建立一個獨立網段做特定的工作(例如：storage data plan)並兼顧低成本且高可用的需求，就可以利用這樣的功能

> 若需要 10Gbps ~ 100Gbps 且穩定的網路品質，可以選用 ENA

## EFA (Elastic Fabric Adapter)

- 一種特殊的網路設備，可用來提昇 HPC 或是機器學習相關應用的效能

- 比起傳統 cloud-based HCP 系統，可以提高更低的網路延遲 & 更高的網路輸出入

- 讓 instance 中的應用可以 bypass Linux kernel(OS-bypass)，直接與硬體通訊，藉此大幅提高速度並降低延遲

- 目前此功能僅支援 Linux 平台

> 特殊與 HPC 相關的應用(例如：機器學習)，就可以考慮此選項



CloudWatch & CloudTrail
=======================

CloudWatch & CloudTrail 都是很大的主題，這邊暫時以 CSA 的考試需求為主，因此了解一下兩個服務的功能特性 & 比較即可。

## CloudWatch

- **主要作為監控效能用**，是 AWS 重要服務之一

- 同時與其他 AWS 服務都有高度整合(例如：EC2, ASG, ELB, Route53, EBS, CloudFront ... 等等)，方便進行監控的設定

- 除了 AWS 服務之外，也可以同時用來監控自行開發的應用程式

- CloudWatch 預設每五分鐘會對 EC2 進行監控資訊的蒐集；但若是希望可以監控到更細節的資訊，可以設定成每一分鐘(在啟用 instance 時勾選 `Enable CloudWatch detailed monitoring`)

- CloudWatch 有幾個重要的組成部分：

    - `Dashboard`：以圖形展現的方式呈現監控數據，可以是 region level or global level

    - `Alarm`：設定某個 thrshold 到達時，觸發告警

    - `Event`：可指定當某個 AWS resource 變更時，執行預先指定的工作

    - `Logs`：CloudWatch 也可以協助蒐集 & 監控、儲存文字型態的 log 資訊


## CloudTrail

- 用來檢視使用者對於 AWS 各項資源的使用記錄 & 狀況，**主要用來提供稽核用**

- 記錄內容包含 console 的操作 & API call，以及來源 IP ... 等資訊



與 IAM Role 的搭配使用
====================

不論要存取任何 AWS resource，都必須要有合法且足夠的權限才有辦法進行；在一般情況下，要取得權限，可以到 IAM 中新增 user，指定需要的權限，並取得 Access Key ID & Secret Access Key 來使用。

以 EC2 instance 為例，若要設定 AWS credential，就必須在檔案 `~/.aws/credentials` 中，設定以下內容：

```ini
[default]
aws_access_key_id=[YOUR_ACCESS_KEY_ID]
aws_secret_access_key=[YOUR_SECRET_ACCESS_KEY]
```

但若是需要串接很多不同服務，就可能需要在很多不同的 EC2 instace 都對 AWS resource 進行存取，在每一台 instance 中設定 Access Key ID & Secret Access Key 並不切實際且難以管理，因此這問題就可以用 `IAM Role` 來解決。

流程如下：

1. 在 `IAM` 中新增 Role，並設定所需要的權限

2. 將上一個步驟新增的 role 與 EC2 instance 進行繫結

如此一來，即使在 EC2 instance 沒有 AWS credential 也可以直接對有權限的 AWS resource 進行存取

## 考試重點

- 若要賦予 EC2 instance 存取其他 AWS resource 的權限，使用 IAM role 會比 Access Key ID & Secret Access Key 更安全

- Role 的概念比較容易管理 & 直覺

- Role 可以在 EC2 instance 被建立後，透過 console 或是 command line 進行繫結

- Role 的有效範圍是 global，因此設定之後可以在所有 region 中使用



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Amazon Elastic Block Store (EBS) - Amazon Web Services](https://aws.amazon.com/ebs/?ebs-whats-new.sort-by=item.additionalFields.postDateTime&ebs-whats-new.sort-order=desc)

- [Deep Dive on EBS | Complete Think](https://rickhw.github.io/2017/07/16/AWS/Deep-Dive_EBS/)

- [Enhanced networking on Linux - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking.html)

- [SR-IOV 簡介 @ 立你斯學習記錄 :: 痞客邦 ::](https://b8807053.pixnet.net/blog/post/345974548)

- [詳解“硬核”虛擬化技術SR-IOV原理-知識星球](http://www.ipshop.xyz/12470.html)

- [Elastic Fabric Adapter - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)

- [Amazon CloudWatch - Application and Infrastructure Monitoring](https://aws.amazon.com/cloudwatch/)

- [AWS CloudTrail – Amazon Web Services](https://aws.amazon.com/cloudtrail/)

- [IAM Roles - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)