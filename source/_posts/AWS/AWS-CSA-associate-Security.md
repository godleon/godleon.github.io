---
layout: post
title:  "AWS CSA Associate 學習筆記 - Security"
description: "此篇文章是學習 AWS 認證課程(Certified Solution Architect Associate)內容時所留下的學習筆記，主要內容描述在使用 AWS 服務時，網路安全性 & 資料存取安全性要如何考量 & 設計，藉以確保服務可以安全地運行"
date: 2020-07-31 06:35:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - CSAA
  - Security
  - KMS
  - CloudHSM
  - SSM Parameter Store
---


如何有效減少安全威脅?
==================

要有效的減少安全威脅，其實總結一句話，就是 **拒絕惡意行為的存取**。

但在在不同的網路 & 系統架構下，作法就會有些不同，以下舉幾個常見範例：

## 只有 EC2 Instance

在這樣的情況下，我們可以從 VPC & EC2 Instance 兩個地方來著手

![Network ACL + host-based firewall](/blog/images/aws/Security_NACL-with-host-firewall.png)

- VPC 中提供的 Network ACL & Secirty Group 可以限制存取 EC2 instance 的來源

- EC2 Instance 本身的防火牆功能也可以限制 (例如：firewalld, iptables, ufw, windows firewall)

- 這樣的設定通常用在 Layer 4 的阻擋，無法有效的處理 Layer 7 的狀況


## 使用 ALB

使用了 ALB 之後，有一點必須注意的是，通過 ALB 進入到 EC2 Instance 的流量，EC2 Instance 看到的來源 IP 永遠是 ALB 本身(看不到真實的使用者來源)，因此在安全設定上就需要調整一下：

![Network ACL + ALB + Security Group](/blog/images/aws/Security_ALB-with-SecurityGroup.png)

- EC2 Instance 可透過 Security Group 的設定，僅允許來自 ALB 的流量

- Network ACL 的設定依然要進行，但 target 則是位於 VPC public subnet 中的 ALB，並非 EC2 Instance

- 這樣的模式還是只能用於 Layer 4 的阻擋

## 使用 NLB

若改用 NLB，EC2 Instance 就可以看到真實的使用者來源：

![Network ACL + NLB + Security Group](/blog/images/aws/Security_NLB-with-ACL.png)

- 可以在 NLB 所在的 public subnet 設定 Network ACL

- 也可以在 EC2 Instance 所在的 private subnet 設定 Network ACL

- EC2 Instance 本身可以設定 Security Group，或是 host-based firewall

- 適用於 Layer 4 的阻擋

## 使用 WAF + ALB

若是 EC2 Instance 提供的是 web service，且需要作到 Layer 7 的惡意行為阻擋，那就會需要用到 WAF：

![WAF + ALB](/blog/images/aws/Security_WAF-with-ALB.png)

- Layer 4 的部份，可以在 ALB 所在的 public subnet 設定 Network ACL

- Layer 7 的部份，則是在 WAF 上設定阻擋條件；WAF 可以看到真實的使用者來源

- EC2 Instance 則是在 Security Group 中設定僅限來自於 ALB 的流量

## 使用 CDN(CloudFront) + WAF + ALB

若是額外還需要做網頁靜態資源的加速，那就需要搭配 CDN(CloudFront)：

![WAF + CloudFront + ALB](/blog/images/aws/Security_WAF-with-ALB-and-CloudFront.png)

- CloudFront 是屬於 edge service，因此可以額外提供使用者所在位置的過濾方式

- 其他網路安全設定方式與上面相同



KMS(Key Management Service)
===========================

## 什麼 KMS?

首先看一下 AWS 官方對 KMS 的說明：
> AWS Key Management Service (KMS) makes it easy for you to create and manage cryptographic keys and control their use across a wide range of AWS services and in your applications. AWS KMS is a secure and resilient service that uses hardware security modules that have been validated under FIPS 140-2, or are in the process of being validated, to protect your keys. AWS KMS is integrated with AWS CloudTrail to provide you with logs of all key usage to help meet your regulatory and compliance needs.

其實簡單來說，就是幫使用者管理金鑰的服務，然後有很高的安全認證....

而 KMS 有以下特性：

- KMS 本身是屬於 Region 範圍的服務，用來協助金鑰的管理 (當然要加解密資料就必須跟這個服務進行整合囉!)

- 協助使用者管理名為 CMK(Customer Master Key) 的金鑰

- 非常適合用來對 S3 object、資料庫密碼、存在 System Manager Parameter Store 的 API key 進行加解密

- 加解密的來源資料，最大限制為 4KB

- 可與 CloudTrail 整合，提供稽核的功能 (相關的 log 資訊會存於 S3 中)

- 通過 FIPS 140-2 Level 2 的認證
> CloudHSM 達到 FIPS 140-2 Level 3

## CMK 的類型

CMK(Customer Master Key) 有三種，分別是：

- `AWS Managed CMK`：免費，由 AWS 負責管理，使用者碰不到，在使用各個 AWS 服務時會預設拿來做加解密用，但也只有 AWS 服務可以直接使用

- `Customer Managed CMK`：完全可由使用者定義 rotation、key policy(誰可使用? 在什麼情況下可用? ... 等等)，也可以開啟 or 關閉

- `AWS Owned CMK`：這完全由 AWS 負責管理，不會屬於特定使用者帳號下，也看不到，用來在多個帳號之間，確保使用者帳號使用服務時的安全性。


## Symmetric & Asymmetric CMK 的差異

在 KMS 中 CMK 有分為 Symmetric & Asymmetric 兩種，使用上有著不小的差異：

### Symmetric CMK

- 使用同樣一把金鑰進行資料加解密

- 使用 AES-256 加密

- AWS 不會提供沒加密過的金鑰內容

- 必須呼叫 KMS API 才能使用

- 與 KMS 整合的其他 AWS service 都使用 Symmetric CMK

- 可用來產生 data keys, data key pairs, random byte strings

- 可匯入自訂的 key material

### Asymmetric CMK

- public/private key pair

- 使用 RSA & ECC(Elloptic-Curve Crytography)，ECC 比 RSA 優秀

- AWS 不會提供沒加密過的 private key 內容

- 必須呼叫 KMS API 才能使用 private key

- public key 可以下載並在 AWS 之外使用

- 通常是提供給外部無法呼叫 KMS API 的使用者使用

- **AWS 其他服務無法與 Asymmetric CMK 整合**

- 常用於訊息的簽名 & 驗證簽名之用


## Key Policy

當使用程式 or AWS CLI 建立 CMK 時，可以自訂 key policy，但若是沒有指定 policy，AWS 則會預設提供一個 key policy，內容大概像以下：

```json
//允許 AWS account 為 root 的使用者有這個 CMK 的完整權限
{
  "Sid": "Enable IAM User Permissions",
  "Effect": "Allow",  
  "Principal": {"AWS": "arn:aws:iam::111122223333:root"},
  "Action": "kms:*", 
  "Resource": "*"
}
```
> 比較需要注意的是，若是不小心刪除了 key policy，使用者就無法存取 CMK 了；此時只能找 AWS 原廠協助重新設定 key policy 才能再度使用 CMK

以下是其他範例，用來指定特定的 User or Role 擁有 CMK 的權限：

```json
//提供給 Role "EncryptionApp" 呼叫資料加解密的權限
{
  "Sid": "Allow use of the key",
  "Effect": "Allow",  
  "Principal": {"AWS": "arn:aws:iam::111122223333:role/EncryptionApp"},
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey",
    "kms:Encrypt",
    "kms:GenerateDataKey*",
    "kms:ReEncrypt*"
  ],
  "Resource": "*"
}

//提供給 User "CMKUser" 呼叫資料加解密的權限
{
  "Sid": "Allow use of the key",
  "Effect": "Allow",  
  "Principal": {"AWS": "arn:aws:iam::111122223333:user/CMKUser"},
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey",
    "kms:Encrypt",
    "kms:GenerateDataKey*",
    "kms:ReEncrypt*"
  ],
  "Resource": "*"
}
```

## 實作注意事項

- 在 AWS Managed Keys 中，列出了目前已經與 KMS 整合的服務所使用的 key，這些都是由 AWS 管理，只要了解當前 AWS 有管理這些 key 即可

- 在 Customer Managed Keys 畫面中的內容，才是由使用者自行管理的部份

- 若是要將放在 region A S3 已經加密的檔案移動到 region B，那需要進行下列操作：
  1. 先將檔案解密(使用位於 region A 的 key)
  2. 將檔案移動到 region B
  3. 在使用 region B 的 key 加密
> KMS 的 key 有效範圍是 region，這很重要!

- 每個 CMK(Customer Managed Key) 建立出來後，用來識別只有 `KeyId`，但這是很難記住的一串亂碼，**可為每個 CMK 設定 alias 藉以方便識別**
> 使用 AWS CLI 的指令關鍵字為 `aws kms create-alias`

- CMK 建立後會預設給一組 key policy，內容就是給 root 這把 key 所有的權限

- CMK 可設定 rotate policy 為一年(也可以不設定)；AWS Managed Key 的 rotation policy 則是預設為三年

- 加密後的資料還會再額外進行 base64 encode；而將資料解密後，也還是 base64 encode 後的結果，因此還需要額外做 base64 decode 才能看到正確結果

- 加密資料時需要指定 Key ID，但解密時不需要，是因為加密時已經把 Key ID 相關資訊一起包進去了

- 若要加密超過 4KB 以上的資料，可以先產生一個 DEK(Data Encryption Key)，再用 DEK 進行資料加密
> 使用 AWS CLI 的指令關鍵字為 `aws kms generate-data-key`


CloudHSM
========

若是希望 key management 可以自己做，或是需要比 KMS 更高的安全性，那 CloudHSM 可能就是唯一選擇了!

## 什麼是 CloudHSM ?

- Dedicated Cloud Security Module (只服務單一使用者)
> single tenant, multi-AZ cluster

- 安全等級為 FIPS 140-2 Level 3 (KMS 只有 Level 2)

- 使用者透過 CloudHSM 自行管理 key

- 由於所有 key 都是自行管理，因此無法與 AWS managed service 整合
> AWS 完全無法存取 CloudHSM 中的 key

- 運行於 VPC 中

- 使用 industry-standard APIs (沒有任何 AWS API 可用)

- 若資料需要用以下加密標準進行加密，CloudHSM 是個合適的選擇：
  - PKCS#11
  - Java Cryptography Extensions (JCE)
  - Microsoft CryptoNG (CNG)

- key 若是遺失了，AWS 是無法協助的

## CloudHSM 如何應用?

以下是使用 CloudHSM 的一個範例：

![CloudHSM example](/blog/images/aws/CloudHSM_Example.png)

1. 首先，CloudHSM 要位於 VPC 中 (可用現有的，或是新建立)

2. CloudHSM 會建立 ENI interface，而通過該 ENI interface 的流量就等同於將資料傳送到 CloudHSM 處理
> 使用上很容易，就建立 EC2 instance，把 CloudHSM 產生的 ENI interface 附加上去即可

3. CloudHSM 並沒有 native HA，有需要的話就要自己做(例如上圖)



System Manager Parameter Store
==============================

服務在不同的環境運行時，總是會根據環境的不同，有不同的設定檔 & 服務存取密碼 ... 等資訊需要保存，而存放 git repository 肯定不是個好選擇，而 `Parameter Store` 正好可以用來解決這個問題。

## 什麼是 Paraneter Store?

- 屬於 AWS System Manager(SSM) 中的一個元件

- 是種安全的 Serverless storage，適合儲存設定檔、密碼、連線字串 ... 等服務運行所必要的資訊

- 存在 paramter store 中的資料可以是明碼純文字，也可以是 KMS 加密後的結果

- 資料可以是以階層式(Hierarchy)的方式儲存

- 提供版本管理的功能

- 可設定每筆資料的 TTL

總而言之，透過 Parameter Store，可以很方便的將程式碼與環境設定分離，進行更好的管理。

## 階層式(Hierarchy)資料儲存

階層式(Hierarchy)資料儲存的樣貌會是類似下圖：

![Parameter Store hierarchy example](/blog/images/aws/ParameterStore_Example.png)

若透過 AWS client SDK 中的 GetParameterByPath 來取得資料，以下是幾個存取路徑範例：

- /dev

- /dev/db

- /prod/app

## 使用存放在 Parameter Store 中的資料

### 與 CloudFormation 整合

以下為實際範例：

```yaml
Parameters:
  InstanceType :
    Type : 'AWS::SSM::Parameter::Value<String>'
    Default: myEC2TypeDev
  KeyName :
    Type : 'AWS::SSM::Parameter::Value<AWS::EC2::KeyPair::KeyName>'
    Default: myEC2Key
  AmiId:
    Type: 'AWS::EC2::Image::Id'
    Default: 'ami-60b6c60a'
    
Resources :
  Instance :
    Type : 'AWS::EC2::Instance'
    Properties :
       Type : !Ref InstanceType
       KeyName : !Ref KeyName
       ImageId : !Ref AmiId 
```

### Lambda Function

以下是幾個標準步驟：

#### 設定 IAM 存取權限

首先設定 policy，以下是範例：

```json
{
  "Version": "2012-10-17", 
  "Statement": [{
    "Effect": "Allow", 
    "Action": [
      "logs:CreateLogGroup", 
      "logs:CreateLogStream", 
      "logs:PutLogEvents", 
      "ssm:GetParameter",
      "ssm:GetParameterByPath"
    ], 
    "Resource": "*"
  }]
}
```

接著要建立一個 IAM execution role：

- 建立 type 為 `AWS service` 的 trusted entity，並選擇 Lambda

- 將上面的 policy 加入這個 role 中

#### 設定 KMS

此步驟是為了確保存在 Parameter Store 中的資料是經過加密的：

1. 建立 CMK(Customer Managed Key)，選擇 Symmetric

2. 指定 Key Usage Permission 為上一個步驟新增的 IAM role (讓 IAM role 有權限可以使用這把 Key)

#### 在 Parameter Store 中新增資料

這個部份只有以下重點：

1. 指定資料的儲存路徑，例如：`/prod/myapp/db/password`

2. 指定 type 為 `SecureString` (此型態的資料才會以 KMS key 進行加密)

3. 指定上述步驟建立的 KMS Key


#### 設定 Lambda Function

1. 新增 Lambda function，並**指定上述步驟設定的 IAM role**

2. 程式中只要指定要存取 Paramter Store 的 data path 即可成功取得解密後的資料
> 不用額外再指定 KMS key，因為這個部份已經在新增資料到 Parameter Store 就已經指定好


References
==========

- [AWS Certified Solutions Architect: Associate Certification Exam | Udemy](https://www.udemy.com/course/aws-certified-solutions-architect-associate/)

- [Key Management Service - Amazon Web Services (AWS)](https://aws.amazon.com/kms)

- [AWS CloudHSM](https://aws.amazon.com/cloudhsm)

- [AWS Systems Manager Parameter Store - AWS Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)