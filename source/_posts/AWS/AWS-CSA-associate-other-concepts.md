---
layout: post
title:  "AWS CSA Associate 學習筆記 - 其他未分類重要觀念"
description: "此篇文章是學習課程的過程中，將未分類但重要的觀念記錄下來，確保在進行相關架構思考 & 規劃時不會有所遺漏；或是遇到特定狀況時有個排查的正確思路"
date: 2020-10-15 18:40:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
---


災難還原(Disaster Recovery)
==========================

## 關於災難還原必須知道的關鍵名詞

- `Recovery Time Objective(RTO)`：指的是從災難發生開始，到線上服務恢復到可運行 & 可接受的狀態所花的時間

- `Recovery Point Objective(RPO)`：指的是以災難發生的時間點為基準，要確保資料可以恢復(還原)的時間點(也就是可以容忍資料遺失的時間長度)
> 假設 RPO 是 2 hours，若是災難在 10PM 發生，那至少要確保資料可以恢復到 8PM 的時間點

## 災難恢復的程度

為了確保當 on-premise 環境發生災難時，可以很快的恢復線上服務，可以考慮在 AWS 上建置備援環境，而根據建置規模的大小，可以分為以下三個等級：

- `Pilot Light`：最小版本可運行的環境；當 on-premise 環境掛掉時，可以透過 DNS 切換的方式，將流量導向 AWS 環境中，並可根據當下的需求將資源調整至所需要的大小

- `Warm Standby`：比最小版本環境再更大些，主要是要確保跟關鍵業務相關的 workload 可以與 on-premise 一比一的一致在 AWS 環境中 standby，確保流量轉移到 AWS 時，關鍵業務絕對不會出問題

- `Multi-Site Solution`：跟 on-premise 環境完全相同，而 traffic 要如何導向就由管理者決定，一般都會在 DNS 層級設定成 latency based 或是 failover



常見問題
=======

## EC2

### 無法連線到 EC2 instance

確認 security group 設定中對應的 port 是否有開啟

### 無法將 EBS volume 與 EC2 instance 繫結

- EBS volume 必須與 EC2 instance 位於同一個 AZ 中 (你不會想要存取 disk 的流量都跨 AZ 的....費用很驚人的)

- 若沒有位於相同 AZ 中，可以對 EBS volume 做 snapshot，並使用 snapshot 在同一個 AZ 中產生新的 volume

### 無法新增 instance

可能達到數量上限了(可以到 AWS console 中確認)，可以與 AWS 聯繫來提高限制

### 無法下載安裝套件

instance 可能沒有 public ip address，或是沒有位於 public subnet 中

### 在 t2.micro 的 application 變慢

- t2.micro 是屬於 burstable instance type，只能短暫的拉高資源使用量，無法一直全速運作

- 建議調整成其他更大的 instance type

### AMI 無法在其他 region 使用

- AMI 只能在其所建立的 region 中被使用

- AMI 可以被 copy 到不同的 region

### 啟動 placement group 中的 instance 收到 "capacity error"

stop 再 start 在 placement group 中的所有 instance，應該就可以解決此問題


## VPC

### 新的 EC2 instance 無法自動取得 public IP

啟動 subnet 中的 `Auto-Assign Public IP` 功能

### 設定了 NAT Gateway 還是無法讓 private subnet 中的 instance 上網

需要在 private subnet 所使用的 route table 中加上 `0.0.0.0/0 的流量導向 NAT Gateway` 的規則

### Security Group 都設定好了，但 instance 的網路還是有問題

還有 Network ACL 也需要確認，記得 Network ACL 是 stateless 的!

### 新增多個 Internet Gateway(or Virtual Private Gateway) 到 VPC 發生錯誤

每個 VPC 只能有一個 Internet Gateway (or Virtual Private Gateway)

### Security Group 中的 rule 不夠 Application 使用

沒關係，那就對 instance 綁定多個 security group

### 無法存取 private subnet 中的資源

需要使用 VPN 或是 bastion node 作為跳板才可以

### site-to-site VPC 連上了，但還是無法存取資源

地端的環境要設定 route rule，將特定的 traffic 導向 AWS VPC 去


## ELB

### LB 只顯示在特定的 AZ 上

確認新增 ELB 時，`Cross-zone Load Balancing` 的設定有開啟

### instance 健康，但註冊到 ELB 後卻是 OutOfService

要把 ELB 上對 instance 的 health check 的規則(ping protocol/port/path)設定正確
> 假設 ping TCP port 1234，但卻在 security group 沒有開放，那就是有問題的

### traffic 沒有正確的被 ELB 轉發到 instance 上

- 可能把 `listener` & `security group` 用法搞混了

- 雖然 listener 開啟了 TCP 80，但 ELB 本身還是要綁定一個開放 TCP 80 的 security group 設定

### Web server 上的 access log 都是 ELB 的 IP，不是 source IP

啟用 ELB 中將 Access Log 轉存至 S3 的功能，透過分析 S3 中的 ELB access log 就可以取得 source IP

### 無法加 instance 加到 ELB 後方

ELB 設定時需要指定可作 load balanece 的 AZ，應該是 instance 所在的 subnet 不屬於 ELB 所指定的 AZ list，調整一下 ELB AZ 的設定即可


## Auto Scaling

### ASG 中的 instance 在短時間內反覆的 start & stop(or create/terminate)

scale up/down 的 threshold 可能設定的太靠近了，導致三不五時就觸發 scale up/down 的條件

### auto scale 沒有正確的發生

可能已經到達了原本設定的 instance 最大數量