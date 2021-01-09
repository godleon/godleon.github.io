---
layout: post
title:  "[AWS EKS] 從零打造一個可運行 Fargate workload 的 EKS cluster"
description: "此篇文章是介紹如何從零打造一個可用來運行 Fargate workload 的 EKS cluster，並介紹整個過程中需要了解的相關流程 & 重要的權限設定觀念"
date: 2021-01-09 07:35:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - EKS
  - IAM
---


前言
===

寫這篇文章的原因是因為，想要透過手動建立的流程，搞清楚建立一個 EKS cluster for pure Fargate workload 需要經過那些步驟，設定那些權限，以及在使用上有什麼需要注意的地方。


預先準備
=======

由於在整個過程中會用到 AWS CLI、kubectl、eksctl，因此可以透過以下方式進行安裝：

## 安裝 AWS CLI v2

v2 是全新的版本，因此建議直上 v2(v1 還需要安裝 python library)，以下是安裝步驟：

```bash
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install
```

## 安裝 kubectl

這部份可以看一下[官網文件](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)目前提供的是什麼版本，基本上根據 EKS 的版本選擇對應的 kubectl 會比較沒有問題。

以下以 `1.18.9` 為例，安裝 kubectl：

```bash
$ curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
$ chmod +x ./kubectl
$ mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin

# 或是直接移到 /usr/loca/bin/ 目錄下也行
$ sudo mv ./kubectl /usr/loca/bin/
```

## 安裝 eksctl：

eksctl 可以在某些時候透過 CloudFormation 幫使用者建立較為複雜的資源(例如：EKS cluster、IAM service account ... 等等)，可以讓使用者很快的入手，以下是安裝 eksctl 的流程：

```bash
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/eksctl /usr/local/bin
$ eksctl version
```

安裝好之後設定 `${HOME}/.aws/config` & `${HOME}/.aws/credentials` 兩個檔案，確認有足夠的權限 & 正確的 region 設定，才可以繼續往下進行。

`${HOME}/.aws/config` 範例：

```ini
[default]
region=us-west-2
output=json
```

`${HOME}/.aws/credentials` 範例：

```ini
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

> 比較需要注意的是，透過 eksctl 建立資源，雖然很快且很方便，但其實很容易忽略掉實際上需要熟悉的地方(特別是權限管理的部份)，而這些地方往往是查找問題時的重要關鍵，因此建議還是另外要找時間將正確的相關知識補上(特別是 IAM、OIDC Provider、Kubernetes Service Account & RBAC .... 等觀念)



權限設定
=======

## 建立 EKS cluster 所需權限

從 IAM 中建立一個 role，而這個 role 的功能是要套用在 EKS cluster 身上，並賦予其有足夠的權限去操控其他的 AWS 資源，達成自動化管理相關 AWS resource 的目的，流程如下：

1. 進入 **IAM console**，選擇 `Role -> Create role`

2. **trusted entity** 選擇 `AWS service`，**use case** 則是選擇 EKS cluster

3. 設定 tag & review(如果有需要)，最後則是輸入一個 unique role name(例如 `eksClusterRole`)，到這邊就完成了 role 的建立

上面的流程完成後，會自動會幫忙帶入 AWS 預先設定好的 IAM Policy `AmazonEKSClusterPolicy` 給這個 IAM Role；此外，在 Trust Relationship 的部份，則會是以下設定：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
> 上面表示 `eks.amazonaws.com` 這個服務可以 assume 成上面建立的 IAM Role 來取得存取 AWS resource 的權限


## 關於權限的說明

當 EKS cluster 被建立時，用來進行操作建立 cluster 的行為的 IAM entity(user or group) 會被加入到 Kubernetes RBAC 認證資訊中，並被設定為 administrator(具備 `system:masters` 的權限)，因此當 EKS cluster 剛建立好，就只有這個 IAM entity 能對 cluster 進行操作。

因此，若是透過 AWS console 建立的 EKS cluster，那就要確認執行操作 EKS cluster 的環境所具備的 AWS credential 是跟建立 EKS cluster 的 IAM entity 相同，才可以正常運作。

但若是要讓其他使用者也可以透過自己的 AWS 帳號取得管理 EKS cluster 的權限，可以參考[官網文件(Managing users or IAM roles for your cluster - Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)的說明。
> 基本上就是調整 namespace `kube-system` 中的 configmap `aws-auth` 來提供權限

以下是一個簡單的範例：

```yaml
apiVersion: v1
data:
  mapRoles: |
    ..... (ignored)
  mapUsers: |
    - userarn: <arn:aws:iam::111122223333:user/admin>
      username: <admin>
      groups:
        - <system:masters>
    - userarn: <arn:aws:iam::111122223333:user/ops-user>
      username: <ops-user>
      groups:
        - <system:masters>
```



設定 EKS cluster Network
========================

為了安全起見，至少要設定 `2 public subnets + 2 private subnets`，透過[此連結](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-console.html#vpc-public-private2) 中的說明，可以很快在 VPC 中建立出所需要的網路。

另外，EKS cluster 本身要求在建置時就必須是 HA 的狀態，因此跨 AZ 的 subnet 設定是必要的。



建立 EKS cluster
===============

建立 EKS cluster 除了 **Name**,**Kuberentes version** 之外，還有一些重要的部份需要了解： 

- `Cluster service role`：這個部份就是要選擇在上一個步驟設定的 `eksClusterRole` (當然選擇其他的 Role 也可以，但要確定有正確的 IAM Policy 才可以用)

- `Secrets encryption`：若是啟用 `envelope encryption`，在 EKS clustere 中的 secret 就會被加密(使用自己選擇的 Customer Master Key)，而 CMK 必須在同一個 region 中，若是 CMK 是另外一個帳號所建立，那就要確保目前這個 user 有權限存取該 CMK

- EKS 需要 VPC HA，因此 subnet 選擇上需要 cross AZ；若要安全性好一點，就將 private subnets & public subnets 分開

- 若要自訂 service IP address range 是可以的(這是 k8s 內部用的 IP)，但要記得不要跟 VPC 上設定的網段衝突

- 若是選擇 Kubernetes 1.18 版本以上，需要安裝 **AWS VPC CNI add-on**，因為 EKS cluster 在網路部份的操作會與 AWS VPC 有大量聯動，同時 AWS VPC Flow Log 功能才可以有作用



產生 kubeconfig
===============

這個步驟需要使用 AWS CLI 來完成，因此要確保要有足夠的權限才能運行以下操作：

```bash
# 先確認目前 AWS CLI 的使用者是誰
$ aws sts get-caller-identity

# 產生 kubeconfig
$ aws eks --region ap-northeast-1 update-kubeconfig --name eks-test
..... (訊息忽略)

# 測試是否可以連上 EKS cluster
$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   157m

# 透過 eksctl 測試
$ eksctl get cluster
NAME		REGION		EKSCTL CREATED
eks-test	ap-northeast-1	False

# 取得指定 cluster 的 addon 資訊
$ eksctl get addon --cluster=eks-test
[ℹ]  Kubernetes version "1.18" in use by cluster "eks-test"
[ℹ]  getting all addons
[ℹ]  to see issues for an addon run `eksctl get addon --name <addon-name> --cluster <cluster-name>`
NAME	VERSION			STATUS	ISSUES	IAMROLE	UPDATE AVAILABLE
vpc-cni	v1.7.5-eksbuild.1	ACTIVE	0
```



設定 Fargate
===========

## 設定 IAM Role for Fargate Pod

設定權限的工作又來了，這次要設定的是 Farget Pod 所使用的 IAM Role。

透過選擇 `EKS -> EKS Fargate pod` 所得到的權限內容如下：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage"
            ],
            "Resource": "*"
        }
    ]
}
```

在 Trust relationship 中確認內容如下：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks-fargate-pods.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
> 以上設定表示 service `eks-fargate-pods.amazonaws.com` 被信任可以執行 `sts:AssumeRole` 操作"


## 建立 Fargate profile

由於預計所有 workload 都以 Fargate 的方式運行，因此設定的方向如下：

1. 一開始就納入 namespace `kube-system` & `default` 的 workload 

2. 不設定其他的 pod selector，讓所有 pod 都以 Fargate 方式運行

接著透過 AWS Console：`Cluster(eks-test)` -> `Configuration` -> `Compute` -> `Fargate Profiles`

設定內容大致可以參考以下的 JSON 設定：

```json
{
    "fargateProfileName": "init-fargate",
    "clusterName": "eks-test",
    "podExecutionRoleArn": "arn:aws:iam::777777777777:role/AmazonEKSFargatePodExecutionRole",
    "subnets": [
        "subnet-09973115f7807ae78",
        "subnet-030e2fe1637aeee0a"
    ],
    "selectors": [{ "namespace": "kube-system" }, { "namespace": "default" }, { "namespace": "monitoring" }, { "namespace": "kubernetes-dashboard" }]
}
```
> 以上會讓 `kube-system`、`default`、`monitoring`、`kubernetes-dashboard` 這幾個 namespace 中的 pod 以 fargate 的形式運作


或是將上面的設定存成 JSON 檔案後再匯入：(假設檔名是 `init-fargate.json`)

```bash
$ aws eks create-fargate-profile --cli-input-json file://init-fargate.json.json
```

## 修復 CoreDNS

由於 CoreDNS 服務預設是要運作在 worker node 上的，但如果我們希望所有 workload 都運行在 fargate 上，那就沒有 worker node 可用了，因此 CoreDNS 服務自然也就無法正常運作，而 k8s cluster 內部的 DNS 解析就會有問題。

那要解決這個問題也不是太困難，基本上就是要完成以下工作：

1. 讓 CoreDNS 改成以 fargate 的方式運行，因此要額外增加一個 fargate profile for CoreDNS (由於上面的 Fargate Profile 範例中已經涵蓋了 `kube-system` namespace，所以這個部份就可以忽略了，因為 CoreDNS 是屬於 `kube-system` namespace)

2. 通知 EKS cluster，CoreDNS deployment 不要運行在 worker node 上

以下是執行相關命令：

```bash
# 移除 CoreDNS 運行在 Worker Node 的設定 (移除特定的 annotation)
$ kubectl patch deployment coredns -n kube-system --type json \
-p='[{"op": "remove", "path": "/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
```

接著 k8s 就會自動移除原本的 CoreDNS pod，重新改由 Fargate pod 來置換。



整合 Kuberentes Service Account & IAM
=====================================

除了上面的 EKS Cluster Role 之外，還有兩個很重要的部份需要設定：

1. k8s 中的 RBAC 機制，取決於每個 pod 在 k8s cluster 中的權限範圍在所使用的 service account 為何；但若是 pod 要存取 AWS resource 呢? 那就必須與 IAM role 繫結，因此要為 EKS cluster 中的 service account 啟用 IAM Role 的功能

2. 由於 EKS Cluster 的網路是由 VPC CNI plugin 所管理，而網路、Load Balancer ...等資源的建立，都會與 VPC 連動，因此如何正確給予 VPC CNI plugin 正確的權限也是要完成的部份


## 為 Service Account 啟用 IAM Role

這必須透過將 EKS OIDC Provider & IAM WebIdentity 兩個機制串起來達成，因此大方向大概是降：

1. EKS cluster 建立時，會產生一組 OIDC Provider URL

2. 在 AWS IAM 中，新增一個 Identity Provider，指定 Provider type 為 `OpenID Connect`(這是 k8s 支援的認證機制)，將 EKS cluster 的 OIDC Provider 填入

完成了以上的設定後，基本上 k8s & IAM 溝通的橋樑就搭起來了。

而 eksctl 很簡單的幫我們完成了這些事情，只要下一個指令即可；首先要確認 eksctl 的版本在 `0.34.0` 以上，接著執行下面指令：

```bash
# 必須使用 eksctl 0.34.0 以上
$ eksctl version
0.34.0

# eksctl utils associate-iam-oidc-provider --cluster <cluster_name> --approve
$ eksctl utils associate-iam-oidc-provider --cluster eks-test --approve
[ℹ]  eksctl version 0.34.0
[ℹ]  using region ap-northeast-1
[ℹ]  will create IAM Open ID Connect provider for cluster "eks-test" in "ap-northeast-1"
[✔]  created IAM Open ID Connect provider for cluster "eks-test" in "ap-northeast-1"
```

如此一來 k8s service account 就有一條橋樑與 IAM role 溝通了!


## 賦予 VPC CNI plugin 正確的權限

### 觀念釐清

首先先釐清在這個步驟中要進行的事情：

1. 建立一個 Kuberentes Service Account，並給予可以操控 VPC 的權限 (相關權限內容的定義可以參考 `arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy`)

2. 重新安裝 VPC CNI plugin，讓 CNI plugin 擁有足夠的權限去控制 VPC

### 實做

要做這件事情，必須要有 `eksctl 0.34.0` 版本以上：

```bash
$ eksctl version
0.34.0


# 這是一開始 VPC CNI plugin 的狀態
# IAMROLE 欄位並沒有任何資訊
$ eksctl get addon --cluster=eks-test
[ℹ]  Kubernetes version "1.18" in use by cluster "eks-test"
[ℹ]  getting all addons
[ℹ]  to see issues for an addon run `eksctl get addon --name <addon-name> --cluster <cluster-name>`
NAME	VERSION			STATUS	ISSUES	IAMROLE	UPDATE AVAILABLE
vpc-cni	v1.7.5-eksbuild.1	ACTIVE	0


# 移除原本的 iamserviceaccount (若是第二次建立 EKS cluster，原本建立的 iamserviceaccount 不會消失)
$ eksctl delete iamserviceaccount --name aws-node --namespace kube-system --cluster eks-test

# 建立一個用來執行 VPC CNI 相關操作的 service account "aws-node"
# 會位於 kube-system namespace 中
# 這個過程會建立一個 IAM Role，並將 attach-policy-arn 指定的 IAM policy 繫結上去
# 並且為 service account 加上一個 annotation，並設定成 IAM Role ARN
$ eksctl create iamserviceaccount \
    --name aws-node \
    --namespace kube-system \
    --cluster eks-test \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
    --approve \
    --override-existing-serviceaccounts
[ℹ]  eksctl version 0.34.0
[ℹ]  using region ap-northeast-1
[ℹ]  1 iamserviceaccount (kube-system/aws-node) was included (based on the include/exclude rules)
[!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
[ℹ]  1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kube-system/aws-node", create serviceaccount "kube-system/aws-node" } }
[ℹ]  building iamserviceaccount stack "eksctl-eks-test-addon-iamserviceaccount-kube-system-aws-node"
[ℹ]  deploying stack "eksctl-eks-test-addon-iamserviceaccount-kube-system-aws-node"
[ℹ]  serviceaccount "kube-system/aws-node" already exists
[ℹ]  updated serviceaccount "kube-system/aws-node"

# 檢視一下 service account aws-node 的狀況
# 可以看出這個 service account 被對應到了一個特定的 IAM Role (eksctl-eks-test-addon-iamserviceaccount-kube-Role1-1PDPBEXZPXHWH)
# 這個 IAM Role 也是在上一個步驟被建立出來的
$ kubectl -n kube-system describe sa/aws-node
Name:                aws-node
Namespace:           kube-system
Labels:              k8s-app=aws-node
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::777777777777:role/eksctl-eks-test-addon-iamserviceaccount-kube-Role1-IFKT0GBAZFVD
Image pull secrets:  <none>
Mountable secrets:   aws-node-token-8w78p
Tokens:              aws-node-token-8w78p
Events:              <none>
```

完成後，必須要移除 & 重新佈署 VPC CNI plugin，才會取得正確的 IAM role 設定：

```bash
# 移除 VPC CNI plugin
$ eksctl delete addon --cluster eks-test --name vpc-cni
[ℹ]  Kubernetes version "1.18" in use by cluster "eks-test"
[ℹ]  deleting addon: vpc-cni
[ℹ]  deleted addon: vpc-cni
[ℹ]  deleting associated IAM stacks
[ℹ]  will delete stack "eksctl-eks-test-addon-vpc-cni"

# 重新佈署 VPC CNI plugin
$ eksctl create addon --name vpc-cni --cluster eks-test --service-account-role-arn arn:aws:iam::777777777777:role/eksctl-eks-test-addon-iamserviceaccount-kube-Role1-IFKT0GBAZFVD
[ℹ]  Kubernetes version "1.18" in use by cluster "eks-test"
[ℹ]  creating role using recommended policies
[ℹ]  deploying stack "eksctl-eks-test-addon-vpc-cni"
[ℹ]  creating addon
[ℹ]  successfully created addon

# 重新檢視 VPC CNI plugin 資訊，在上面所產生
$ eksctl get addon --cluster eks-test
[ℹ]  Kubernetes version "1.18" in use by cluster "eks-test"
[ℹ]  getting all addons
[ℹ]  to see issues for an addon run `eksctl get addon --name <addon-name> --cluster <cluster-name>`
NAME	VERSION			STATUS	ISSUES	IAMROLE												UPDATE AVAILABLE
vpc-cni	v1.7.5-eksbuild.1	ACTIVE	0	arn:aws:iam::777777777777:role/eksctl-eks-test-addon-iamserviceaccount-kube-Role1-1PDPBEXZPXHWH
```

完成到這邊，VPC CNI plugin 已經有權限可以操作 VPC 並建立網路資源了!


## 如果上面的設定不用 eksctl 做，要如何完成?

不用 eksctl 做就真的會麻煩不少，我想就是因為這樣，AWS 才會開發出一個 eksctl 給大家用，方便使用者可以很快的進入佈署 workload 的階段。

但基本上的流程是這樣：

1. 首先要建立 k8s & IAM 的溝通橋樑，這個部份就要在 AWS IAM 中新增 Identity Provider(type 為 `OpenID Connect`)，並將 EKS cluster 提供的 OIDC Provider URL 填上去，這樣 IAM 才會認得從特定 EKS cluster 送過來的權限相關需求

2. 根據要存取的 AWS resource 範圍，設定合適的 IAM Policy (例如上面已經有現成的 `arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy`)

3. 新增 IAM Role，並與設定完成的 IAM Policy 繫結，取得 IAM Role ARN

4. **設定 IAM Role 的 Trust Relationship，允許來自 WebIdentity(EKS 的 OIDC Provider) 可以執行 AssumeRole 的操作** (這個步驟很重要) 

5. 回到 k8s 中，新增 service account，並設定 annotation，指定 IAM Role ARN

如此一來，使用上面建立的 service account 的 pod，就會有能力存取 AWS resource 了，完全不需要 AWS access/secret key。

### IAM Role Trust Relationship 設定範例

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::777777777777:oidc-provider/oidc.eks.ap-northeast-1.amazonaws.com/id/${OIDC_WEB_IDENTITY_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/${OIDC_WEB_IDENTITY_PROVIDER}:aud": "sts.amazonaws.com",
          "oidc.eks.us-west-2.amazonaws.com/id/${OIDC_WEB_IDENTITY_PROVIDER}:sub": [
            "system:serviceaccount:userA:default-editor",
            "system:serviceaccount:userB:default-editor",
          ]
        }
      }
    }
  ]
}
```

### service account 設定範例

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-node
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::777777777777:role/tf_EKS-ServiceAccount_aws-node
```



總結
====

其實參考 [AWS 官網文件(Getting started with Amazon EKS – eksctl - Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)，使用 eksctl 建立 cluster 真的很快，但在快速建立 EKS cluster 的背後，其實往往忽略了很多重要的細節，特別是權限的部份。

透過上面一步一步做下來，搞清楚從無到有設定一個 EKS cluster，並搞清楚權限部份是如何運作的，當未來遇到權限相關問題時(這在 AWS 肯定常常會遇到)，也會有能力從比較正確的方向進行排查。



References
==========

- [Getting started with the AWS Management Console - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html)

- [Getting started with AWS Fargate using Amazon EKS - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/fargate-getting-started.html)

- [Enabling IAM roles for service accounts on your cluster - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)

- [Configuring the VPC CNI plugin to use IAM roles for service accounts - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html)

- [Amazon EKS with OIDC provider, IAM Roles for Kubernetes services accounts | by Marcin Cuber | Medium](https://marcincuber.medium.com/amazon-eks-with-oidc-provider-iam-roles-for-kubernetes-services-accounts-59015d15cb0c)

- [EKS 的一些筆記 - Kakashi's Blog](https://kkc.github.io/2018/10/04/EKS-notes/)
