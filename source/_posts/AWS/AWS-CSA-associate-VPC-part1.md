---
layout: post
title:  "AWS CSA Associate 學習筆記 - VPC(Virtual Private Cloud) Part 1"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容為 VPC 服務，會分成多篇文章來記錄，此為第一部份，包含了簡介、Security Group、Network ACLs, NAT Gateway, 實作重點 ... 等內容"
date: 2020-05-21 05:45:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - VPC
---


Overview
========

## What is VPC?

要了解一個產品，直接看官方的定義是最快的，因此來看一下 AWS 原廠如何介紹 VPC：

> Amazon Virtual Private Cloud (Amazon VPC) lets you provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define. You have complete control over your virtual networking environment, including selection of your own IP address range, creation of subnets, and configuration of route tables and network gateways. You can use both IPv4 and IPv6 in your VPC for secure and easy access to resources and applications.

> You can easily customize the network configuration of your Amazon VPC. For example, you can create a public-facing subnet for your web servers that have access to the internet. You can also place your backend systems, such as databases or application servers, in a private-facing subnet with no internet access. You can use multiple layers of security, including security groups and network access control lists, to help control access to Amazon EC2 instances in each subnet.

根據上面的介紹，可以歸納出幾個重點：

- 可以在 AWS region 中建立一個獨立 & 可完全控制的網路環境(包含 IP、subnet、routing rules、Gateway ... 等等)

- 在 VPC 中可以根據需求，同時定義 public subnet(通常提供 web service) & private network(通常提供 backend service)

- 可透過 security group & Network ACLs 來加強安全性

- 甚至可以將地端環境 & AWS VPC 建立一個 VPN 連線，成為一個 Hybrid cloud 的架構


## VPC 網路示意

![VPC Example 2](/blog/images/aws/VPC_Example-2.png)

以下是幾個 VPC 網路的特點：

- 在 Region 中

- 可以跨 AZ (因為 AZ 是屬於 datacenter 等級，因此也表示跨 datacenter)


## VPC 標準範例介紹

以下介紹一個標準的 VPC 架構的規劃：

![VPC Example 1](/blog/images/aws/VPC_Example-1.png)

### General

- 紅色外框的部份表示 AWS Region 的範圍，上圖的範例是在 `us-east-1`

- 黃色外框的部份則是 VPC 範圍

### Gateway & Router & Route Table

- 要存取 VPC 的方式，可透過 `Internet Gateway`(來自網際網路) 或是 `Virtual Private Gateway`(可能來自地端的 VPN 連線)

- 透過兩個 gateway 進來的 traffic 第一個位置都是 `Router`

- Router 會根據 traffic source 不同，並套用不同的 routing table rules

- 接著 traffic 會根據 routing rules 被導向不同的 Network ACL 進行流量的過濾

### Network ACL

- Network ACL 是第一道防線(可以想像成 firewall)，用來根據需求決定哪些 traffic 允許(或是拒絕)進到內部的服務 or VM

- Network ACL 是 stateless，因此 inbound/outbound rules 都必須正確的定義

- 可在 Network ACL 中設定黑名單，阻擋特定的 traffic 進到內部

### Security Group

- 用來作為 Network ACL 之後，保護 EC2 instance 的第二道防線

- Security Group 為 stateful，因此 inbound rule 中允許的規則，會自動開放對應的 outbound rule
> 即使移除 **allow outbound traffic** 的規則，也都還是會通

### public/private subnet

- 外部進來的流量可以存取 public subnet 中的 Instance

- private subnet 中的 Instance 是無法被外界存取，對外連線也是禁止的

- public subnet 中的 Instance 是唯一有能力存取 private subnet 中 Instance 的地方
> private subnet 通常是存放 DB、backend servivce ... 等後端服務的地方

- 若真的有需要存取 private subnet 中的服務，可在 public subnet 中放一台 bastion node(俗稱**跳板機**)來進行

### IP 範圍的設定

目前 VPC 支援下面三個 IP 範圍可供使用者使用：

- 10.0.0.0 ~ 10.255.255.255 (10.0.0.0/8)

- 172.16.0.0 ~ 172.32.255.255 (172.16.0.0/12)

- 192.168.0.0 ~ 192.168.255.255 (192.168.0.0/16)

> 若需要正確的計算 IP 範圍，可以使用 [CIDR.xyz](https://cidr.xyz/) 網站。

### 補充示意圖

了解上面的概念後，接著看下面這張 VPC 網路示意圖，就不會覺得很突兀了：

![VPC Example 1](/blog/images/aws/VPC_Example-3.png)


## 深入認識 VPC

### 使用者可以用 VPC 作到哪些事情?

- 可以將 EC2 instance 放到你所指定的任何 subnet 中

- 可以為每個 subnet 指定想要的 IP 範圍(但僅限定於上面那三個 private IP 的範圍)

- 可以調整 subnet 之間互相通訊用的 routing rules (route table)

- 建立 Internet Gateway 並與 VPC 相連，提供 VPC 連網的能力

- 對 AWS resource 提供更多安全相關的管理 & 控制

- 設定 Security Group(for **Instance**) & Network ACLs(for **Subnet**)

### Default VPC 的特性

- default VPC 是很簡單好用的，可以讓使用者直接佈署 instance 在裡面 

- 在 default VPC 中，每個 subnet 都有一組可以連外的 routing 設定，因此所有 subnet 都預設具備連網的能力

- 每個 subnet 都會有一個 Internet Gateway 的設定，因此都是 public

- 每個 EC2 instance 都會同時有 public/private IP

### VPC Peering

- 可以使用 private IP 將兩個 VPC 連接起來，使用上就像是在同一個 private network 中

- VPC 的連結不限定要同一個帳號下的 VPC，也可以跟其他帳號的 VPC 對接

- VPC peering 是可以跨 region 的

- peering 是種星狀的連結設定，無法進行 **Transitive Peering**
> A 與 B 相連，B 與 C 相連 => 但 A 無法與 C 通訊 (需要額外進行 A & C 的相連設定)


## 應考重點整理

- 可以將 VPC 視為在 AWS 中的一個 logical datacenter

- VPC 中包含了 `IGW(Internet Gateway)`、`Virtual Private Gateway`、`Route Table`、`Network ACLs`、`Subnet`、`Security Group` 這幾個重要元素

- 1 subnet = 1 AZ
> subnet 是無法跨 AZ 的，不同的 AZ 一定是不同的 subnet

- security group 是 stateful；而 Network ACLs 則是 stateless

- VPC 無法做 transitive peering，VPC 之間若有通訊的需求，都必須做好連結的設定



實作重點
=======

## 了解 Default VPC

- AWS 會為每個使用者在每一個 region 中建立一個預設的 VPC，並在每個 region 中的 AZ 都建立一個 subnet(不同 AZ，IP 範圍肯定不一樣)

- 同時也會有預設的 route table & Internet Gateway 的設定，主要是為了提供連網的能力

## 建立 VPC

- **Tenancy**：若是希望 workload 可以跑在專屬的硬體上，可以選擇 `Dedicated`，但這要花不少錢；一般都會選擇 `Default`(共用)

- 自行建立的 VPC 時，不會自動建立 subnet & IGW(Internet Gateway)，必須自己額外建立

- 但 AWS 會幫忙建立 default route table、security group & Network ACLs ... 等設定

剛建立好的 VPC 會變成下面的架構：

![VPC arch 1](/blog/images/aws/VPC_Creation-1.png)

## 建立 Subnet

建立 subnet 需要設定以下資訊：

- **Name**(option)：可用來作為識別用

- **VPC**：指定 subnet 要新增在哪個 VPC 中

- **VPC CIDRs**：此區域會顯示當初建立 VPC 時所指定的 IP range

- **Availability Zone**：指定 subnet 所在的 AZ；但需要注意的是，**不同使用者看到的相同名稱 AZ，不一定是同一個 AZ**
> 這是 AWS 為了確保資源設定可以平均分散在不同 AZ 的設計

**IPv4(IPv6) CIDR block**：subnet 所要使用的 IP 範圍 (IPv6 可以不指定)

其他注意事項：

- `Auto-assign public IP` 是預設關閉的

- 一個 class C 的 subnet 中只有 251 個 IP 可用 (AWS 保留了幾個 IP，詳細規劃可參考[官網文件](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html))

當 subnet 建立完成後，VPC 架構圖就會變成如下：

![VPC arch 2](/blog/images/aws/VPC_Creation-2.png)

## 為 Subnet 提供連網能力

連網能力的設定需要從兩個方面下手，分別是：

- Internet Gateway

- Route Table

### Internet Gateway

1. 建立 Internet Gateway (只需要指定 name)

2. 指定 VPC 並進行 attach
  - 一個 VPC 只可以 attach 到一個 Internet Gateway，但 AWS 會確保 Internet Gateway 的 HA
  - 當有 resource 在 VPC 上運作時，是無法將 Internet Gateway 從 VPC 上 deattach


### Route Table

> 用來決定網路流量被導向何處

1. 新增 route table，指定所綁定的 VPC

2. 新增 `Routes` 設定，設定 `0.0.0.0/0`(IPv6) & `::/0`(IPv6)，並將 Target 設定為 Internet Gateway

3. 設定 `Subnet Associations`，將新的 route table 與 subnet 綁定，如此一來 subnet 就會具備連網能力了

### 注意事項

- 在 Subnet Association 中，subnet 不會主動與特定的 route table 關聯，但**會自動與 main route table 關聯**

- 若是 main route table 中包含了連外的規則設定，那表示所有的 subnet 預設都會變成 public，會有安全性的疑慮

- 比較正確的作法是確保 main route table 僅處理內部流量

當 Internet Gateway & Route Table 設定好之後，VPC 架構就會變成以下的樣子：

![VPC arch 3](/blog/images/aws/VPC_Creation-3.png)


## Public/Private 之間如何互連?

- 從上面的架構圖可以看出，唯一可以管制 subnet 之間的流量方式，就剩下 `Security Group`

- 透過在 private subnet 中設定 security group，允許特定協定 & port 的流量可以進來

- private subnet 連網能力可以透過 NAT gateway 來提供


## 應考重點

- 當建立 VPC 後，同時會建立預設的 Route Table, Network ACLs, Seciruty Group

- 但建立 VPC 後，AWS 不會在 VPC 中建立任何 subnet，也不會建立 Internet Gateway

- 在 console 中看到的 AZ(例如：`us-east-1a`)，跟別人看到的相同 AZ 名稱，實際上不一定是相同的

- 每個 subnet 會有 5 個 IP 保留給 AWS 管理使用

- 一個 VPC 只能有一個 Internet Gateway

- Security Group 無法跨 VPC 套用



NAT(Network Address Translation)
================================

NAT 在這裡的目的很簡單，就是為了讓 private subnet 具備連網的能力，但又不想將其暴露出來，希望維持在 private 環境中。

因此在環境中可以加入 `NAT gateway`，於是架構變成如下所示：

![VPC NAT gateway](/blog/images/aws/VPC_NAT-gateway.png)

但其實你也可以自己建立一個 NAT 服務來提供連網能力，這樣的作法叫做 `NAT instance`，以下是兩個不同方式的簡單介紹。

## NAT instance

這是一個完全自己動手建立 NAT 的概念，在使用上有一些事項需要注意：

- 必須建立一個 EC2 instance，且關閉 `Source & Destination Check`，才可以轉發來自其他 instance 的封包

- 必須放在 public subnet 中，本身才會有連網能力

- private subnet 中要設定 route 規則，將連外的流量導向 NAT instance

- NAT instance 可以承載的流量取決於 instance 的大小，需要加大流量的話也需要把 instance scale out

- 若要達成 HA，需要搭配 ASG(Auto Scaling Group)，並設定多台 instance & 橫跨不同的 AZ，並且還需要額外的 script 處理 failover 的需求
> 總之很麻煩....你肯定不會想這麼做....

- 必須額外設定 security group 規則

## NAT Gateway

- 在 AZ 中是有 redundant 的配置

- 對外頻寬可以從 5Gbps ~ 45Gbps

- 不需要更新，也不用跟任何的 security group 綁定

- 會自動被派發 public IP

- 若要使用 NAT gateway，還是要設定 route table 將要對外的流量導向 NAT gateway

- 不需要做 Source/Destination Check

- 建議若是 resource 橫跨 AZ，就不要使用單一 AZ 的 NAT gateway，這樣就會造成 single point of faliure；比較好的作法是讓每個 AZ 都有一個 NAT gateway，而該 AZ 中的 resource 就使用該 AZ 中的 NAT gateway



Network ACLs
============

![VPC arch 3](/blog/images/aws/VPC_Creation-3.png)

根據上圖，Network ACLs 在 VPC 中，是在 security group 之前可以進行流量過濾的手段，以下將一些使用上的特性進行整理：

- VPC 會有一個 default network ACL，允許所有 inbound & outbound 的流量

- 自訂的 network ACL，預設是拒絕所有 inbound & outbound 流量

- 每個 subnet 都必須與一個 network ACL 進行關聯，若是沒有設定，就會自動與 default(main) network ACL 關聯

- 要進行黑名單的管制(例如：阻擋特定來源 ip 流量)，只能用 network ACL，無法使用 security group(只能開白名單)

- 可以將同一個 network ACL 與多個 subnet 關聯；但一個 subnet 只能與一個 network ACL 關聯

- 每個 network ACL 中的規則，每一條都會有 number，而這個 number 用來決定執行的順序(數字越小越優先執行)

- Network ACL 中的 inbound rule & outbound rule 是分開的，雙向都要明確設定正確才會達到想要的效果

- Network ACL 是 stateless => 允許 inbound 的流量，並不會自動產生對應的 outbound rule

> 重點!! **Network ACLs 是用來管理進出 subnet 的流量；而 Security Group 則是用來管理進出 instance 的流量**



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Amazon Virtual Private Cloud (VPC)](https://aws.amazon.com/tw/vpc/)

- [NAT - Amazon Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat.html)

- [Network ACLs - Amazon Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)