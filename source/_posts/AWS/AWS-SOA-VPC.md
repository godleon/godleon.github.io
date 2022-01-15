---
layout: post
title:  "AWS SOA 學習筆記 - VPC(Virtual Private Cloud)"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 VPC(Virtual Private Cloud) 服務在實際操作維運上需要注意的事項"
date: 2022-01-15 16:00:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - VPC
---

DNS Resolution Options
======================

在 VPC 中關於 DNS 的設定有兩個，分別是：

1. `enableDnsSupport`：DNS Resolution

2. `enableDnsHostnames`：DNS Hostnames

## DNS Resolution

![VPC - DNS resolution](/blog/images/aws/VPC/AWS-VPC_DNS-Resolution.png)

`enableDnsSupport` 這個設定是用來決定是否 VPC 中的 DNS 解析是由 Route53 Resolver 來處理(此選項預設為 `true`)，而這個 AWS 預設提供的 DNS server 會在以下兩個 IP address：

1. `169.254.169.253`

2. `xx.xx.xx.2` (這是服務所在的 subnet 的第二個 IP address)
> 此 IP 來自於 AWS 在每個 subnet 預佔的 5 個 IP 的第二個，相關資料可參考 [AWS 官網文件](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html)說明

而若是這個選項沒有啟用，那就必須自己建立 DNS server，並將所有需要 DNS 查詢的部份指向自己的 custom DNS server(如上圖所示)

## DNS Hostnames

![VPC - DNS resolution](/blog/images/aws/VPC/AWS-VPC_DNS-Hostnames.png)

- 這個選項 `enableDnsHostnames` 在 default VPC 中預設是開啟(true)的，但在新建立的 VPC 中預設是關閉(false)的

- 這選項必須在 `enableDnsSupport` 也設定 true 下才會有作用

- 開啟此選項，每個有 public IP address 的 EC2 instance 都會得到一個 public hostname

此外，若是有設定 Route53 private host zone，那就必須同時啟用 `enableDnsSupport` & `enableDnsSupport` 兩個選項，裡面的 DNS record 設定在 VPC 中才會有效

![VPC - Route53 Private Host Zone](/blog/images/aws/VPC/AWS-VPC_Route53-Private-Zone.png)


VPC Reachability Analyzer
==========================

當 Security Group & NACL(Network Access Control List) 在設定上有誤需要排查網路問題時，AWS 提供了 [VPC Reachability Analyzer](https://docs.aws.amazon.com/vpc/latest/reachability/what-is-reachability-analyzer.html) 這個工具，可以用來協助分析網路不通的原因。

此工具本身不會主動送 packet，只是單純分析從 source 到 destination 中的 NACL & Security Group 設定，檢查那一段的設定出了問題以至於網路不通；若是不通會在畫面顯示問題是出現在那一段。

而這服務是要收費的，一次是 $0.1 美金，有需要的排查網路問題時應該是可以派上些用場。


VPC Flow Logs Troubleshooting
=============================

關於 VPC Flow Log 的介紹可以看一下之前寫的 [CSA 考試重點內容](AWS/AWS-CSA-associate-VPC-part2/)中有提到，以下簡單說明一下如何透過 VPC Flow 來協助排查 security group or NACL 設定上的問題

![VPC Flow Logs - Troubleshooting](/blog/images/aws/VPC/AWS-VPC_VPC-FlowLogs-Debug.png)

上圖是用來說明觀測 VPC Flow log 重點，在下方逐一說明

對於外部進來的 request：

1. 若是 inbound 被 reject，那可能 SG & NACL 都有問題

2. 若 inbound accept，但 outbound reject，通常是 NACL 的問題

對於內部出去的 request：

1. 若 outbound 被 reject，那可能 SG & NACL 都有問題

2. 若 oubound accept，但 inbound 被 reject，通常是 NACL 的問題

> 其實整個核心的檢查概念還是在 Security Group 是屬於 stateful 的，而 NACL 是屬於 stateless


Site-to-Site VPN
================

## Site-to-Site VPN

![VPC - Site to Site VPN](/blog/images/aws/VPC/AWS-VPC_Site-to-Site-VPN.png)

在建置 Site-to-Site VPN 的部份，需要注意的重點有：

- 在 VPC 端建立 `Virtual Private Gateway(VGW)`，可以設定 ASN(Autonomous System Number)

- 在 on-premise 端建立 `Customer Gateway(CGW)`，這可以是透過軟體安裝的方式 or 直接使用支援的硬體設備

- Customer Gateway 可以直接使用 public IP，但放在 NAT 後面其實也是沒問題的

- **Virtual Private Gateway 必須啟用 Route Propagation**，相關的 NACL & Security 也需要設定正確，兩端才會互通

- 完成了 `Virtual Private Gateway(VGW)` & `Customer Gateway(CGW)` 的設定，才可以新增 site-to-site VPN 的設定

## VPN CloudHub

![VPC - VPN CloudHub](/blog/images/aws/VPC/AWS-VPC_VPN-CloudHub.png)

若是 on-premise 的 site 有多個，都同時想要與 AWS VPC 進行 site-to-site VPN 串接，on-premise site 之間也有互通的需求，那就可以利用 VPN CloudHub 這個服務：

- 可提供 VPC 同時與多個 on-premise site 串接

- 可作為備援線路之用，但因為是 VPN，因此還是走 public Internet

- 每個 site 都連到同一個 Virtual Private Gateway，設定 dynamic routing & routing table 後，就可以互通了


VPC Endpoint (AWS PrivateLink)
==============================

關於 VPC Endpoint 的介紹可以看一下之前寫的 [CSA 考試重點內容](AWS/AWS-CSA-associate-VPC-part2/)中有提到，以下繼續補充在 SOA 課程中額外看到的部份。

當要讓 VPC 中某個服務同時給很多個 VPC 存取時，有幾種方式：

1. 透過 public Internet；若是內部服務，其實這樣不是很合適

2. 透過 VPC peering；這方法要建立很多 peering 相關設定

3. 透過 VPC Endpoint；這是目前解決此類問題的最佳解

![VPC - AWS PrivateLink](/blog/images/aws/VPC/AWS-VPC_PrivateLink.png)

VPC Endpoint(AWS PrivateLink) 有以下特點：

1. 更安全，自動擴展，可支援上千的 VPC 的連線(甚至來自其他帳號的 VPC 也可以)

2. 不需要做 VPC peering、Internet Gateway、NAT、Route Table .... 等設定

3. Service VPC 中需要一個 NLB(Network Load Balancer)，Consumer VPC 則只要一個 ENI(or GWLB) 即可

另外一個比較複雜的場景如下圖：

![VPC - AWS PrivateLink 2](/blog/images/aws/VPC/AWS-VPC_PrivateLink2.png)

上圖的重點如下：

- 內部服務透過 ALB 暴露出來，前面掛 NLB 來滿足 AWS PrivateLink 建置需求

- consumer VPC 直接黏一張 ENI，就可以有個 VPC endpoint 可以存取 service VPC 的服務

- on-premise site 存取的入口統一都是 Virtual Private Gateway，可以走 DX(DirectConnect) or VPN 過來


Transit Gateway
===============

## Overview

當 VPC 數量多起來，且 VPC 之間的串接越來越複雜的時候，peering 部份的設定自然也會變得越來越複雜，結果狀況就會變成如下圖所示：

![VPC - Before Transit Gateway](/blog/images/aws/VPC/AWS-VPC_Before-Transit-Gateway.png)

從上圖可看出，除了多個 VPC 之間的複雜 peering 關係之外，考慮到還有 on-premise site 要連上 VPC，因此又會包含了 VPN connection，整體的 network topology 越來越複雜且難以管理。

因此 AWS 還提供了另外一個稱為 [Transit Gateway](https://aws.amazon.com/transit-gateway)的服務，來解決此問題，使用 Transit Gateway，network topology 可以優化成下面的樣子：

![VPC - After Transit Gateway](/blog/images/aws/VPC/AWS-VPC_After-Transit-Gateway.png)

- 將多個 VPC & on-premise site 以 star topology 的方式串接起來

- 雖然 Transit Gateway 屬於 regional service，但可以串接多個 region 的 VPC

- 可利用 RAM(Resource Access Manager) 將 Transit Gateway 可以跨 AWS Account 使用

- 也可以將 on-premise site 所用的 Direct Connect Gateway or VPN connection 串接上來

- Transit Gateway 是唯一支援 IP Multicast 的服務


## 多帳號共享 Direct Connect 

![VPC - After Transit Gateway - share direct connect between multiple accounts](/blog/images/aws/VPC/AWS-VPC_After-Transit-Gateway.png)

Transit Gateway 提供給多個 AWS account 共用的功能，如上圖所示，這樣的作法需要搭配 Direct Connect 進行，因此會需要透過 `Direct Connect Gateway` + `Direct Connect Endpoint` 的組合來完成。

## Site-to-Site VPN ECMP

![VPC - After Transit Gateway Site-to-Site VPN ECMP](/blog/images/aws/VPC/AWS-VPC_Transit-Gateway-VPN-ECMP.png)

Transit Gateway 還額外提供了 Site-to-Site VPN ECMP(Equal-Cost Multi-Path Routing) 的功能，這功能的特點是：

- 允許多個網路封包可以各自走在最佳化的路徑上(不一定會是同一條)，封包來回也可以走不同線路 (如上圖框起來的地方所示)

- 透過建立多條 site-to-site VPN 連線到 Transit Gateway，可達到增加到 AWS 的連線頻寬的效果 (如上圖的兩個 site-to-site VPN connection)


Traffic Mirroring
=================

![VPC - Traffic Mirroring](/blog/images/aws/VPC/AWS-VPC_Traffic-Mirroring.png)

- traffic mirroring 的功能通常用來排查問題時使用、content inspection、threat monitoring .... 等場景下使用

- 可以將指定將某個 ENI 的流量複製起來並導向另一個 ENI or NLB(Network Load Balancer)，甚至導向不同的 VPC 也可以(不一定需要在同一個 VPC)

- 可捕捉所有流量，也可以設定過濾條件來過濾某些不需要的網路封包


IPv6
====

## Overview

以下列出使用 AWS VPC 時需要注意的 IPv6 重點：

- IPv6 沒有 private IP range，所有的 IP 都是 public accessible

- IPv4 預設無法 disable，但 IPv6 可以 (所以因為 IP 不夠而導致 EC2 instance 無法啟動應該幾乎都是 IPv4 address 不足的問題)

- 在 VPC 啟用 IPv6 時，可選擇由 AWS 提供的 IPv6 range，也可以用自己的

- 若是啟用 IPv6，一樣會需要通過 Internet Gateway 來處理

- 並非所有 Internet Provider 都支援 IPv6，因此不是所有 client 都支援以 IPv6 的形式連線，可以到 [Test your IPv6 Connectivity](https://test-ipv6.com/) 網站測試看看自己是否有 IPv6 的連線能力

## Egress-Only Internet Gateway

![VPC - Egress-Only Internet Gateway](/blog/images/aws/VPC/AWS-VPC_Egress-Only-Internet-Gateway.png)

- 這是專門給 IPv6 的 EC2 Instance 使用的，目的是要讓對外 IPv6 的存取可以通，但從外面連不到 VPC 內部的 IPv6 EC2 instance

- 若用的是 Internet Gateway，就會像上圖左邊一樣，外面也可以透過 IPv6 連到 EC2 instance

- 新增 Egress-Only Internet Gateway，還是要把 Route Table 設定好才會如預期的運作

正確的 route table 設定如下圖所示：

![VPC - Egress-Only Internet Gateway](/blog/images/aws/VPC/AWS-VPC_IPv6-Routing.png)


References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)