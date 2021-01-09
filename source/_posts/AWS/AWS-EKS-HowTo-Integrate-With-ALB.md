---
layout: post
title:  "[AWS EKS] 從零打造一個可運行 Fargate workload 的 EKS cluster"
description: "此篇文章是介紹如何透過管理 Kubernetes Ingress resource 與 AWS ALB 直接進行整合，並解釋要完成這件事情的過程中所需要完成的相關權限設定"
date: 2021-01-10 04:50:00
published: true
comments: true
categories:
  - AWS
tags:
  - AWS
  - EKS
  - ELB
  - IAM
---


Overview
========

在這篇文章中，要介紹如何在建立 k8s ingress resource 時，可以串接到 AWS，並建立對應的 ALB 資源。



權限設計說明
==========

## 存取權限在 k8s & AWS 是如何管理的?

![AWS/Kubernetes Access Control](/blog/images/aws/EKS/EKS_aws-access-control.png)

要了解如何在 k8s 存取 AWS resource 權限前，必須先弄清楚"存取權限"在 k8s & AWS 中是如何被管理的：

- 在 k8s 中有自己的 RBAC 機制，透過 ServiceAccount/ClusterRole/ClusterRoleBinding，可以指定某個 service account 在 k8s 中有哪些 API 的存取權限

- AWS 的部份，主要則是透過 IAM Role + IAM Policy 的組合，來決定套用那個 IAM Role 時可以有哪些 API 的呼叫權限

## 如何賦予 k8s 存取 AWS Resource 的權限?

![Kubernetes service account annotation](/blog/images/aws/EKS/EKS_k8s-service-account-annotation.png)

有了上面的概念後，那要存取 AWS Resource 到底要怎麼做呢? 重點大概分為以下幾個部份：

1. 設定 IAM Policy，指定有建立 & 管理 ALB 的相關權限 (可參考 [kubernetes-sigs
/
aws-load-balancer-controller](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy.json) 專案來取得最新資訊，目前是版本為 `v2.1.0`)

2. 設定 IAM Role，並與上一個步驟的 IAM Policy 繫結

3. 設定 k8s service account 的 annotation，指定要使用上一個步驟建立的 IAM Role(要使用 Role ARN)；如此一來，有設定 annotation 的 k8s service account 就會擁有 IAM Role 的權限來存取 AWS resource 了


## AWS 如何認證來自 k8s 的請求?

到這裡就差一個問題，設定了 k8s service account annotation 之後，要怎麼讓 k8s service account 可以 assume 成指定的 IAM Role 呢?

這個就需要透過 **Assume Role With WebIdentity** 的方式，而這方式就會依賴 EKS cluster 提供的 `OpenID Connect provider` 來達成，而 Assume Role 的設定則是在 IAM Role 中的 `Trust Entity` 中設定，以下是個簡單的範例：

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

如此一來 IAM 可以辨識存取 AWS resource 的請求是來自於哪個 k8s cluster，並決定有沒有存取 AWS API 的權限。



行前準備
=======

要開始進行 EKS cluster & AWS ALB 的互動之前，要先完成兩件事情：

1. 已經有一個 EKS cluster，並做好相關的權限設定 (可參考[前一篇文章](AWS-EKS-create-cluster-for-pure-fargate-workload.md)進行設定)

2. Kuberntes 必須知道哪個 subnet 是 public，哪個是 private，這樣才可以正確的佈署好 Internet-facing & Internal ALB，因此要做以下設定：
    - 在 public subnet 中新增 tag `kubernetes.io/role/elb = 1`
    - 在 private subnet 中新增 tag `kubernetes.io/role/internal-elb = 1`
    - 在所有 EKS 使用的 subnet 中新增 tag `kubernetes.io/cluster/eks-test  = shared` (假設 EKS cluster 名稱為 `eks-test`)

AWS Load Balancer Controller 會持續監聽 k8s 上 Ingress resource 的變化；若是要明確指定特定的 k8s Ingress resource 要使用 AWS ALB 建立，那就要在 Ingress resource 的 YAML 宣告中加上 annotation 設定 `kubernetes.io/ingress.class: alb`。



ALB Controller Traffic Mode
===========================

而目前 AWS Load Balancer Controller 支援兩種 traffic mode，分別是：

- `Instance`：(預設)註冊到 ALB 的單位是 worker node，而 ALB 會將流量導入 worker node 中，k8s service 所開放的 NodePort；若要明確指定，則是可以明確的指定 annotation `alb.ingress.kubernetes.io/target-type: instance` (**目前 Fargate 不支援 instance mode，因為 Fargate pod 不支援 NodePort service**)
> 這是透過設定 service type 為 NortPort，並將流量導向 NodePort 去 

- `IP`：註冊的單位是 Pod，ALB 會將 traffic 直接導入 Pod 中，必須明確指定 annotation `alb.ingress.kubernetes.io/target-type: ip`，**若是使用 fargate，則只有 IP traffic mode 可以選**

此外，若要為 ALB 加上 tag 資訊，則是透過設定 annotation `alb.ingress.kubernetes.io/tags` 來完成。



佈署 ALB Controller
===================

接著要佈署 ALB Controller 到 EKS Cluster 中，以下透過 eksctl 進行權限設定的部份：

## 設定 IAM Role for AWS ALB Controller

```bash
$ mkdir -p /tmp/eks-test

$ cd /tmp/eks-test

# 下載 AWS Load Balancer Controller 所會使用到的 IAM Policy 清單
$ curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy.json

# 建立 IAM Policy
$ aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json


# 若是原本在 eksctl 管理下存在 IAM service account，那就先移除
# 因為第二次重裝的 EKS cluster 不會有該 service account 的資訊
$ eksctl delete iamserviceaccount --name aws-load-balancer-controller --namespace kube-system --cluster eks-test

# 新增一個 IAM Role，並加入指定的 IAM Policy
# 並指定 service account "aws-load-balancer-controller" 使用此 IAM Role
$ eksctl create iamserviceaccount \
  --cluster=eks-test \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::777777777777:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
[ℹ]  eksctl version 0.34.0
[ℹ]  using region ap-northeast-1
[ℹ]  1 existing iamserviceaccount(s) (kube-system/aws-node) will be excluded
[ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
[ℹ]  1 iamserviceaccount (kube-system/aws-node) was excluded (based on the include/exclude rules)
[!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
[ℹ]  1 task: { 2 sequential sub-tasks: { create IAM role for serviceaccount "kube-system/aws-load-balancer-controller", create serviceaccount "kube-system/aws-load-balancer-controller" } }
[ℹ]  building iamserviceaccount stack "eksctl-eks-test-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
[ℹ]  deploying stack "eksctl-eks-test-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
[ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"

# 檢視 Service Account 的內容
# 可看到 aws-load-balancer-controller service account 所對應到的 IAM Role 是那一個
# 進 AWS Console 檢查一下，可以看到此 IAM Role 繫結的就是上面建立的 IAM Policy
$ kubectl -n kube-system describe sa/aws-load-balancer-controller
Name:                aws-load-balancer-controller
Namespace:           kube-system
Labels:              <none>
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::777777777777:role/eksctl-eks-test-addon-iamserviceaccount-kube-Role1-1MNNDI4RVIZFF
Image pull secrets:  <none>
Mountable secrets:   aws-load-balancer-controller-token-z2xg9
Tokens:              aws-load-balancer-controller-token-z2xg9
Events:              <none>
```

其實上面有提到權限設定的說明部份，因此即使不透過 eksctl，要自行完成設定 IAM service account 也不是太困難的事情，大概就是以下流程：

1. 使用 [AWS 提供的 IAM Policy 檔案](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy.json)，新增 IAM Policy

2. 新增 IAM Role，與上個步驟的 IAM Policy 繫結

3. 設定 IAM Role 的 Trust Relationship，完成 `sts:AssumeRoleWithWebIdentity` 的設定，來源為 EKS cluster 的 OIDC Connect Provider

4. 在 EKS 中新增 service account，並透過 annotation 指定 IAM Role ARN

5. 佈署 AWS Load Balancer Controller 時指定使用上面建立的 IAM Role


## 透過 Helm 安裝 AWS Load Balancer Controller

```bash
$ kubectl apply -f https://raw.githubusercontent.com/aws/eks-charts/master/stable/aws-load-balancer-controller/crds/crds.yaml

# 要先安裝 Helm
$ helm repo add eks https://aws.github.io/eks-charts

$ helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=[EKS_CLUSTER_ID] \
  --set region=[REGION)_ID] \
  --set vpcId=[VPC_ID] \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  -n kube-system
```



佈署服務進行測試
==============

- AWS LoadBalancer Controller 產生出來的 ingress 有能力自動發現指定 domain 對應的 ACM 並直接套用


```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: deployment-2048
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  replicas: 2
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
      - image: alexwhen/docker-2048
        imagePullPolicy: Always
        name: app-2048
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: default
  name: service-2048
  
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048

---
# 搭配上一篇文章 EKS fargate 的建置
# 配合 "alb.ingress.kubernetes.io/target-type" 設定為 ip
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: default
  name: ingress-2048
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}, {"HTTPS":443}]'
spec:
  tls:
  - hosts:
    - eks-test.mydomain.com
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: service-2048
              servicePort: 80
```



References
==========

- [Application load balancing on Amazon EKS - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)

- [Network load balancing on Amazon EKS - Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html)

- [AWS LoadBalancer Controller 官網](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/)