---
layout: post
title:  "[Packer] 學習重點節錄"
description: "此篇文章是學習 Udemy 課程 \"HashiCorp Packer - Build automated machine images\" 時，將學習 HashiCorp Packer 過程中理解到的重要觀念 & 使用方式進行統整並分享"
date: 2021-01-09 21:50:00
published: true
comments: true
categories:
  - DevOps
tags:
  - DevOps
  - Packer
---

Overview
========

[HashiCorp Packer](https://www.packer.io/) 是一個可以用來打包 Image 的工具，像是 Amazon AMI、Docker Image .... 等等，透過 Packer 可以將打包 image 的工作標準化並且以更有效率的方式進行。

因此如果在工作上有需要在各個不同的 cloud provider 平台上進行 machine image 的打包，那 Packer 肯定會是一個可以幫上很大的忙。



常用場景
=======

- 可加入到 CI/CD pipeline 中，用來產生各種環境所需要的 machine image

- 可與其他工具(例如：Consul, Terraform, Vagrant)整合

- 透過製作 image 的方式，讓多個環境(development/testing/staging/production)所使用的環境可以保持一致


整合進到 CI/CD pipeline，大概會是下面這個模樣：(裡面的工具是可以替換的)

![Packer - Use Case 1](/blog/images/devops/packer/packer-use-case-1.png)

1. 當 developer 將程式開發完成後，會將程式送到 GitHub 上

2. 觸發 CI server(上圖以 Jenkines 為例)，開始進行自動化工作，透過 packer 並搭配 Ansible/CHEF/Pupput 這類的 configuration management tool，打包成新的 AWS EC2 AMI

3. 接著透過 terraform，使用上一個步驟產生的 AMI，產生新的 EC2 instance 

> 當然，在 microservice 的時代，可能就會改成打包成 container image 後，放到 Kubernetes 中運行



常見術語
=======

介紹在 Packer 中需要了解的術語之前，可以先看一下下面這張圖：

![Packer - Terminology](/blog/images/devops/packer/packer-terminology.png)

這張圖展示了在不同的階段，是由下面哪些元件發生了作用，可以當作一個流程圖來參考：

## Builds

- 用來定義要產生 machine image 用的 task，而這些定義的內容是與被要給 builder 所拿來處理用，例如以下範例：

```json
{
   "builders":[
      {
         "type":"amazon-ebs",
         "access_key":"YOUR_ACCESS_KEY",
         "secret_key":"YOUR_SECRET_KEY",
         "region":"us-east-1",
         "source_ami":"ami-0885b1f6bd170450c",
         "instance_type":"t3a.nano",
         "ssh_username":"ubuntu",
         "ami_name":"packer-demo-1"
      }
   ]
}
```
> Amazon EC2 支援的 Builder type 有 `amazon-ebs`、`amazon-instance`、`amazon-chroot` ... 等等

- 如果要一次打包多個 cloud provider 的 Machine Image，可以同時有多個 build 同時運作

## Builders

- Packer 的元件之一，主要用來運行 & 產生 machine image

- 根據 build 中定義的需求，build process 中可能會呼叫不同的 builder 來製作 machine image (例如：Virtual Box, VMware, Amazon EC2、GCC ... etc)

## Artifacts

- 單一 build 的產出內容，基本上就是 machine image

- 每個 build 都會產生一個 artifact

- 以 AWS 為例，可以為每一個 region 都產生一個 machine image (AMI 屬於 region level，並非 global level)

## Provisioners

- Packer 的元件之一，用來在運行中的 image(其實就是某個 instace) 中安裝所需要的軟體、工具、設定 .... etc

- 可用執行這件任務的工具，有 Ansible、Puppet、Chef、Shell Script .... 等等

## Post-processors

- 可以針對 provisioner 產生出來的 image 再進行後續的處理，甚至是產生新的 machine image

- 常在此階段進行的工作可能像是壓縮 artifacts、將 artifact 上傳到特定的 repository ... 等等

## Templates

- 預先定義好的 JSON 檔案，可讓 packer 用來產生新的 machine image



Builder
=======

常用的 builder 除了 [Amazon AMI](https://www.packer.io/docs/builders/amazon) & [Docker](https://www.packer.io/docs/builders/docker) 之外，還有一些特殊的 builder 也可以了解一下：


## [Null Builder](https://www.packer.io/docs/builders/null)

Null Builder 啥也不做，因此它的用途其實是用來加速 debug 後面的 Provisioner & Post-processor 用的，**因為一般正常的 builder 會花費大量的時間來處理工作**，相關的使用細節可以查詢 [Null Builder 官網文件](https://www.packer.io/docs/builders/null)


## [File Builder](https://www.packer.io/docs/builders/file)

用途是在 builder 端產生一個檔案，看似好像是個沒啥用途的 builder，但其實常用來作為 debug 之用



建立 Amazon AMI 需要的最低權限
===========================

使用 Packer 建立 Amazon AMI 需要有 EC2 特定的權限，必須設定 IAM Policy + IAM Role + IAM User(Assume Role) 的方式取得可用的權限。

基本上製作 Amazon AMI 會有以下流程產生：

1. 從指定的 AMI 中產生 EC2 instance

2. 根據 provision 的設定內容，安裝相關套件，進行各種設定

3. 將 EC2 instance 關機

4. 為關機後的 EC2 instance 進行 snapshot 產生 AMI

5. 移除 EC2 instance

因此以下是製作 AWS IAM 的最小範圍權限：([官網文件來源](https://www.packer.io/docs/builders/amazon#iam-task-or-instance-role))

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:CopyImage",
        "ec2:CreateImage",
        "ec2:CreateKeypair",
        "ec2:CreateSecurityGroup",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteKeyPair",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSnapshot",
        "ec2:DeleteVolume",
        "ec2:DeregisterImage",
        "ec2:DescribeImageAttribute",
        "ec2:DescribeImages",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeRegions",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSnapshots",
        "ec2:DescribeSubnets",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume",
        "ec2:GetPasswordData",
        "ec2:ModifyImageAttribute",
        "ec2:ModifyInstanceAttribute",
        "ec2:ModifySnapshotAttribute",
        "ec2:RegisterImage",
        "ec2:RunInstances",
        "ec2:StopInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*"
    }
  ]
}
```


Provisioner
===========

## Shell Local(Remote)

兩者個差異在於 **Shell Script 所執行的位置**不同：

- [`Shell Local`](https://www.packer.io/docs/provisioners/shell-local)：script 會執行在 builder 所在的環境內

- [`Shell Remote`](https://www.packer.io/docs/provisioners/shell)：script 會執行在遠端用來打包 & 產生 machine image 的機器上


## Ansible Local(Remote)

兩者的差異在 **Ansible 所在的位置**不同：

- [`Ansible Local`](https://www.packer.io/docs/provisioners/ansible-local)：Ansible 是安裝在遠端用來打包 & 產生 machine image 的機器上

- [`Ansible Remote`](https://www.packer.io/docs/provisioners/ansible)：Ansible 安裝在執行 packer 的環境中

實際上在應用上，`Ansible Local` 可能會是相對方便的方法，因為 Builder 環境使用的 user 可能跟 remote 環境不同，因此還要各自指定正確的 user 才可以使用 Ansible Remote (不同的 AMI 會有其不同的 sudoer)。


## Provision 常用的 Parameter

- 在 [parallel build](https://learn.hashicorp.com/tutorials/packer/getting-started-parallel-builds) 的場景下，在 `provisioners` 中可以指定只針對 build 完後特定幾個的 artifact 進行後續的 provision 的工作

- 也可以透過 [`override`](https://www.packer.io/docs/templates/provisioners#build-specific-overrides) 關鍵字，為特定 build 的 artifact 指定不一樣的 provision 操作



Post Processors
===============

## [Artifice](https://www.packer.io/docs/post-processors/artifice)

Artifice post-processor 的功能在於，覆蓋 provision 階段產生出來的 artifact，並將 post-processor 後續的處理都轉到 Artifice post-processor 指定的檔案上，以下是一個簡單範例：

```json
{
  "builders": [
    {
      "type": "vmware-vmx",
      "source_path": "/opt/ubuntu-1404-vmware.vmx",
      "ssh_username": "vagrant",
      "ssh_password": "vagrant",
      "shutdown_command": "sudo shutdown -h now",
      "headless": "true",
      "skip_compaction": "true"
    }
  ],
  "provisioners": [
    {  //安裝 consul
      "type": "shell",
      "inline": [
        "sudo apt-get install -y python-pip",
        "sudo pip install ifs",
        "sudo ifs install consul --version=0.5.2"
      ]
    },
    {  //下載 consul binary
      "type": "file",
      "source": "/usr/local/bin/consul",
      "destination": "consul",
      "direction": "download"
    }
  ],
  "post-processors": [
    [
      {  //指定後續的 post-processor 改成對下載的 consul binary 進行處理
        "type": "artifice",
        "files": ["consul"]
      },
      {  //壓縮
        "type": "compress",
        "output": "consul-0.5.2.tar.gz"
      },
      {  //將壓縮後的結果上傳到 aws s3 bucket
        "type": "shell-local",
        "inline": [
          "/usr/local/bin/aws s3 cp consul-0.5.2.tar.gz s3://<s3 path>"
        ]
      }
    ]
  ]
}
```


## [Manifest](https://www.packer.io/docs/post-processors/manifest)

第一次使用 Packer 打包 machine image 時，當時就有個疑問：當 image 打包完，我如果要透過程式取得打包後產生的 AMI ID，這要怎麼做?

當時想，該不會要我去 parsing packer output 吧...? 這樣實在太 low 了，而 `Manifest` 就為這樣的需求提供了一個很好的解法。

透過 Manifest post-processor，就可以將打包過程中的重要資訊以 json 的格式產生出來，而每次打包個過程都會更新 manifest，這讓後續的自動化變得容易串接了。



References
==========

- [HashiCorp Packer - Build automated machine images | Udemy](https://www.udemy.com/course/hashicorp-packer-build-automated-machine-images/)

- [Documentation | Packer by HashiCorp](https://www.packer.io/docs)