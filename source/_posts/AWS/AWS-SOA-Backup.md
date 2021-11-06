---
layout: post
title:  "AWS SOA 學習筆記 - 備份相關服務"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為備份服務(DataSync & Backup)在實際操作維運上需要注意的事項"
date: 2021-11-06 20:45:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - AWS DataSync
  - AWS Backup
---


AWS DataSync
============

## Overview

DataSync 服務的功能主要是將地端(on-premise)大量的資料移動到 AWS 上存放，需要注意的重點有以下部份：

- 可以將資料送到 AWS S3、EFS、FSx for Windows .... 等服務

- 可以將資料透過 NFS or SMB protocol 將存放於 NAS(or local) 的資料送到雲端

- 資料備份的任務可以 by hour/daily/weekly 設定排程，也可以設定上傳時的頻寬限制

- 上述的功能需要透過在地端放一個 DataSync agent 來完成


## 範例應用場景

以下舉兩個可能的應用場景：

![AWS DataSync Example 1](/blog/images/aws/Backup/DataSync_Example.png)

此場景中，資料存放於地端的 NFS or SMB server：

- 地端會找台機器安裝 AWS DataSync Agent

- DataSync Agent 會與 AWS DataSync 服務進行連線，並將資料上傳到 AWS storage 服務中

![AWS DataSync Example 2](/blog/images/aws/Backup/DataSync_Example2.png)

這個場景是要將 EFS 中的資料傳到另外一個 region 中的 EFS 中：

- 需要在 origin region 中安裝一台 EC2 instance，並在其中安裝 AWS DataSync Agent

- DataSync Agent 會透過 NFS protocol 與 EFS 串連，並透過 AWS DataSync 服務，將資料轉送到 destination region 中的 EFS 服務中



AWS Backup
==========

![AWS Backup](/blog/images/aws/Backup/Backup.png)

- 全託管服務，支援 EFx、EFS、DynamoDB、EC2、EBS、RDS、Aurora、Storage Gateway(volume gateway) ... 等服務的備份

- 統一並全自動管理 AWS resource 的備份工作

- 支援 cross-region & cross-account 的備份

- 對於備份的內容，可以作到 point-in-time recovery

- 可自訂 backup policy (實際名稱為 **Backup Plans**)，可以進行以下細節的設定：
  - 備份頻率 (every 12 hours, daily, weekly, monthly, cron)
  - backup windows
  - 可設定一段時間後轉存至冷儲存(Never、Days、Weeks、Months、Years)
  - 可設定備份保存期間 (Always、Days、Weeks、Months、Years)

- 可制定 tag-based 的 backup policy



References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)

- [What is AWS DataSync? - AWS DataSync](https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html)

- [What is AWS Backup? - AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html)