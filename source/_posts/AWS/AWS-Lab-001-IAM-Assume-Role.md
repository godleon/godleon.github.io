---
layout: post
title:  "[AWS] Lab 001 - 實作 IAM Assume Role with Terraform & GitHub Action"
description: "本篇文章的目的是要弄清楚 IAM User & EC2 Instance 如何進行 Assume Role 的實作細節，並透過 Terraform & GitHub Action 進行實作 & 驗證"
date: 2021-01-16 18:30:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - IAM
  - Terraform
  - GitHub Actions
---

完整的實作程式碼可以參考[此專案(godleon/aws-labs at lab001-iam-assume-role)](https://github.com/godleon/aws-labs/tree/lab001-iam-assume-role)


觀念說明
=======

從之前學習 IAM 的幾篇文章，可以了解到為了要取得更好的安全性，在權限管理的作法上，應該採用以下幾個方式：

1. 設定 IAM User，但僅有 Assume Role 的權限 

2. 根據 AWS resource 存取需求制定 IAM policy

3. 根據規劃設定所需要的 IAM Role，並與上一個步驟建立的 IAM policy 繫結；並設定 Trust Relationship，允許特定的 IAM User 可以 Assume 成這個 IAM Role

到目前為止，已經可以讓 IAM User 透過 AssumeRole action 來切換成指定的 IAM Role 來取得所需要的權限。

但此時還無法讓 EC2 Instace 取得指定 IAM Role 的權限，這時候還需要補上一個 IAM instance profile 的設定給 IAM Role，讓這個 IAM Role 可以套用在 EC2 Instance Level 上。



實作流程
=======

由於實作的過程使用 Terraform，會遇到 Resource 互相參照的情況，應該資源佈建的流程要從最末端開始來，因此順序會變成：

1. IAM Policy

2. IAM Role

3. IAM User

## 建立 IAM Policy

首先我們可以透過以下設定建立一個 IAM Policy：

```hcl
resource "aws_iam_policy" "s3_management" {
  name = "s3_management"
  path = "/storage/"

  policy = <<-EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Action": [
          "s3:*"
        ],
        "Effect": "Allow",
        "Resource": "*"
      }
    ]
  }
  EOF
}
```

透過上面語法建立的 IAM Policy，因為加了 `path` 設定，policy ARN 就會變成 `arn:aws:iam::[ROOT_ACCOUNT_ID]:policy/storage/s3_management`。


## 建立 IAM Role

接著是建立 IAM Role：

> Trust Relationship 的部份要設定在 resource `aws_iam_role` 中；實際 AWS resource 的 access policy 則是要定義在 resource `aws_iam_policy` 中

```hcl
# 建立 instance profile，才能讓 IAM Role 套用在 EC2 instance 上
resource "aws_iam_instance_profile" "profile-robot" {
  name = "profile-robot"
  role = aws_iam_role.robot.name
}

# 若是想要接受特定 root user 下的所有 IAM User 都可以 AssumeRole
# Principal 的部份可以改成類似 => "AWS": "arn:aws:iam::777777777777:root"
resource "aws_iam_role" "robot" {
  name = "robot"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }, 
    {
        "Effect": "Allow",
        "Principal": { 
            "AWS": "${aws_iam_user.robot.arn}"
        },
        "Action": "sts:AssumeRole",
        "Condition": {}
    }
  ]
}
EOF

  tags = {
    Name = "robot"
  }
}

resource "aws_iam_role_policy_attachment" "robot-s3_management" {
  role       = aws_iam_role.robot.name
  policy_arn = aws_iam_policy.s3_management.arn
}
```

上面的設定會建立一個 Role，並設定 Trust Relationship 有兩個部份，分別是：

1. `"Service": "ec2.amazonaws.com"`：這就是為了要套用在 EC2 instance 上時能生效的設定

2. `"AWS": "${aws_iam_user.robot.arn}"` 則是等一下會建立個 IAM User ARN reference，為了要讓 IAM User 可以自行切換 IAM Role 之用


## 建立 IAM User

最後這一步就是要建立 IAM User，然後只要給 AssumeRole 的權限即可：

```hcl
resource "aws_iam_user" "robot" {
  name = "robot"
  path = "/auto/"

  tags = {
    Name = "robot"
  }
}

# 設定 AssumeRole 的 IAM Policy
# 讓 IAM User 可以切換到指定的 IAM Role
resource "aws_iam_user_policy" "assume_role-robot" {
  name = "assume_role-robot"
  user = aws_iam_user.robot.name

  policy = <<-EOF
  {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "${aws_iam_role.robot.arn}"
        }
    ]
  }
  EOF
}
```


驗證
===

## EC2 Instance 套用 IAM Role

這個部份驗證其實很簡單，流程如下：

1. 啟動一個 EC2 instance，選擇 Amazon AMI (啟動就會自帶 AWS CLI)，過程中選擇 IAM Role 為上面的 IAM Role `robot`

2. 成功啟動後，SSH 連入 instance，輸入 `aws s3 ls` 就會出現帳號底下存在的 S3 bucket

但需要注意的是，**每次只有一個 IAM role 可以被分配給 EC2 instance，在 EC2 instance 上的所有 application 都會共用此 IAM role 的權限**。


## IAM User 切換 IAM Role

若是 IAM User 可以進入 AWS Console，那就可以透過 AWS Console 畫面右上角的選項切換 Role。

但如果只有 program access 的權限，那就要準備好 `~/.aws/credentials`(需要 Access/Secret Key) & AWS CLI v2，並透過以下方式 Assume Role：(**EC2 instance 就不需要指定 IAM Role 了**)

```bash
# 透過 Assume Role 的操作取得一個臨時的 Access/Secret Key & Session Token
$ aws sts assume-role --role-arn "arn:aws:iam::568846173001:role/robot" --role-session-name AWSCLI-Session
{
    "Credentials": {
        "AccessKeyId": "ASIAYI4O7YNEZ4NKESNO",
        "SecretAccessKey": "gZ77UjALYjYho47PiAWIqpqzADSuKzW+0HOdJ/+0",
        "SessionToken": "IQoJb3JpZ2luX2VjEE4.....",
        "Expiration": "2021-01-16T06:54:31+00:00"
    },
    "AssumedRoleUser": {
        "AssumedRoleId": "AROAYI4O7YNE23XKHJR2Z:AWSCLI-Session",
        "Arn": "arn:aws:sts::777777777777:assumed-role/robot/AWSCLI-Session"
    }
}

# 設定 Assume Role 所需要的環境變數
$ export AWS_ACCESS_KEY_ID=ASIAYI4O7YNEZ4NKESNO
$ export AWS_SECRET_ACCESS_KEY=gZ77UjALYjYho47PiAWIqpqzADSuKzW+0HOdJ/+0
$ export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjEE4.....

# 查詢目前的身份
$ aws sts get-caller-identity
{
    "UserId": "AROAYI4O7YNE23XKHJR2Z:AWSCLI-Session",
    "Account": "777777777777",
    "Arn": "arn:aws:sts::777777777777:assumed-role/robot/AWSCLI-Session"
}

# 已經可以成功存取 S3 了!
$ aws s3 ls
2020-06-30 02:38:23 your-bucket
```


在 GitHub Action 的應用
======================

由於 GitHub Action Runner 是一個獨立運行的 VM instance，無法使用將 IAM Role 套用在 EC2 instance 的方式來賦予身份(除非使用 self-hosted runner)，那就可以透過 IAM User Assume Role 的方式取得 IAM Role 權限。

> 需要先設定三個 repository secret，分別是 `AWS_ASSUME_ROLE_ARN` & `AWS_ACCESS_KEY_ID` & `AWS_SECRET_ACCESS_KEY`

以下是 GitHub Action 的 workflow 內容：

```yaml
name: AWS Assume Role

on: push

jobs:
  aws-assume-role:
    runs-on: ubuntu-latest
    steps:
      - name: Assume Role
        uses: docker://abatilo/aws-assume-role-action:v1.0.0
        env:
          ACTIONS_ALLOW_UNSECURE_COMMANDS: true
          ROLE_ARN: ${{ secrets.AWS_ASSUME_ROLE_ARN }}
          ROLE_SESSION_NAME: testing
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          DURATION_SECONDS: 120
          
      - name: Identity Check
        run: |
          aws sts get-caller-identity

      - name: Show S3 Bucket List
        run: |
          aws s3 ls
```

利用現成的 GitHub Marketplace Action，就可以很快將上面的實驗過程自動化，再根據需求串到其他自動化流程中。



References
==========

- [Using an IAM role to grant permissions to applications running on Amazon EC2 instances - AWS Identity and Access Management](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2.html)

- [Assume an IAM role using the AWS CLI](https://aws.amazon.com/premiumsupport/knowledge-center/iam-assume-role-cli/)

- [AWS Assume Role Action · Actions · GitHub Marketplace](https://github.com/marketplace/actions/aws-assume-role-action)