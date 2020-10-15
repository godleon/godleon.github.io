---
layout: post
title:  "AWS CSA Associate 學習筆記 - Serverless"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容描述在使用 AWS Serverless 相關服務"
date: 2020-08-07 05:55:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSA
  - Serverless
  - Lambda
  - ECS
---


Lambda
======

## Serverless 與傳統架構的比較

![Serverless v.s. Traditional](/blog/images/aws/Serverless_Traditional-vs-Serverless.png)

- 傳統架構中，最前端會是 ELB，接到提供運算能力的 EC2 instance，並存取內部的 RDS

- 在 Serverless 架構中，最前端的可能就會改成 API gateway，接到後面的 Lambda function，並存取內部的 DynamoDB


## Lambda 相關概念

若同時有多個 user 發送 request 到 API gateway，對到相同的 Lambda function，其實在後端會觸發多個 Lambda function 起來運行，因此不需要考慮 scaling 的問題

支援的語言有 `.NET Core`, `Go`, `Java`, `node.js`, `Python`, `Ruby` ... 等。

計費是以以下兩個基準計費：

- Request 數量

- 每個 Request 所花費的時間(但單位還是會分配多少 memory 來決定單位價格)


## 選擇 Lambda 的理由

- 不需要 server 了! 省掉很多管理成本

- 會根據需求進行 auto scaling

- 可以省很多錢

## 應考重點

- Lambda 會根據需求自動 scale out

- Lambda function 都是獨立的，每個 event 都會觸發產生一個 Lambda function 運作

- Serverless

- 了解哪些服務是 serverless
> 例如：Aurora serverless, Lambda

- Lambda function 可以觸發另一個 Lambda function (可以一個串一個)

- 使用 Lambda 的架構變得複雜時，AWS X-ray 可以協助 debug

- Lambda 的運行範圍可以是 global，不會被特定的 region 侷限

- 搞清楚 Lambda function trigger 可以有哪些



實作課程重點節錄
==============

## Serverless Webpage

### 基本概念

- Lambda function 的 trigger 有很多種(例如：CodeCommit, DynamoDB, Kinesis, S3, SNS, SQS, AWS IoT, API Gateway ... 等等)

- 要透過 Lambda function 提供動態網頁內容，則是需要搭配 `API Gateway`

- Environment variables 是其中一種將變數傳到 Lambda function 的方法

- 可為 Lambda function 設定執行時所配置的 memory 大小 & timeout

- Concurrency 預設為 1000，可以調整

### 設定 trigger


API 部份：

- 因為只提供單純的動態頁面，不需要給 ANY(所以可以刪除)，只需要 GET 即可

- 新增 GET method 需要重新設定 Integration Type(Lambda Function)、勾選 Use Lambda Proxy Integration、選擇 Lambda function ... 等等

- API 設定完後，需要執行 `Deploy API` 工作才能使用
> Deployment stage 選擇 `default` 即可



Serverless Application Model (SAM)
=================================

## 什麼是 SAM?

- Serverless Application Model(SAM) 被開發出來的主要用意是要讓開發者能很容易的開發 serverless application 

- 一種 CloudFormation extension，用來建立 serverless application (因此所有 CloudFormation 的功能，SAM 都有支援)

- 提供了新的 type，分別是 `function`, `API`, `table`(DynamoDB)

- 可在 local 使用 Docker 運行 serverless application

- 可搭配 CodeDeploy 來打包並佈署程式

## 使用 SAM 需要注意的事項

- 透過 SAM cli 可以很方便的產生 sample template，並根據需求進行調整修改 (以下是一個 SAM template 的範例)

```yaml
# 宣告這是 SAM template
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Hello Worl SAM Template

# 可用來套用到所有 function 的 global configuration
Globals:
  Function:
    Timeout: 3

# 使用本地端程式建立 Lambda function 的定義
# 會建立 API Gateway endpoint, mappings & permissions
Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.8
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /hello
            Method: get

# 輸出與上述資源相關的資訊
Outputs:
  HelloWorldFunction:
    Description: "Hello World Lambda Function ARN"
    Value: !GetAtt HelloWorldFunction.Arn
```
> 其實這就是一個 CloudFormation 的定義，只是多了 `Type: AWS::Serverless::Function` 在上面

- 透過 `sam build` & `sam deploy --guide` 就會出現一系列的互動操作，協助將程式佈署到 AWS Lambda 上

## 其他

可作為 Lambda Function event source：

![Lambda Function event source](/blog/images/aws/Serverless_Lambda-Function-Event-Sources.png)



Elastic Container Service (ECS)
===============================

## 什麼是 ECS?

![ECS introduction](/blog/images/aws/ECS_Introduction.png)

ECS 其實就是 AWS 幫你運行 container，幫你省掉很多管理的功夫，也整合了原有的 IAM，大概有以下幾個特點：

- 全託管服務，是種 container orchestration service

- 佈署 container 時，會建立相對應的 cluster

- 如果不想要管理 VM，還可以選擇 serverless Fargate

- 內含 scheduler，會將 container 自動放到最佳的位置運行

- 可定義每個 container 的 CPU & memory 資源需求規則

- 提供完整的監控功能 (當然也能與 CloudTrail & CloudWatch 進行整合)

- 提供開發者方便佈署、更新、roll back workload 的方式

- 上面的管理功能都是免費的，使用者只要為 workload 付費即可

- 可與現有的 VPC, security groups, EBS volumes, ELB .... 等服務整合


## ECS components

要正確的使用 ECS，就必須先知道 ECS 中包含了以下幾個 component & concepts：

- `Cluster`：一個邏輯的概念，包含了 ECS 的各種資源(包含 EC2 instance or Fargate instance)

- `Task Definition`：workload 的定義，類似 Dockerfile，但運行在 ECS 中；重點是 Task Definition 中可以包含多個 container 的定義 (類似 k8s pod)

- `Container Definition`：單一 container 的定義，CPU, memory, port mapping 相關的設定會放在這
> 想當然爾，這個會包含在 Task Definition 中

- `Task`：就是真實運行的 workload，透過 Task Definition 產生出來

- `Service`：用來作為 scaling 的設定，可定義 minimum & maximum 的值

- `Registry`：用來儲存 container image 的服務，例如：ECR or Docker Hub

- `ECS Agent`
  - 運行在 ECS cluster 中的每一個 EC2 instance 中
  - 協助互相傳遞 instance 的相關訊息，例如：正在運行的 task、資源使用狀況...等等
  - 負責啟動 & 停止 task


## AWS Fargate 的特色

AWS Fargate 是 AWS 的新服務，其實目的很簡單，就是要減輕使用者管理的負擔，有以下幾個特點：

- 是種 Serverless cotnainer engine，因此可用在 ECS & EKS 上

- 使用者再也不用管理 VM 了，專注在服務 & 應用上就可以了

- 僅須根據 application 的資源使用量付費

- 每個 workload 都會擁有獨立的 kernel，因此共享 kernel 所造成的安全疑慮也就沒有了

- 每個 workload 都是獨立 & 安全的

### 使用 EC2 instance 的時機

Fargate 看似美好，但若是遇到以下情況，可能就還是需要選用 EC2 instance 來管理 workload：

- 公司規定

- 需要更高度的客製化

- 需要 GPU


## 如何接入外部的流量?

要接入外部的流量，則需要跟 ELB 搭配，而 ECS + ELB 的搭配有以下幾個特性需要注意：

- ELB 的部份，有 ALB、NLB、CLB 可選擇

- ALB 用來分流 HTTP/HTTPS(layer 7) 的流量，同時支援以下特性：
  - dynamic host port mapping
  - path-based routing
  - priority rules

- NLB & CLB 用來分流 TCP(layer 4) 的流量

- ALB 是建議優先選用的 ELB type

- 同時支援 EC2 & Fargate


## 安全性的考量

在安全相關的設定上，基本上就是**以最小權限原則來進行設定**，那這件事情應該要如何完成呢? 先看下圖：

![ECS security](/blog/images/aws/ECS_Security.png)

- 圖中的左半部，安全性設定是套用在 EC2 instance上，因此所有運行在 EC2 instance 上的 task 都有存取同樣資源的權限

- 圖中的右半部，安全性設定則是套用在每個 Task 上，每個 task role 都會有不同的資源存取權限

- 綜合以上兩點，使用 ECS 時若要取得較高的安全性，則是可以將 IAM role 設定在 task level 上



References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [AWS Lambda – Serverless Compute - Amazon Web Services](https://aws.amazon.com/lambda/)

- [AWS Serverless Application Model - Amazon Web Services](https://aws.amazon.com/serverless/sam/)

- [Amazon ECS - Run containerized applications in production](https://aws.amazon.com/ecs/)

- [AWS Fargate - Run containers without having to manage servers or clusters](https://aws.amazon.com/fargate/)