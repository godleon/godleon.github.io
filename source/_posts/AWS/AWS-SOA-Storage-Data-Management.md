---
layout: post
title:  "AWS SOA 學習筆記 - Storage Service(EBS & EFS)"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為 EBS & EFS 服務在實際操作維運上需要注意的事項"
date: 2021-10-24 12:20:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - SOA
  - EBS
  - EFS
---


Encryption & DownTime
=====================

- 若是希望資料可以加密，大多數的服務都是需要在啟用時就指定好，例如：EFS, RDS, EBS volume

- 在上面的服務中，若是要再之後才要進行加密，就必須透過新增服務加上 migrate data 的方式來達成

- S3 在這個部份則是彈性許多，不僅可以 bucket/object level 啟用加密，也不會影響運行中的服務


KMS & CloudHSM
==============

- KMS & CloudHSM 都是用來加密資料用

- KMS 用在可以接受 shared tenancy 的環境；CloudHSM 則是用在嚴格的獨立環境中的

- KMS 只支援對稱加密(symmetric encryption)，CloudHSM 則是同時支援對稱加密 & 非對稱加密(asymmetric encryption)


AMI
===

- AMI 是用來產生 EC2 instance 的 template

- 可以從 EC2 instance 建立自己的 AMI

- 若是要透過 AMI 產生 instance，那就要先註冊

- AMI 是 region level 的物件；若是要在其他 region 使用同樣的 AMI，那就要複製到 target region 並註冊


EBS Volume Type
===============

- [官網文件比較](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volume-types.html)

- 僅有 `gp2/gp3` & `io1/io2` 可以作為 boot volume，`st1` & `sc1` 兩種 HDD 類型的 EBS volume 則無法

## gp2 v.s. gp3

gp2 是舊型的 general purpose SSD type，有以下特性：
- IOPS 可以 burst 到 3000，但這需要 credit
- IOPS 會與 volume size 有關連，最大可以到 16000
- 每增加 1GB 會多 3 IOPS，因此 5334 GB 的 gp2 volume 可以取得最大的 IOPS

gp3 則是新型的 general purpose SSD type，特性如下：
- 基礎 IOPS 為 3000，並且有 125 MiB/s 的 throughput
- 可以獨立增加 IOPS 到 16,000 & throughput 到 1,000 MiB/s

## Provisioned IOPS SSD

- 若是需要穩定的高 IOPS 的應用或是 IOPS 要求超過 16000，那可以考慮用這個類型的 volume type

- 適合用在 DB 或是對儲存效能敏感的 workload

- `io1`/`io2`(容量範圍：4GB ~ 16TB)
  - 最大 PIOPS(provisioned) 為 64000，但這僅限於 [Nitro EC2 instance](https://aws.amazon.com/ec2/nitro/)；其他 type 的 EC2 instance 僅能發揮到 32000 IOPS 
  - 可以單獨調整 IOPS 上限，跟 volume size 無關
  - 相較於 io1，io2 有較高的可用性 & IOPS (以相同價格為基礎作為計算)

- [io2 Block Express](https://aws.amazon.com/about-aws/whats-new/2021/07/aws-announces-general-availability-amazon-ebs-block-express-volumes/)(容量範圍：4GB ~ 64TB)：
  - 低於 ms 級別的延遲
  - 最大的 IOPS 可到 256,000，跟 volume size 有關，每增加 1GB 會多 1000 IOPS

- **io 系列的 volume type 都支援 EBS multi-attach (在同一個 AZ 的前提下)**
  - 但這情境平常不容易遇到，可能僅有在 application level 需要管理到 concurrent write 操作時才會使用
  - 必須使用具備 cluster-aware 能力的檔案系統，像是 `XFS` or `EXT4` 就並非是這一類的檔案系統
> `io1` 的 multi-attach 僅在特定幾個 region 有支援 ([官網文件來源](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volumes-multi.html))

## Hard Disk

- 無法作為 boot volume

- 容量從 125MiB ~ 16TB

- `st1`(throughput optimized) 適合用於 big data、data warehouse、log processing ... 這一類的 workload，max IOPS 為 500，max throughput 則為 500 MiB/s

- `sc1`(cold hdd)：不常存取的資料可以用這 volume type，max IOPS 250，max throught 250 MiB/s


EBS 重要功能(操作)說明
===================

## Volume Resizing

- EBS volume 的容量僅能增加，不能減少；但若要增加 IOPS，那就只有 io 系列的 volume type 可以作到

- EBS volume 容量調整後，還需要在作業系統中調整硬碟設定才可以吃到增加的容量，相關操作可參考[官方文件](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/recognize-expanded-volume-linux.html)


## Snapshot

- 執行 snapshot 操作不需要先 detach volume，但建議這麼做，避免可能的資料不一致問題

- snapshot 可以複製到不同的 AZ or region 

- 若要有效管理 EBS volume snapshot，可考慮使用 [Amazon Data Lifecycle Manager](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snapshot-lifecycle.html) 來協助管理

- Amazon Data Lifecycle Manager 可協助作到以下幾件事情：
  - 對 EBS snapshot & EBS-backed AMI 進行自動化的 creation、retention、deletion ... 等操作
  - 執行定期備份(scheduled backup)、跨帳號 snapshot 複製(cross-account snapshot copy)、移除過期備份(delete outdated backup) .... 等功能
  - 功能運行的基礎是基於 `tag` 的設定，因此必須為 EC2 instance or EBS volume 加上正確的 tag
  - 無法管理並非由 DLM 所建立出來的 snapshot or AMI

另外，還有個值得一提的功能是 `FSR(Fast Snapshot Restore)`；由於 snapshot 是儲存在 S3，當真正要使用時才會從 S3 開始拉取，因此對於第一次存取的 volume block 就會很慢(I/O latency)，而 FSR 基本上就是幫你把完整的 volume block 都先拉取過一輪，這樣之後使用透過 snapshot 產生的 EBS volume 的效能就會提昇很多；不過這功能很昂貴，要慎用，而且通常會是容量很大的 snapshot/volume 使用這功能才會有比較好的效益。
> DLM 中可以直接開啟這功能
![EBS - Fast Snapshot Restore](/blog/images/aws/Storage/EBS_Fast-Snapshot-Restore.png)


## Encryption

一個加密後的 EBS volume 可以帶來以下的效果：
- 資料在 volume 中是加密的
- 資料從 instance 傳遞到 volume 的過程中是加密的
- 所有的 snapshot 都是加密的
- 加解密的過程都是透明的，因此對使用者來說，沒有什麼額外的使用負擔，只要選擇加密即可

若是要將非加密的 EBS volume 改成加密的 EBS volume，可以透過以下幾個步驟完成：
1. 建立一個 snapshot (from 未加密的 EBS volume)
2. 透過 `copy` 的方式，加密 snapshot
3. 透過加密後的 snapshot 產生 EBS volume
> 其實可以跳過第二個步驟，未加密的 snapshot 也可以直接產生一個加密後的 EBS volume

其他需要注意的地方：
- 加密會稍微增加資料存取時的 latency


EFS
===

## 服務特色

- managed NFS，用在 Linux-based EC2 instance 上，可讓超過 100 個 EC2 instance 同時存取

- multi-AZ 的設計，高可用，但價格是 EBS volume gp2 的三倍價格

- 使用 NFS v4.1 協定

- 存取權限除了 NFS 服務本身提供的之外，也可以透過 security group 來管理

- 搭配 KMS 可作到 encrption at rest

- EFS 提供了 `Storage Tiers` 的機制，可以指定檔案多少天內沒有被存取，就從 `Standard` 自動轉到 `EFS-IA(Infrequnt Access)` 以取得更便宜的使用價格

## 效能相關選項

- Performance Mode：有 `General Purpose`(適用於 web server、CMS ... 等服務) & `Max I/O`(適用於 big data、media processing ...等服務) 可選

- Throughput Mode：有 `Bursting`(1 TB = 50 MiB/s，最大 100 MiB/s) & `Provisioned` 可選

## Access Points

![EFS - Access Points](/blog/images/aws/Storage/EFS_Access-Points.png)

若要作到更細膩的 EFS 存取控制，那就需要了解 [Access Points](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html) 這個功能，可提供的功能如下：

- 可以限定存取 EFS 時所使用的 POSIX user & group

- 可限定存取的是某一個目錄作為 client 的 root directory

- 可以使用 IAM policy 用來限制存取 EFS 服務的 client 端

## 其他注意事項

在 EFS 中，有些設定可以直接進行變更，但有些不行，舉例來說，以下設定可以直接進行變更：

- 調整 lifecycle policy

- 調整 throughput mode & 調整 provisioned throughput 的上限

- access points

但有些設定則無法直接變更，例如：

- 將資料移到設定加密的 EFS 設定中

- 調整 Performance Mode (例如：general purpose -> Max I/O)

上面這兩個操作，就需要新建立一個 EFS 設定，並搭配 DataSync 服務來完成。


References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)

- [Amazon Elastic Block Store (Amazon EBS) - Amazon Elastic Compute Cloud](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Storage.html)

- [What is Amazon Elastic File System? - Amazon Elastic File System](https://docs.aws.amazon.com/efs/latest/ug)