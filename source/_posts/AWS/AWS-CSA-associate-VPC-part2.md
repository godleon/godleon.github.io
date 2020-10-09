---
layout: post
title:  "AWS CSA Associate 學習筆記 - VPC(Virtual Private Cloud) Part 2"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 VPC 服務，會分成多篇文章來記錄，此為第二部份，包含了 Flow Logs, Bastion Host, Direct Connect, Global Accelerator、VPC Endpoint ... 等內容"
date: 2020-05-28 04:10:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - VPC
---


VPC Flow Logs
=============

## What is VPC Flow Logs?

首先看看 AWS 官網對於 VPC Flow Logs 的說明：

> VPC Flow Logs is a feature that enables you to capture information about the IP traffic going to and from network interfaces in your VPC. Flow log data can be published to Amazon CloudWatch Logs or Amazon S3. After you've created a flow log, you can retrieve and view its data in the chosen destination.

可以歸納出幾個重點：

- 可協助擷取指定 VPC 網路界面的 inbound & outbound 流量的相關資訊

- 資料可以存放在 CloudWatch 或是 S3

## 功能用途

- 可用來診斷 security group 規則是否設定正確

- 監控網路流量是否有到達預期的 instance

## 實作重點

- Flow Logs 可建立在三個不同層級上，分別是 `VPC` / `Subnet` / `Network Interface`

- 若是選擇要將 log 存入 CloudWatch，必須先在 CloudWatch 設定好對應的 log group，並且在 IAM 中設定好對應的 role

- 可以選擇記錄 Accept/Reject/All 三種不同的條件

## 考試重點

- 若是 VPC 與其他帳號的 VPC 設定 peering，那就無法在上面啟用 flow log
> 若是兩個 VPC 都屬於同一個帳號就可以

- 可以針對 flow log 設定 tag

- 一旦 flow log 建立了，就無法改變其設定 (例如：修改與 flow log 關聯的 IAM role)

- 並非所有流量都可以監控的，包含以下幾項：
  
  - 查詢 Amazon DNS 服務所產生的流量 (除非自己架設 DNS service)

  - 啟用 Amazon Windows license 所產生的網路流量

  - 與 IP `169.254.169.254` 往來的流量

  - DHCP 相關流量

  - 到 default VPC router 所在的 IP 的流量



Bastion Host
============

## 服務說明

當服務規劃階段，為了安全性考量，把服務 or EC2 instance 都往 private subnet 放以後，就會遇到一個問題：

> 要如何存取放在 private subnet 的資源或是進行佈署?

這時候就需要 Bastion Host 的協助，以下是一個架構示意圖：

![Bastion Host](/blog/images/aws/VPC_Bastion-Host.png)

從上面的架構示意圖可以看出幾個重點：

- Bastion Host 必須放在 public subnet 上，因為它是可以從外面連入的跳板

- 要進入 Bastion Host，要搞定 `Internet Gateway`、`Route Table`、`Network ACLs`、`Security Group`，必須層層關卡都打通，才可以從外面透過 SSH or RDP 連到 Bastion Host

- Bastion Host 會將管理流量導入在 private subnet 的服務 or instance

- NAT Gateway(or NAT instance) 是用來提供在 private subnet 中的 EC2 instance 之用，跟 Bastion Host 沒有關係；同時也無法作為 Bastion Host 的用途

## 優點

- 只要盡可能的強化 Bastion Host 的安全性管理即可

- 在 private subnet 中的服務 or instance 就可以暫時先不用花心力處理安全性的部份


VPN (Virtual Private Network)
=============================

![Bastion Host](/blog/images/aws/VPC_VPN-arch.png)

- VPN 是用來連接地端 & AWS VPC 用，目的是讓兩地的連線可以透過 private IP 進行

- 兩端之間互傳的的流量都是加密過的

- 每個 VPN 連線會同時使用兩個 IPSec tunnel 作為 redundancy

- 兩端都要有對應的 routing rule 才能將 traffic 導向正確的位置
> 在 VPC 端有正確的 Route Table 設定；地端則是透過 DHCP 應該可以處理這個部份

- 地端需要設定好 `Customer Gateway`(可能是實體裝置 or 軟體，必須要有 public IP)，AWS VPC 端則是需要設定 `Virtual Private Gateway`，而 VPN 連線就是將兩個 Gateway 連起來

- VPC 可以同時與 Virtual Private Gateway & Internet Gateway 綁定，但都只能各有一個



Direct Connect
==============

## 服務說明

Direct Connect 服務是讓使用者可以從地端資料中心建立一條私有的專線到 AWS 環境中存取資源用，此服務的架構圖 & 說明如下：

![Direct Connect](/blog/images/aws/VPC_Direct-Connect.png)

- AWS 在各地提供許多 **Direct Connect Location**，讓使用者可以就近選擇合適的點接入(透過 `cross-network connection`)

- 在 Direct Connect Location 中有許多 AWS Cage，裡面有 Direct Connect Router；使用者也會放置自己的 Router，然後透過 cross line 進行實體對接

- 使用者還是需要將地端設備網路接到 Router 中，但這一段通常都會落在同一個資料中心

- Direct Connect Router 則會有 AWS 自己佈建的專線網路回到 AWS 中

- 藉由此專線，可以使用專線，同時存取 AWS 公用資源(例如：S3)，也可以直接存取位於 VPC private subnet 中的服務

- 一個 Direct Connect Location 只能是一個 region，無法存取 cross region resource

## 總結

- 連接地端資料中心 & AWS

- 可以節省網路費用、降低延遲

- 若是地端到 AWS 的流量很大，很適合使用 Direct Connect 來處理

- Direct Connect 可提供一條安全的私有專線


## 建立 Direct Connect 的步驟

> 建立 Direct Connect 的步驟常會考試中出現，務必要記住

1. 在 Direct Connect console 中建立一個 virtual interface (這是 public virtual interface)

2. 到 `VPC -> VPN`，建立 Customer Gateway

3. 同樣在 `VPC -> VPN`，建立 Virtual Private Gateway

4. 將上一個步驟建立的 Virtual Private Gateway 與想要串連的 VPC 進行連接

5. 選擇並建立新的 VPN 連線，並選擇上面步驟中產生的 Customer Gateway & Virtual Private Gateway

6. 當 VPN 連線建立完成，就可以設定在地端機房中的 Gateway 或是防火牆



Global Accelerator
==================

首先看看 AWS 對於 Global Accelerator 這項服務的定義：

> AWS Global Accelerator is a service that improves the availability and performance of your applications with local or global users. It provides static IP addresses that act as a fixed entry point to your application endpoints in a single or multiple AWS Regions, such as your Application Load Balancers, Network Load Balancers or Amazon EC2 instances.

> AWS Global Accelerator uses the AWS global network to optimize the path from your users to your applications, improving the performance of your traffic by as much as 60%. You can test the performance benefits from your location with a speed comparison tool. AWS Global Accelerator continually monitors the health of your application endpoints and redirects traffic to healthy endpoints in less than 30 seconds.

![Global Accelerator](/blog/images/aws/VPC_Global-Accelerator.png)

從服務說明中可以看出幾個特性：

- Global Accelerator 的用途是要讓全球不同地區的使用者，提昇服務的體驗 & 可用性

- 為了作到這件事情，可以直接利用的就是 AWS 佈建在全世界各地的 Edge Location

- 根據 Edge server 的狀態、權重的設定 ... 等因素，AWS 會自動給出一個對使用者最佳的 edge location


## Global Accelerator 改變了什麼?

沒有設定 Global Accelerator 之前，使用者要連線到放置在 AWS 的服務，完全使用各個 ISP 所提供的 routing，整體的模樣會是下圖的樣子：

![without Global Accelerator](/blog/images/aws/VPC_without-Global-Accelerator.png)

但使用了 Global Accelerator 服務，連線的路徑則是透過 Edge location，經由 AWS 骨幹網路連線到 AWS 服務，連線狀況就會變成如下圖：

![with Global Accelerator](/blog/images/aws/VPC_with-Global-Accelerator.png)


## 設定 Global Accelerator 需要了解的概念

Global Accelerator 是由很多部份設定所組成，因此了解 Global Accelerator 中的各元件是一件很重要的事情。

### 固定 IP

- Global Accelerator 會提供兩組固定 IP，當一個 IP 因為某原因無法存取時，就可以改用另外一個 IP。

- 也可以使用自己的 IP

### Accelerator

- Accelerator 可以利用 AWS 全球骨幹網路，將流量導入最佳的節點上，藉此提昇效率 & 可用性。

- 每個 Accelerator 包含一個 or 多個 listener

### DNS Name

- Global Accelerator 會分配給每個 Accelerator 一個 DNS name(例如：`abcdexxxxx.awsglobalaccerlerator.com`)，並指向預先分配好的固定 IP

- IP or DNS name 都可以使用，也可以使用自己的 domain name 搭配 CNAME 設定的方式指向 Global Accelerator 派發的 DNS name

### Network Zone

- Network Zone 是實際提供 Accelerator 固定 IP 的地方，類似 AZ，會有一個獨立的 infra

- 當因為特殊原因導致 IP 無法存取，就可以使用另外一個 IP

- 由此可知，Global Accelerator 所配發的兩個固定 IP，將會有兩個不同的 Network zone 來提供服務

### Listener

- 在 Global Accelerator 中必須預先定義要加速的流量(例如：tcp 80 & 443)，Listner 就是用來設定這一類的規則

- 每個 Listener 可以與多個 Endpoint Group 關聯，並會將流量根據規則導向 endpoint group

- AWS 會自動根據導向的目標來決定最佳路徑

### Endpoint Group

- 每個 Endpoint Group 只會在一個 region 內，包含了同一個 region 內的多個 endpoint

- 可以透過 `traffic dial` 的功能，調整流量導向不同的 endpoint group 的百分比

- 透過 `traffic dial` 的協助，就可以容易的進行藍綠佈署，甚至要跨 region 也都沒問題 

### Endpoint

- Endpoint 可以是 NLB(Network Load Balancer)、ALB(Application Load Balancer)、EC2 instance 或是 Elastic IP address

- 簡單來說，大多數的應用下，Endpoint 會是直接提供服務給使用者的端點

- 可以針對每個 endpoint 設定權重，進而將網路流量按照比例導向不同的 endpoint(例如：讓流量全部進到某個 region，讓另外一個 region 保持淨空以方便進行性能測試)



VPC Endpoint
============

## 簡介

為了資訊安全，在網路架構的規劃中，我們可能會進可能的將服務放在 private subnet 中，並禁止其對外的流量。

但如果服務有對外與其他 managed service(例如：S3) 連接的需求呢?

原本的作法可以透過 NAT gateway 讓服務取得連網的能力；但另外一種較好的方式，則是透過 `VPC enpoint` 將 managed service 黏到 VPC private subnet 中，變成 private subnet 中的一個 endpoint。

**如此一來，即使沒有 NAT gateway，也可以讓服務直接存取 AWS managed service。**

舉例來說，原來要存取 S3 必須要有連網能力，因此需要透過 NAT Gateway，所以會是以下的架構：

![without VPC Endpoint](/blog/images/aws/VPC_Endpoint-1.png)

若是透過 VPC endpoint，則可以在 VPC private subnet 中建立出一個虛擬裝置，透過該裝置就可以直接與 S3 通訊：

![with VPC Endpoint](/blog/images/aws/VPC_Endpoint-2.png)

但並非所有服務都支援 VPC endpoint，詳細列表可以參考 [AWS 官網資訊](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints.html)。

## 特性

- VPC endpoint 是透過 PrivateLink 的方式提供的

- 使用 VPC endpoint，就不需要 Internet Gateway、NAT device、VPN connection ... 等方式來取得對外的連網能力才能存取 AWS managed services

- VPC enpoint 是種黏在 VPC subnet 上的虛擬裝置，可以輕易的作到水平擴展並達到 redundant & HA 的目的

- VPC endpoint 類型有兩種，分別是 `Interface Endpoint` & `Gateway Endpoint`

- 目前支援 Gateway Endpoint 的服務有 `S3` & `DynamoDB` 



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [VPC 流程日誌 - Amazon Virtual Private Cloud](https://docs.aws.amazon.com/zh_tw/vpc/latest/userguide/flow-logs.html)

- [AWS 上的 Linux 堡壘主機 - 快速入門](https://aws.amazon.com/tw/quickstart/architecture/linux-bastion/) ([英文](https://aws.amazon.com/quickstart/architecture/linux-bastion/))

- [AWS Direct Connect](https://aws.amazon.com/tw/directconnect/) ([英文](https://aws.amazon.com/directconnect/))

- [AWS Global Accelerator – Amazon Web Services](https://aws.amazon.com/tw/global-accelerator/) ([英文](https://aws.amazon.com/global-accelerator/))

- [VPC 端點 - Amazon Virtual Private Cloud](https://docs.aws.amazon.com/zh_tw/vpc/latest/userguide/vpc-endpoints.html)