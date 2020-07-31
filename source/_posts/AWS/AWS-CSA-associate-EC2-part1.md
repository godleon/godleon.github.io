---
layout: post
title:  "AWS CSA Associate 學習筆記 - EC2(Elastic Compute Cloud) Part 1"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 EC2 服務，會分成三篇文章來記錄，此為第一部份，包含了簡介、設定、Security Group ... 等內容"
date: 2020-05-28 04:50:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSAA
  - EC2
  - VPC
---


EC2 簡介
========

## Overview

首先是原廠介紹：

> Amazon Elastic Compute Cloud (Amazon EC2) is a web service that provides secure, resizable compute capacity in the cloud. It is designed to make web-scale cloud computing easier for developers. Amazon EC2’s simple web service interface allows you to obtain and configure capacity with minimal friction. It provides you with complete control of your computing resources and lets you run on Amazon’s proven computing environment.


可以看出 EC2 具備以下特性：

- 提供 compute capacity，也就是俗稱的 virtual machine(VM)

- 包含了許多安全性功能

- 可以因應使用者需求，提供可大可小運算能力的 VM，並且可隨時 scale up & down

- 大大降低了產生 VM 所花費的時間，讓使用者可以很低門檻的開始使用 VM 相關服務

- VM 是運行在 AWS 長久以來被驗證很穩定的基礎建設上


## Pricing Type

- **Om Demand**：用多少算多少，不用保證用量，但每個時間單位費用是最高的

- **Reserved**：預先跟 AWS 保留使用量，需要簽約 & 預付一大筆金額，但整體來說可以得到很大的折扣(相較於 on demand 模式)，

- **Spot**：透過競價模式取得的 EC2 instance，最便宜，但由於價格是浮動的，一旦價格高過原本設定的出現，instance 就會被砍掉；適合用在使用時間彈性的應用上(例如：大數據資料處理)

- **Dedicated Host**：整台實體機器都給你用了，如果軟體必須跟實體機器綁定時 or 想要先把地端的運用完全不動的先移植到雲端時，才會用到的選項。

- **Saving Plan**：這是新一代的省錢方式，對使用者來說，運用彈性變大的(例如：可改變 instance type)，詳情可參考[官網說明](https://aws.amazon.com/tw/savingsplans/)，但同樣會需要使用者預付一筆費用


## Spot Instance 計費

- spot instance 是以小時(hour)為單位計費

- 如果因為出價太低，導致 AWS 主動砍了 spot instance，未滿一小時的部份則不收費

- 如果使用者自行砍掉 VM，未滿一小時的部份則會被收一小時整的費用


## Instance Type

每個 instance type 的代碼其實都有其意義，可以對應到特別的場景 & 應用，以下列出說明：

- `F`：FPGA

- `I`：IOPS

- `G`：Graphics

- `H`：High Disk Throughput

- `T`：Cheap general purpose (例如：**t2.micro**)

- `D`：Density

- `R`：RAM

- `M`：Main choice for general purpose apps

- `C`：Compute

- `P`：Graphics (例如：Pics)

- `X`：Extreme Memory

- `Z`：Extreme Memory & CPU

- `A`：ARM-based workloads

- `U`：Bare Metal



關於 EC2 設定的選項
=================

## 設定 Instance

### 選項中的 AZ 跟實際中的 AZ 並不同

在 subnet 的部份，可以選擇對應到不同 AZ 的 subnet，但上面的選項不一定會對應到實際的 AZ，例如在 region **us-east-1** 中，有以下 AZ：

- `us-east-1a`

- `us-east-1b`

- `us-east-1c`

假設使用者 Andy 選了 `us-east-1a`，另外一位使用者 Bill 也選了 `us-east-1a`，但實際上兩個使用者的 VM 並不一定放在同一個 AZ 中。

但對同一個使用者來說，`us-east-1a`、`us-east-1b`、`us-east-1c` 就是三個不同實體的 AZ，只是你不需要去關心到底對應到那個實體 AZ；這表示使用者其實不需要知道或關心 AZ 代碼對應到的實際哪個 AZ，只要知道若是當 cloud resource 指定放置在不同的 AZ，AWS 就會幫你在實體上放在不同的 AZ 中。

### 監控

- 預設情況下，CloudWatch 每五分鐘會收集 EC2 instance metrics

- 若希望可以取得更細節的資訊(例如：每一分鐘)，那可以勾選 **Monitoring** 選項中的 `Enable CloudWatch detailed monitoring`

### 一般選項

- **[Capacity Reservation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)**：跟 saving plan 有關係，看起來像是要在省錢 & 確保有特定容量的資源可用所設計的機制，詳細的內容可以參考[官方文件說明](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)。

- **IAM Role**：可以指定 EC2 instance 所使用的 IAM 身份為何，若是需要串接其他服務，這個部份需要有正確的設定才行

- **Shutdown Behavior**：可指定關閉 instance 時，只是單純的停止(stop)，或是直接將其終止(terminate)

- **Enable Termination Protection**：可以用來保護此 VM 不會被誤刪

- **Tenancy**：可用來決定是否可以跟別的使用者共享同一台實體機的資源，有 `Shared`、`Dedicated`、`Dedicated host` 三個選項

### 進階選項

- **User data**：這裡可以用來指定 bootstrap script


## 增加 Storage

- 預設會有一個 Volume Type 為 `Root` 的磁碟，這個就是 EC2 instance 的開機磁碟，容量通常很小，預設只有 8GB

- Root volume 只能選擇 `SSD` & `Magnetic(standard)` 兩種 volume type

- Root volume 中的 `Delete on Termination` 是固定會勾選的，表示 instance 砍了，Root volume 也會跟著不見

- 可以指定增加 EBS volume，指定容量 & Volume Type(HDD, SSD ... 等等)

- EBS volume 則可以選擇更多種 Volume Type，例如：Cold HDD、Throughput Optimized HDD ... 等等

- 若是選擇 **Throughput Optimized HDD**，畫面就會出現 Throughtput 能力(MB/s)的資訊，基本上容量越大 throughtput 就會越大

- 指定不同的 Volume Type 會顯示對應的 IOPS 資訊

- General Purpose SSD 能提供的 IOPS 是根據磁碟容量而定的，另外有一個 burst 的上限

- Provisioned IOPS SSD 則是可以提供固定 IOPS 能力的磁碟

- 原本 Root volume 是無法加密的，但現在已經可以進行加密了


## 設定 Security Group

- Security Group 就是個虛擬的防火牆

- 若要連到 EC2 instance，設定正確的 security group rule 是需要的

- security group rule 在設定上不同的 protocol, port, source, destination .... 等規則


## 設定 Key Pair

- 要用來設定連線到 VM 的 ssk keypair，可以預先上傳自己的 key，也可以直接在 console 中產生

- 透過 SSH 連線，使用者名稱為 `ec2-user`



EC2 Instance 管理功能
====================

## Status Check

- **System Status Checks**：這是用來檢查底層實體機器 & Hypervisor 的狀態

- **Instance Status Check**：這是用來檢查 VM 本身的狀態


## Actions

- 若有開啟 **Termination Protection** 的功能，要先在 **Instance Settings -> Change Termination Protection** 中關閉此功能，才可以砍掉此 VM



Security Group
==============

## EC2 & Security Group

- EC2 instance 可以同時與多個 security group 綁定

- EC2 instance 一旦綁定 security group，規則馬上就會生效

- 一個 security group 可以對應到多個 EC2 instance

> 所以 EC2 instance & security group 是多對多的關係

## Security Group v.s. VPC Network ACL(Access Control List)

### Security Group

- Security Group rule 是 stateful 的

- 承上，若是 allow HTTP in，就會有一個隱性的 allow HTTP out 存在

- 承上，即使將 outbound rule 中允許向外的流量規則刪除，EC2 還是可以回應 HTTP request

- 預設所有 inbound traffic 都會被擋住，需要根據需求一條一條規則打開

- 預設所有 outbound traffic 都是開放的

- 無法設定黑名單(blacklist)，因此無法進行類似 "**阻擋存取特定服務的 traffic**" 這樣的設定

### VPC Network ACL(Access Control List)

- VPC network ACL 是 stateless 的

- 承上，若是建立了 inbound rule，就要建立相對應的 outbound rule，否則對應的服務不會通

- 可以設定黑名單，因此 "**阻擋存取特定服務的 traffic**" 這類的規則是可以設定的


應考重點整理
==========

## EC2 概觀

- **Termination Protection 預設是關閉的**，有需要就必須自己打開

- VM 被刪除(Terminated)，Root volume 會一起被刪除

- Root volume 可以加密，可以搭配第三方工具(例如：bot locker)使用，或是透過 AWS 提供的 API

- 額外加入的 volume 也可以加密

## Security Group

- 所有 inbound traffic 預設都會被擋住

- 所有 outbound traffic 預設是開放的

- security group 規則修改後馬上會生效

- EC2 instance & security group 是多對多的關係

- security group 是 stateful，在 inbound 設定的規則會同時在 outbound 中自動開放

- security group 無法設定黑名單，若需要黑名單功能，則需要使用 Network ACL 來做 (只可以設定 allow rules，無法設定 deny rules)



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Amazon EC2](https://aws.amazon.com/tw/ec2/)

- [Amazon EC2 定價 – Amazon Web Services](https://aws.amazon.com/tw/ec2/pricing/)

- [Savings Plans – Amazon Web Services](https://aws.amazon.com/tw/savingsplans/)

- [Amazon EC2 常見問答集 – Amazon Web Services](https://aws.amazon.com/tw/ec2/faqs/) [(英文版)](https://aws.amazon.com/ec2/faqs/?nc1=h_ls)

- [Security groups for your VPC - Amazon Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)