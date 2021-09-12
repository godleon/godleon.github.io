---
layout: post
title:  "AWS SOA 學習筆記 - CloudFormation"
description: "此篇文章是學習 AWS 認證課程(Certified SysOps Administrator Associate)內容時所留下的學習筆記，主要內容為實際操作維運 CloudFormation 服務時，需要了解的相關概念 & 基礎知識，以及相關需要注意的事項"
date: 2021-09-12 18:15:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CloudFormation
---

CloudFormation Update & Delete
==============================

- 調整 CloudFormation stack YAML 後，若是從 console 中看到 `Action=Modify` & `Replcace=True`，以 EC2 instance 為例，就會被 terminated 後重建，原本的資源不會保留；若是 `Replcace=False` 那就會保留原有的 resource 僅進行調整

- 資源建立的順序，是透過 `!Ref` 來判斷的


CloudFormation Template 內容組成
===============================

## Resources

- `Resources` 的設定是必要的，是用來指定要建立 or 設定什麼 AWS resource (可參考 [AWS 官方文件](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html)找到 resource 列表)

- 以 `AWS:"aws-product-name::data-type-name` 的形式表示

- 可互相參考 & 連結，而這樣的定義通常也可以用來看出 resource 之間的相依關係

- 所有要建立的 resource 都必須清楚定義在 CloudFormation tempalte 中，不能做 code generation 這類的行為，因此無法產生動態數量的 AWS resource

- 並非所有 AWS resource 都可以透過 CloudFormation 建立，還是有少數的資源無法；但可以透過 AWS Lambda Custom Resource 來做 work around


## Parameters

- `Parameters` 是提供 input 給 CloudFormation template 的方式

- 通常在以下情況會用到 parameters：
  - 需要復用 CloudFormation template 時，可能是在跨公司 or 組織的狀況下
  - 某些設定無法預先知道，例如：SSH Key

- 透過 type 的特性，可以在開發 CloudFormation template 時避免掉很多錯誤

- CloudFormation 中有很多設定可以用來控制如何使用 parameter，例如：`Type`、`Description`、`Constraints`、`ConstraintDescription`、`Min/Max Length`、`Min/Max Value`、`Defaults`、`AllowValues(array)`、`AllowedPattern(regexp)`、`NoEcho(Boolean, 用來傳遞 secret)` ... 等等

- 要引用透過 parameter 傳進來的內容，必須使用 `Fn::Ref`，也就是 `!Ref` 關鍵字

- 除了自訂的 parameter 之外，AWS 還提供了 [`pseudo parameter`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/pseudo-parameter-reference.html) 可供使用，基本概念就是用來參考到 AWS resource 相關資訊，例如：
  - `AWS:AccountID`：1234567890
  - `AWS::NoValue`：沒有任何內容
  - `AWS::Region`：us-east-2
  - `AWS::StackId`：arn:aws:cloudformation:us-east-1:：1234567890:stack/MyStack/xxx-xxx-xxx-xxx
  - `AWS::StackName`：MyStack

## Mappings

`Mappings` 是一群預先定義好的 value，以三層結構的方式定義，常用來當 key 的內容像是 region id, environment ... 等等，但需要注意的是，在 `Mappings` 的 section 中，不能包含 paramter、pseudo parameter、intrinsic functions .... 等內容存在。

以下是一個 Mappings 的定義範例：

```yaml
Mappings: 
  RegionMap: 
    us-east-1:
      HVM64: ami-0ff8a91507f77f867
      HVMG2: ami-0a584ac55a7631c0c
    us-west-1:
      HVM64: ami-0bdb828fd58c52235
      HVMG2: ami-066ee5fd4a9ef77f1
```

在 CloudFormation template 用引用 Mappings 的內容，則是需要使用到 [`Fn::FindInMap`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html) function，參考使用範例如下：
> 語法 `!FindInMap [ MapName, TopLevelKey, SecondLevelKey ]`

```yaml
.... (略)
Resources: 
  myEC2Instance: 
    Type: "AWS::EC2::Instance"
    Properties:
      # 這裡同時 pseudo parameter
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", HVM64]
      InstanceType: m1.small
```

## Outputs

- `Outputs` section 是用來定義 cloudformation stack 的輸出內容(非必要)，若是有定義，透過 AWS console/CLI 就可以看到

- 這通常用在有其他 stack 要利用特定 stack output value 時，例如：參考到 network stack 產生出來的 VPC/subnet ID

- 若是 stack A 的 output 有被其他 stack 參考到，那在所有參考到 stack A output 的 stack 都被刪除前，stack A 都無法被刪除

Ouputs section 的定義格式如下：

```yaml
# 格式
Outputs:
  Logical ID:
    Description: Information about the value
    Value: Value to return
    Export:
      Name: Value to export

# 實際範例
Outputs:
  BackupLoadBalancerDNSName:
    Description: The DNSName of the backup load balancer
    Value: !GetAtt BackupLoadBalancer.DNSName
    Condition: CreateProdResources
  InstanceID:
    Description: The Instance ID
    Value: !Ref EC2Instance
```

### Exports & Cross Stack Reference

在 CloudFormation template 中除了定義 outputs 之外，也可以透過定義 `Export` 的方式讓其他的 stack 可以透過 `Fn::ImportValue` 的方式參考到目前 stack 的 outputs，重點如下：

- `Export` 的定義是用來做 cross-stack reference 用的，在同一個帳號中，region 內的 Export name 不能重複

- Export name 的值不能 ref 到其他的 AWS resource，但可以用 pseudo parameter，例如：

```yaml
Export:
  Name: !Join [ ":", [ !Ref "AWS::StackName", AccountVPC ] ]
```

以下是使用範例：

```yaml
# Stack A Export
Outputs:
  PublicSubnet:
    Description: The subnet ID to use for public web servers
    Value:
      Ref: PublicSubnet
    Export:
      Name:
        'Fn::Sub': '${AWS::StackName}-SubnetID'
  WebServerSecurityGroup:
    Description: The security group ID to use for public web servers
    Value:
      'Fn::GetAtt':
        - WebServerSecurityGroup
        - GroupId
    Export:
      Name:
        'Fn::Sub': '${AWS::StackName}-SecurityGroupID'

# Stack B Import
Resources:
  WebServerInstance:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: t2.micro
      ImageId: ami-a1b23456
      NetworkInterfaces:
        - GroupSet:
            - !ImportValue 
              'Fn::Sub': '${NetworkStackNameParameter}-SecurityGroupID'
          AssociatePublicIpAddress: 'true'
          DeviceIndex: '0'
          DeleteOnTermination: 'true'
          SubnetId: !ImportValue 
            'Fn::Sub': '${NetworkStackNameParameter}-SubnetID'
```

## Conditions

`Conditions` 是用來讓開發者可以決定什麼情況下要不要建立 resource，例如：

```yaml
Parameters:
  EnvType:
    Type: String
    AllowedValues:
      - prod
      - test
  BucketName:
    Default: ''
    Type: String
# condition 需要獨立定義
Conditions:
  IsProduction: !Equals 
    - !Ref EnvType
    - prod
  CreateBucket: !Not 
    - !Equals 
      - !Ref BucketName
      - ''
  CreateBucketPolicy: !And 
    - !Condition IsProduction
    - !Condition CreateBucket
Resources:
  Bucket:
    Type: 'AWS::S3::Bucket'
    # 透過 "Condition" 關鍵字決定要不要建立此 resource
    Condition: CreateBucket
  Policy:
    Type: 'AWS::S3::BucketPolicy'
    # 透過 "Condition" 關鍵字決定要不要建立此 resource
    Condition: CreateBucketPolicy
    Properties:
      Bucket: !Ref Bucket
      PolicyDocument: ...
```

## Intrisic Function

CloudFormation 中比較重要需要認識的 Intrisic Function 大概有以下幾個：(詳細可參考[官網文件](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html))

- [`Ref`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-ref.html)：若是用在 parameter 上，會回傳 parameter value；若是用在 resource 上，則會回傳 resource ID

- [`Fn::GetAtt`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getatt.html)：取得特定 resource 的屬性資料

- [`Fn::FindInMap`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-findinmap.html)：透過 `!FindInMap [ MapName, TopLevelKey, SecondLevelKey ]` 的格式從 Mapping 定義中尋找資料

- [`Fn::ImportValue`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html)：從另外一個 stack outupt 取得資料

- [`Fn::Join`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-join.html) & [`Fn::Sub`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-sub.html)：字串處理

- [Condition Functions](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-conditions.html)：例如 `Fn::And`、`Fn::Equal`、`Fn:If`、`Fn::Not`、`Fn::OR` .... 等等

- `Fn::Base64`：若是要為 EC2 instance 加入 User Data，就必須要透過此 function


Helper scripts
==============

CloudFormation 設計出來主要目的是要達成 Infrastructure as Code 的目標，但其實還是有些缺點存在的，例如：

1. 要為 EC2 User Data 的可讀性不高

2. 宣告式的內容要`描述流程+條件判斷`不容易

也因為這個，就有了 helper script 這樣的功能出現，以下介紹些常用的 helper script。

## cfn-init

若是覺得 EC2 User Data script 沒這麼友善，可以考慮 [`cfn-init`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-init.html) 這個功能，簡單說明如下：

- 透過 `Metadata` 欄位設定 init script，而這個 script 的撰寫方式類似 Ansible

- 將原本寫在 User Data 的內容全部改寫到 init script 中

- 在 User Data 中透過 `/opt/aws/bin/cfn-init` 指令，**從 CloudFormation 服務中取得 init script 的內容並執行**(上面定義的 metadata 會儲存在 CloudFormation 服務中)

- init script 執行的相關 log 存放於 `/var/log/cfn-init.log`、`/var/log/cfn-init-cmd.log`、`/var/log/cfn-init-output.log` 這幾個檔案中

> 結論：cfn-init 的用途其實就是要讓 User Data 的內容可讀性更高而已

## cfn-signal & wait condition

`cfn-signal` 要處理的就是`處理流程+條件判斷`，這在一般程式語言中是再平常不過的功能，但在 CloudFormation template 這樣的宣告式語言中要作到就需要花點功夫。

從上面 `cfn-init` 的功能繼續延伸下來，如果要判斷 init script 是否有執行成功才能決定要不要往下建立其他的 resource，只有 `cfn-init` 是作不到的，還需要搭配 `cfn-init` + `wait condition` 來完成這件事情，以下是相關 helper function 的運作流程：

![CloudFromation Help Script - cfn-signal & wait condition](/blog/images/aws/CloudFormation_signal-wait.png)

1. CloudFormation 中的 `cfn-init` 為 EC2 instance 執行初始化

2. EC2 instance 在初始化過程中，將相關的 log 資料回傳給 CloudFormation service

3. CloudFormation service 中啟動一個 wait condition，等待 EC2 instance 透過 `cfn-signal` 送訊號過來

4. EC2 instance 透過 `cfn-signal` 成功發送訊號給 CloudFormation service，滿足 wait condition，繼續往下執行

> CloudFormation 撰寫範例可以參考[官網文件](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-signal.html)


## 使用 cfn-signal 遇到問題時如何排除?

透過以上的流程說明，可以清楚知道 `cfn-signal` 是怎麼運作的，哪就會有些條件需要滿足，如果遇到了問題就可以比較有清晰的思路排查問題，因此若是使用 `cfn-signal` 遇到問題時，大概可以從以下幾個方向著手：

1. EC2 instance 需要向 CloudFormation service 回報狀態，因此連外網路必須是暢通的

2. 承上，EC2 instance 中也必須要包含相關的 helper script，Amazon Linux AMI 有自帶，若是使用其他的 AMI，就得先下載

3. 如果 signal 一直沒送到 CloudFormation service，就要檢查 init script 是否成功執行，那這時候就需要進到 EC2 instance 檢查 `/var/logs/cloud-init.log` & `/var/logs/cfn-init.log` 的內容 (在那之前需要先停掉 rollback 功能，否則 EC2 會被 CloudFormation 移除)


Auto Scaling Group
==================

透過 CloudFormation 建立 ASG(Auto Scaling Group) 需要注意兩個部份，分別是 `Creation Policy` & `Update Policy`，這兩個部份是用來驗證 stack 可否算建立成功以及實際更新策略的執行方式。

## CreationPolicy

以下面這個範例來說明：(來自於[官網文件](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-creationpolicy.html))

```yaml
AutoScalingGroup:
  Type: AWS::AutoScaling::AutoScalingGroup
  Properties:
    AvailabilityZones:
      Fn::GetAZs: ''
    LaunchConfigurationName:
      Ref: LaunchConfig
    # 指定 capacity 為 3
    DesiredCapacity: '3'
    MinSize: '1'
    MaxSize: '4'
  CreationPolicy:
    # 需要有 3 個 instance 在 15 分鐘內回傳 success signal 才能算成功
    ResourceSignal:
      Count: '3'
      Timeout: PT15M
#### .... (略)
```

上面的範例中，指定了 ASG 的 desired capacity，並且在 `CreationPolicy` 中指定要在 15 分鐘內(`PT15M`)收到 3 個 instance 回傳 signal 後才算是建立成功。


## UpdatePolicy

ASG update policy 中有分為 rolling update & replace，以下使用範例來說明。

### rolling update

使用(來自於[官網文件](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html))的範例來解釋：

```yaml
AutoScalingGroup:
  Type: AWS::AutoScaling::AutoScalingGroup
  Properties:
    AvailabilityZones:
      Fn::GetAZs: ''
    LaunchConfigurationName:
      Ref: LaunchConfig
    DesiredCapacity: '3'
    MinSize: '1'
    MaxSize: '4'
## ........... (略)
  UpdatePolicy:
    AutoScalingScheduledAction:
      # 這設定為 "true" 會忽略目前 ASG 的原有設定，使用上面新的設定
      IgnoreUnmodifiedGroupSizeProperties: 'true'
    # 這會最少保留 1 instance，並最多一次更新 2 個，以 rolling update 的方式進行更新，並要求在 1 分鐘內要收到 signal
    AutoScalingRollingUpdate:
      MinInstancesInService: '1'
      MaxBatchSize: '2'
      PauseTime: PT1M
      WaitOnResourceSignals: 'true'
## ........... (略)
```

上面的範例會達成以下效果：

1. `IgnoreUnmodifiedGroupSizeProperties: 'true'`：因此會忽略原有 ASG 的設定，直接以新的為主

2. `MinInstancesInService: '1'`：至少會保留 1 個 instance 不動

3. `MaxBatchSize: '2'`：在滿足上面的條件下，最多一次更新 2 個 EC2 instance

4. `PauseTime: PT1M` & `WaitOnResourceSignals: 'true'`：會等待 1 分鐘來自 EC2 instance 送往 CloudFormation service 的 signal 

### replace

replace 就比較直覺了，以下是範例：

```yaml
UpdatePolicy:
  AutoScalingReplacingUpdate:
    WillReplace: 'true'
CreationPolicy:
  ResourceSignal:
    Count: !Ref 'ResourceSignalsOnCreate'
    Timeout: PT10M
  AutoScalingCreationPolicy:
    MinSuccessfulInstancesPercent: !Ref 'MinSuccessfulPercentParameter'
```

沒什麼特別，就是換掉就對了；只是會在搭配 `CreationPolicy` 來驗證是否有完整更新成功。



移除 CloudFormation stack
=========================

## Deletion Policy

在 CloudFormation template 中定義的每個 resource，都可以設定 `DeletionPolicy`，用來指定當 stack 被移除時的行為，一共有以下幾個：

- `DeletionPolicy=Retain`：此 resource 會被保留，適用於所有 resource type，包含 nested stack

- `DeletionPolicy=Snapshot`：會透過 snapshot 備份，但僅支援 EBS volume、ElastiCache cluster、ElastiCache ReplicationGroup、RDS DBInstance、RDS DBCluster、Redshift Cluster ... 等等

- `DeletionPolicy=Delete`：這是預設值；但對於 `AWS::RDS::DBCluster` 來說，default policy 還是 snapshot；此外，要移除 S3 bucket 之前還是要自行先清空 bucket 內的 object


## Termination Protection

若是擔心 cloudformation stack 被移除，可以在 stack 被建立出來後，將 Termination Protection 屬性啟用，那 stack 就會變成無法移除的狀態。

若是後須需要移除 stack，將 Termination Protection 關閉後，即可正常移除。


其他需要留意的功能
===============

## Rollback

當 CloudFormation 建立(or 更新) stack 失敗後，就會進入到 rollback 的狀態，分為以下兩種：

- stack 建立失敗：
  - 預設會將 stack 所有的 resource 給 rollback，可以從 cloudformation log 中看到過程，但看不到細部的錯誤訊息
  - 若是要進行進一步的 troubleshooting(例如：檢查 `cfn-init` script 為何執行失敗)，那就要先 disable rollback 功能

- stack 更新失敗：
  - 預設會還原成原本的狀態
  - 在 cloudformation 可以看到 log 中有失敗的過程 & 相關的錯誤訊息

還有個比較麻煩的狀況，就是連 rollback 都失敗時，而這個問題通常是人操作所造成的，例如下面的情境：

1. 使用者 A 建立了 stack 

2. 使用者 B 修改了 stack 內容

3. 使用者 A 透過 CloudFormation 更新 stack，但遇到更新失敗

4. CloudFormation 嘗試還原原本的 stack 狀態，但因為第二個步驟中已經被修改掉，因此 rollback 失敗

遇到 rollback 失敗情況，有兩個方式可以處理：

1. 明確 skip 有問題的 resource

2. 修正 resource 錯誤的狀態，再一次進行 update/rollback

## Nested Stack

- 顧名思義，就是 stack 中包含了 stack；若是有特定的 stack 內容常被重複使用，就可以利用這樣的設計

- 像是 Load Balancer & Security Group 這一類的 resource 就適合獨立拉出來設定並透過 nested stack 建立

- nested stack 是種 best practice 的設計

- 要更新 nested stack，盡可能從 parent(root) stack 動手

## ChangeSets

- 通常用在更新 stack 前，執行前想弄清楚到底有哪些資源會被更新時使用

- ChangeSets 內容中不會提到變更是否成功 or 失敗 (實際執行後才會知道)

![CloudFromation Help Script - cfn-signal & wait condition](/blog/images/aws/CloudFormation_ChangeSets.png)

- ChangeSets 可以透過不斷修改 CloudFormation template 後重新產生，確認有符合需求後再執行

## Drifts

使用 CloudFormation 這一類 IaC 的方式建立 resource，最怕遇到的就是人為手動修的部份，為了讓使用者可以確認目前的 stack 有無被手動修改，AWS 提供了一個 `Drifts` 檢測功能，可以讓使用者知道 stack 有沒有被手動修改過。

只是需要注意的是，`Drifts` 功能並無法保護 stack 不被修改，只能偵測而已。

## DependsOn

前面提到像 CloudFormation template 這種 IaC 的方式，要定義流程其實不太容易，不過透過 `DependsOn` 關鍵字，可以明確定定義 resource 建立的先後。

## StackSet

- 若是要透過 CloudFormation 將資源佈署到 multiple account/region 中，就需要使用到 `StackSet` 的功能

- 執行這功能需要有 Administator 的權限

- 當 stackset 被更新時，所有 account/region 中的 stack 都會被更新

- 可以指定同時間有多少 resource 被建立 or 容忍失敗的數量


References
==========

- [Ultimate AWS Certified SysOps Administrator Associate 2021 | Udemy](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate/)

- [AWS CloudFormation Init with Examples - DevOps4Solutions](https://devops4solutions.com/aws-cloudformation-init-with-examples/)