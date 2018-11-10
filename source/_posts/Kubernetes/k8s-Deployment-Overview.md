---
layout: post
title:  "[Kubernetes] Deployment Overview"
description: "本篇文章的主題在介紹 Kubernetes 中的 Deployment resource object 是如何運作 & 相關細節"
date: 2018-09-16 20:20:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Core Concept
  - CKA
  - CKA Core Concept
---


What is Deployment ?
====================

![Deployment](/blog/images/kubernetes/k8s-deployment.png)

Deployment 為 pod & replicaset 提供了一個宣告式的設定 & 更新方式，透過定義 desired status，Deployment controller 會在所謂的 **controlled rate** 下達到使用者所期望的狀態，這些機制是由 k8s 自動化完成，因此官方建議應該透過 Deployment 來佈署 pod & replicaset。



使用案例
=======

了解什麼是 Deployment 後，接下來的問題是.......什麼情況下可以使用 Deployment 來解決問題?

官方文件中列出了不少 use case，來回答此問題的答案：

- [Create a Deployment to rollout a ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment)

- [Declare the new state of the Pods](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment)

- [Rollback to an earlier Deployment revision](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment)

- [Scale up the Deployment to facilitate more load](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment)

- [Pause the Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pausing-and-resuming-a-deployment)

- [Use the status of the Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#deployment-status)

- [Clean up older ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#clean-up-policy)

以下針對上面每個使用案例使用一個範例來說明。


建立 Deployment
==============

以下是搭配 ReplicaSet 來產生 3 個 nginx pod：(假設檔名為 `nginx-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
# deployment metadata
metadata:
  # deployment name
  name: nginx-deployment
  labels:
    app: nginx
spec:
  # 同時建立 3 個 nginx 的 pod 
  replicas: 3
  # replicaset 的效果套用在帶有 app=nginx 的 pod 上
  # 必須要與下面的 pod label 有相符合
  selector:
    matchLabels:
      app: nginx
  # .spec.template 其實就是 pod 的定義
  template:
    # pod metadata
    metadata:
      # 設定給 pod 的 label 資訊
      labels:
        app: nginx
    spec:
      # 可看出這個 pod 只運行了一個 nginx container
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

使用以下指令佈署 deployment：

```bash
# 透過 "--record" 參數來保留後續的 revision history
$ kubectl apply -f nginx-deployment.yaml --record
```

接著透過 kubectl 來查詢剛剛佈署的 deployment 相關資訊：

```bash
# NAME： 列出了在目前 namespace 中的 deployment 清單
# DESIRED： 使用者所宣告的 desired status
# CURRENT： 表示目前有多少個 pod 副本在運行
# UP-TO-DATE： 表示目前有多個個 pod 副本已經達到 desired status
# AVAILABLE： 表示目前有多個 pod 副本已經可以開始提供服務
# AGE： 顯示目前 pod 運行的時間
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            0           15s
```

如果要即時監控 deployment 佈署的狀況，可以使用以下指令：

```bash
$ kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx-deployment" successfully rolled out
```

完成之後，AVAILABLE 的數字就會與 DESIRED 相同了：

```bash
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           32s
```

接著繼續往下看關於 ReplicaSet 的細節：

```bash
# ReplicaSet 的名稱會是 [DEPLOYMENT-NAME]-[POD-TEMPLATE-HASH-VALUE]
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-67594d6bf6   3         3         3         9m


# 檢視 ReplicaSet 的細節
$ kubectl describe rs/nginx-deployment-67594d6bf6
Name:           nginx-deployment-67594d6bf6
Namespace:      default
# deployment controller 主動為 replicaset 新增了一個 "pod-template-hash" label
Selector:       app=nginx,pod-template-hash=2315082692
Labels:         app=nginx
                pod-template-hash=2315082692
Annotations:    deployment.kubernetes.io/desired-replicas=3
                deployment.kubernetes.io/max-replicas=4
                deployment.kubernetes.io/revision=1
....(略)
Pod Template:
  Labels:  app=nginx
           pod-template-hash=2315082692
....(略)
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  18m   replicaset-controller  Created pod: nginx-deployment-67594d6bf6-22nrx
  Normal  SuccessfulCreate  18m   replicaset-controller  Created pod: nginx-deployment-67594d6bf6-rkztl
  Normal  SuccessfulCreate  18m   replicaset-controller  Created pod: nginx-deployment-67594d6bf6-ccx87
```

透過 Deployment 建立的 ReplicaSet，名稱都會是 **[DEPLOYMENT-NAME]-[POD-TEMPLATE-HASH-VALUE]**，因此透過檢視 ReplicaSet name，其實很容易就可以知道這是由那一個 Deployment 所建立出來的。

接著再往下看 Pod 的細節：

```bash
# deployment controller 同時也幫 pod 增加了一個 "pod-template-hash" label，hash value 也是相同的
$ kubectl get pod --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-67594d6bf6-22nrx   1/1       Running   0          12s       app=nginx,pod-template-hash=2315082692
nginx-deployment-67594d6bf6-ccx87   1/1       Running   0          12s       app=nginx,pod-template-hash=2315082692
nginx-deployment-67594d6bf6-rkztl   1/1       Running   0          12s       app=nginx,pod-template-hash=2315082692
```

跟 ReplicaSet 相同，Pod 的命名則會是 **[DEPLOYMENT-NAME]-[POD-TEMPLATE-HASH-VALUE]-[UNIQUE-ID]**，而且會發現被多加了一個名稱為 **pod-template-hash** 的 label


## Pod-Template-Hash Label

關於 **pod-template-hash** label，有幾點：

- 是由 deployment controller 加到每個 replicaset 的，主要是透過將 template 拿來計算 hash value 的方式來確保每個 replicaset 中的 template 定義不會有重複，

- pod-template-hash label 同時會被 deployment controller 加到 replicaset & pod 中，利用此 hash value 就可以確認那些 deploymen/replicaset/pod 是屬於同一組

- replicaset 名稱的 hash value 跟 pod-template-hash 不一樣，但應該是不同的 hash 方法所導致(因為透過相同的 YAML 檔案重新建立 deployment 也還是會得到相同的結果)


## 如果在另外一個 namespace 中建立相同的 Deployment ?

```bash
# 建立一個新的 namespace，名稱為 deployment-test
$ kubectl create namespace deployment-test
namespace/deployment-test created

# 將同樣的 deployment 宣告佈署到 deployment-test namespace 中
$ kubectl -n deployment-test apply -f nginx-deployment.yaml 
deployment.apps/nginx-deployment created

# 新的 deployment 佈署完成
$ kubectl -n deployment-test get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           16s

# 檢視 replicaset 的狀態
$ kubectl -n deployment-test get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-67594d6bf6   3         3         3         21s

# 檢視 pod 的狀態(包含 label 資訊)
$ kubectl -n deployment-test get pod --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-67594d6bf6-l2jg4   1/1       Running   0          29s       app=nginx,pod-template-hash=2315082692
nginx-deployment-67594d6bf6-phhkg   1/1       Running   0          29s       app=nginx,pod-template-hash=2315082692
nginx-deployment-67594d6bf6-vqptg   1/1       Running   0          29s       app=nginx,pod-template-hash=2315082692

# 完成實驗，移除 namespace
$ kubectl delete ns/deployment-test
```

從上面不難看出，即使在新的 namespace，同樣的 deployment 宣告依然會拿到同樣 value 的 pod-template-hash，原因就是因為 `.spec.template` 並沒有改變，因此透過 hash 產生出來的 value 當然也不會改變。



更新 Deployment
==============

剛使用者需要更新 deployment 內容時，就會考慮到這個問題，但更新的發生可能來自以下兩種變更：

1. 修改 replica 的數量 (`.spec.replicas`)

2. 修改 template 的內容 (`.spec.template`)


```bash
# 將 container image 的版本從上面的 nginx:1.7.9 改成 nginx:1.9.1
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record
deployment.extensions/nginx-deployment image updated

# 檢視 rollout status
$ kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out

# 新的變更已經成功套用
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           1d
```

> 也可以透過 `kubectl edit deployment/nginx-deployment` 指令直接對 YAML 檔案進行修改。


透過修改 replica 數量不會讓 **pod-template-hash** 有變動，但是修改 template 的內容就會造成 pod-template-hash 的變動了，由於上方已經變更了 container image version，因此來檢查一下目前 replicaset & pod 的狀態資訊：

```bash
# 檢視 replicaset 的狀態
# 可以發現 pod-template-hash 已經從 67594d6bf6 變為 6fdbb596db
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-67594d6bf6   0         0         0         1d
nginx-deployment-6fdbb596db   3         3         3         9m

# 檢視 replicaset 的細節
# pod-template-hash label 的值已經從 2315082692 變為 2986615286
$ kubectl describe rs/nginx-deployment-6fdbb596db
Name:           nginx-deployment-6fdbb596db
Namespace:      default
Selector:       app=nginx,pod-template-hash=2986615286
Labels:         app=nginx
                pod-template-hash=2986615286
Annotations:    deployment.kubernetes.io/desired-replicas=3
                deployment.kubernetes.io/max-replicas=4
                deployment.kubernetes.io/revision=2
Controlled By:  Deployment/nginx-deployment
Replicas:       3 current / 3 desired
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=nginx
           pod-template-hash=2986615286
  Containers:
   nginx:
    Image:        nginx:1.9.1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  9m    replicaset-controller  Created pod: nginx-deployment-6fdbb596db-67cwx
  Normal  SuccessfulCreate  8m    replicaset-controller  Created pod: nginx-deployment-6fdbb596db-r9qmv
  Normal  SuccessfulCreate  8m    replicaset-controller  Created pod: nginx-deployment-6fdbb596db-27dmh

# 檢視 pod 的狀態
# pod 的名稱也會跟著上面一起改變
$ kubectl get pod --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-6fdbb596db-27dmh   1/1       Running   0          8m        app=nginx,pod-template-hash=2986615286
nginx-deployment-6fdbb596db-67cwx   1/1       Running   0          9m        app=nginx,pod-template-hash=2986615286
nginx-deployment-6fdbb596db-r9qmv   1/1       Running   0          9m        app=nginx,pod-template-hash=2986615286
```

從上面的結果可以看出，當 Deployment 中的 `.spec.template` 改變後，replicaset & pod 中的 **pod-template-hash** 也會一併跟著改變。

此外，關於 update deployment 這件事情，k8s 在運作機制上會遵守以下原則：

- 至少會確保有 25% 以上的期望 pod 數量是維持可服務狀態的

- 最多不會讓超過 25% 的 pod 數量無法使用 (在 3 replicas 的條件下，create 會先執行，然後才是 delete)

檢視一下 deployment 的細節就可以看出詳細的運作流程：

```bash
$ kubectl describe deployment/nginx-deployment
Name:                   nginx-deployment
.... (略)
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.9.1
    Port:         80/TCP
.... (略)
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-6fdbb596db (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  # available pod = 4(new 1, old 3)
  Normal  ScalingReplicaSet  20m   deployment-controller  Scaled up replica set nginx-deployment-6fdbb596db to 1
  # available pod = 3(new 1, old 2)
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 2
  # available pod = 4(new 2, old 2)
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled up replica set nginx-deployment-6fdbb596db to 2
  # available pod = 3(new 2, old 1)
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 1
  # available pod = 3(new 3, old 1)
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled up replica set nginx-deployment-6fdbb596db to 3
  # available pod = 3(new 3, old 0)
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 0
```

以上狀況是在原本的 deployment 已經完成佈署的狀態下，變更 template 後會發生的行為，那如下發生以下狀況：

> 佈署 5 個副本 + nginx:1.7.9 的 deployment，但當只有 3 個 pod 佈署完成時，馬上將 template 中的 container image version 修改為 nginx:1.9.1

以上面的例子來說，k8s 就不會遵守原本的更新規則，而是會直接砍掉原本的 3 個 pod，然後開始建立新版本的 5 個 pod。

最後，有個需要注意的是 Label Selector 的部份，在 API version **apps/v1** 之後，`.spec.selector` 被設定之後就無法再修改了!



Deployment 版本回溯
=================

在預設情況下，系統中會包含 Deployment rollout 的歷史資訊(也就是 `.spec.template` 有變動時)，因此使用者可以在 在 revision history limit 的範圍內，回復到任何一個時間點的狀態。

> 由於只有當 `.spec.template` 變更時才會有 revision 記錄，因此改變 replica 數量就不會產生新的 revision 記錄

```bash
# 將 container image 從 nginx:1.9.1 更新為 nginx:1.91
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment.extensions/nginx-deployment image updated

# 檢視 rollout status
# 因為 container image version error，所以無法正確完成工作，因此更新會卡住，按下 Ctrl+C 跳離
$ kubectl rollout status deployments nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
^C

# 取得 replicaset 狀態
# 新建的 replicaset 一直在無法完成工作的狀態
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-58c7645486   1         1         0         4m
nginx-deployment-67594d6bf6   0         0         0         2d
nginx-deployment-6fdbb596db   3         3         3         23h

# pod 則是明確的顯示遇到 image pull 的問題，時間一久就會開始進入 back off 的狀態
$ kubectl get pod
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-58c7645486-xrgn8   0/1       ImagePullBackOff   0          4m
nginx-deployment-6fdbb596db-27dmh   1/1       Running            0          23h
nginx-deployment-6fdbb596db-67cwx   1/1       Running            0          23h
nginx-deployment-6fdbb596db-r9qmv   1/1       Running            0          23h

# 檢視 deployment 的細節
$ kubectl describe deployment/nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Mon, 10 Sep 2018 20:13:20 +0000
Labels:                 app=nginx
# 從 annotation 可以看出目前 deployment status 是由什麼指令造成的
Annotations:            deployment.kubernetes.io/revision=3
                        kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"kubernetes.io/change-cause":"kubectl apply --filename=nginx-deployment.yaml --r...
                        kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.91 --record=true
Selector:               app=nginx
Replicas:               3 desired | 1 updated | 4 total | 3 available | 1 unavailable
....(略)
OldReplicaSets:  nginx-deployment-6fdbb596db (3/3 replicas created)
NewReplicaSet:   nginx-deployment-58c7645486 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled up replica set nginx-deployment-67594d6bf6 to 3
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled up replica set nginx-deployment-6fdbb596db to 1
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 2
  Normal  ScalingReplicaSet  1m    deployment-controller  Scaled up replica set nginx-deployment-6fdbb596db to 2
  Normal  ScalingReplicaSet  59s   deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 1
  Normal  ScalingReplicaSet  59s   deployment-controller  Scaled up replica set nginx-deployment-6fdbb596db to 3
  Normal  ScalingReplicaSet  48s   deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 0
  Normal  ScalingReplicaSet  16s   deployment-controller  Scaled up replica set nginx-deployment-58c7645486 to 1
```

接著來看一下 rollout history：

```bash
# 透過 rollout history 指令可以看出曾經下過什麼指令
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=nginx-deployment.yaml --record=true
2         kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record=true
3         kubectl set image deployment/nginx-deployment nginx=nginx:1.91 --record=true

# 檢視 rollout hsitory 的細節
$ kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" with revision #2
Pod Template:
  Labels:	app=nginx
	pod-template-hash=2986615286
  Annotations:	kubernetes.io/change-cause=kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record=true
  Containers:
   nginx:
    Image:	nginx:1.9.1
....(略)
```

> 請注意! 在目前的版本 1.11.2 中，如果指令沒有加上 `--record` 參數，就不會在這邊看到指令的細節(其實這些指令是存放在 deployment 的 annotation 中)


這時候我們知道 revision 2 這個版本是正常的，因此我們可以透過以下指令將 deployment 進行 rollback：

```bash
# 指令效果相同於 "kubectl rollout undo deployment/nginx-deployment --to-revision=2"
$ kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment

# 重新檢視 deployment status => 正常
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           19m

# 重新檢視 replicaset status => 正常
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-58c7645486   0         0         0         17m
nginx-deployment-67594d6bf6   0         0         0         19m
nginx-deployment-6fdbb596db   3         3         3         18m

# 重新檢視 pod status => 正常
$ kubectl get pod
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-6fdbb596db-7zqm6   1/1       Running   0          18m
nginx-deployment-6fdbb596db-kcl8k   1/1       Running   0          18m
nginx-deployment-6fdbb596db-rljdk   1/1       Running   0          18m
```



Scale out/in Deployment
=======================

## Scale out

在 k8s 中設定 scale out 相當簡單，以下語法就可以完成：

```bash
# 設定目前的 pod repplica 為 10
$ kubectl scale deployment/nginx-deployment --replicas=10
deployment.extensions/nginx-deployment scaled

# 10 個 pod 順利產生
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   10        10        10           10          23h
```


## 設定 Horizontal Pod Autoscaling

如果希望 pod replica 可以根據某個 resource 指標進行自動伸縮，則可以使用 [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/) 的功能：

```
$ kubectl autoscale deployment nginx-deployment --min=10 --max=20 --cpu-percent=80
horizontalpodautoscaler.autoscaling/nginx-deployment autoscaled

$ kubectl get hpa
NAME               REFERENCE                     TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx-deployment   Deployment/nginx-deployment   <unknown>/80%   10        20        0          5s
```


## Proportional Scaling

當變更 deployment template 時，其實 k8s 是允許多個版本同時存在的(會嚴格遵守 [maxSurge](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-surge) & [maxUnavailable] 的設定(https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-unavailable))

開始測試之前，先來檢視一下 deployment 的細節：

```bash
$ kubectl describe deployment/nginx-deployment
Name:                   nginx-deployment
.... (略)
RollingUpdateStrategy:  25% max unavailable, 25% max surge
```

可以看到關於 Proportional Scaling 的兩個重要設定，分別是：

- **25% max unavailable**

- **25% max surge**

接著以上面 rollback 的例子來說，如果當 replica 的數量比較大時，就比較容易看的出來，以下用個範例來說明：

```bash
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   10        10        10           10          23h

# 將 container image version 指定到一個不存在的版本
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91

# k8s 調整後 (目前因為 image version 問題，所以已經卡住)
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   10        13        5            8           23h

# 兩個不同的 version 同時存在(template hash code 為 "67594d6bf6" & "58c7645486")
# 原本的 replicaset 中有 2(10-8) 個 pod 已經暫時被移除 (maxUnavailable < 25%)
# 所有的 replicaset 的 pod 數量為 13(8+5)，沒有超過 max surge 定義的 25% => (13 - 10) / 13 約莫 24%
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-58c7645486   5         5         0         23h
nginx-deployment-67594d6bf6   8         8         8         23h
nginx-deployment-6fdbb596db   0         0         0         23h

# 可以再度看一下 deployment 細節
# 可以發現無論何時，所有 pod 的總和都沒超過 13
# 也就是表示 max surge 低於 25%
$ kubectl describe deployment/nginx-deployment
Name:                   nginx-deployment
....(略)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  9m    deployment-controller  Scaled up replica set nginx-deployment-67594d6bf6 to 10
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-58c7645486 to 3
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled down replica set nginx-deployment-67594d6bf6 to 8
  Normal  ScalingReplicaSet  2m    deployment-controller  Scaled up replica set nginx-deployment-58c7645486 to 5
```

> 這邊比較需要注意的是，`max surge` 的計算中，是以 **新舊 replica 中所有的 pod 數量為分母** 去計算出來的，而不是以原有設定的 10 replica。



暫停 & 恢復 Deployment
====================

k8s 允許使用者暫停某個 deployment 後續的任何變更，然後再恢復。這種功能有利於當有多個調整要套用到 deployment 上時，而不需要不斷的觸發 rollout。

首先來看看目前 k8s cluster 內部 deployment & replicaset 的狀況：

```bash
$ kubectl get deployment
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           12s

$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-67594d6bf6   3         3         3         15s
```

確認了目前　deployment　的狀況後，接著透過 `rollout pause` 來將 deployment 的變更暫停：

```bash
# 暫停 deployment/nginx-deployment 的 rollout (template 變更暫時不會觸發 rollout)
$ kubectl rollout pause deployment/nginx-deployment
deployment.extensions/nginx-deployment paused

# 變更 deployment/nginx-deployment 中的 container image 設定
# 並將指令紀錄到 rollout history => --record
$ kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record
deployment.extensions/nginx-deployment image updated

# 檢視 deployment/nginx-deployment 的 rollout history
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION  CHANGE-CAUSE
1         <none>

# 變更 deployment/nginx-deployment 中的 resource quita 設定
# 這個變更不會馬上觸發 rollout，因為已經被暫停
$ kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
deployment.extensions/nginx-deployment resource requirements updated
```

從暫停到目前為止，已經完成了上面的兩個變更，分別是：

1. 修改 container image version

2. 設定 resource quota

接著把 rollout 功能恢復：

```bash
# 恢復 deployment/nginx-deployment 的 rollout
$ kubectl rollout resume deployment/nginx-deployment
deployment.extensions/nginx-deployment resumed

# 檢視目前 replicaset rollout 的即時狀態
# 仔細看下方的輸出，就可以發現 rollout 的過程都是會遵守 maxUnavailable & maxSurge 的設定
$ kubectl get rs -w
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-67594d6bf6   3         3         3         2m
nginx-deployment-84bc848fd6   1         1         0         8s
nginx-deployment-84bc848fd6   1         1         1         11s
nginx-deployment-67594d6bf6   2         3         3         3m
nginx-deployment-84bc848fd6   2         1         1         11s
nginx-deployment-67594d6bf6   2         3         3         3m
nginx-deployment-84bc848fd6   2         1         1         11s
nginx-deployment-84bc848fd6   2         2         1         11s
nginx-deployment-67594d6bf6   2         2         2         3m
nginx-deployment-84bc848fd6   2         2         2         22s
nginx-deployment-67594d6bf6   1         2         2         3m
nginx-deployment-84bc848fd6   3         2         2         22s
nginx-deployment-67594d6bf6   1         2         2         3m
nginx-deployment-84bc848fd6   3         2         2         22s
nginx-deployment-84bc848fd6   3         3         2         22s
nginx-deployment-67594d6bf6   1         1         1         3m
nginx-deployment-84bc848fd6   3         3         3         33s
nginx-deployment-67594d6bf6   0         1         1         3m
nginx-deployment-67594d6bf6   0         1         1         3m
nginx-deployment-67594d6bf6   0         0         0         3m
^C

# 最後檢視一下完成狀態
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-67594d6bf6   0         0         0         3m
nginx-deployment-84bc848fd6   3         3         3         52s
```

最後記得一件事情，若是 rollout 目前已經暫停，那就連 rollback 也無法做了，因為 rollback 也會觸發 rollout 的發生。



佈署狀態
======

一個 deployment 整體生命周期中會有好幾個狀態，例如：

- **Progressing**

- **Complete**

- **Fail to progress**

以下就針對上面3個狀態進行額外說明。

## Progressing

當 deployment 在進行以下工作時，狀態會變更為 **Progressing**：

- 建立一個 replicaset

- scale up 新的 replicaset

- scale down 舊的 replicaset

- 新的 pod 狀態變成 **ready** or **available**


## Complete

當以下的條件 or 狀態有達成時，k8s 會將 deployment 狀態變更為 **Complete**：

- 所有與 deployment 相關的 replicaset 都已經更新到最新的 template 版本(表示已經達到 desired status)

- 所有相關的 pod 的狀態都是 available

- 沒有舊版本的 pod 還在執行中

透過以下指令可以簡單的檢視目前 deployment 是否已經進入 complete 的狀態：

```bash
# 檢視 deployment rollout status
$ kubectl rollout status deployment/nginx-deployment
deployment "nginx-deployment" successfully rolled out

# 若上一個指令的 exit code 為 0 表示 deployment 目前為 complete 狀態
root@leon-k8s-node01:~# echo $?
0
```


## Fail to progress

有時會遇到 deployment 執行到一半發生問題 or 卡住而無法完成工作，而這些問題通常是以下情況引起的：

- Readiness probe 診斷失敗

- container image pull 發生錯誤

- 權限不足

- resource quota 不足 (namespace level)

- 超過 limit range 設定範圍 (pod level)

- application 本身造成的錯誤

其實 k8s 判定 deployment 為 fail to progress 的預設時間為 600 秒，如果想要提早確認 deployment 無法正常運作，可以透過設定 **[.spec.progressDeadlineSeconds](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#progress-deadline-seconds)** 來縮短 or 延長判定為 fail 的時間，可以直接定義在 deployment spec 中，也可以透過以下指令修改：

```bash
# progressDeadlineSeconds 預設為 600 秒，因此不會變更
$ kubectl patch deployments/nginx-deployment -p '{ "spec": { "progressDeadlineSeconds": 600 }}'
deployment.extensions/nginx-deployment not patched

#　將 progressDeadlineSeconds 從 600 秒改為 300 秒
$ kubectl patch deployments/nginx-deployment -p '{ "spec": { "progressDeadlineSeconds": 300 }}'
deployment.extensions/nginx-deployment patched
```

當超過　progressDeadlineSeconds 設定值，就會有一筆新的 **DeploymentCondition** 資訊被加入到 deployment 的 `.status.conditions` 中，內容如下：

| Type | Status | Reason |
|------|--------|--------|
| Progressing | False | ProgressDeadlineExceeded | 

當 deployment 已經被判定為 fail to prgress 後，除了回報 deployment 狀態外，k8s 就不會再額外多做什麼了。但若是有更上層的工具，依然還是可以利用這樣的狀態完成一些管理操作，例如：rollback 到前一個版本。

最後，即使是一個 failed deployment，當它的 template 變更時，依然還是會觸發 roll out 的進行，就跟一般正常的 deployment 是完全相同的。


## 當 Deployment 發生 Failure 時怎辦？

從上面可以看出，可能發生 fail to progress 的原因其實也不少，當遇到 deployment 無法正常佈署時，可以透過以下幾種方式來查 root cause：

### 檢查 rollout status

以上面 **deployment/nginx-deployment** 為例，利用以下語法可以檢查 rollout status：

```bash
# 檢視 deploy/nginx-deployment 的 rollout status
$ kubectl rollout status deploy/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
error: deployment "nginx" exceeded its progress deadline

# 上一個指令回傳的 exit code 不等於 0
$ echo $?
1
```

### 輸出完整的 YAML 資訊

透過 `kubectl describe` 檢視 deployment 狀況時，可能會出現以下資訊：

```bash
$ kubectl describe deployment nginx-deployment
.....(略)
Conditions:
  Type            Status  Reason
  ----            ------  ------
  Available       True    MinimumReplicasAvailable
  Progressing     True    ReplicaSetUpdated
  ReplicaFailure  True    FailedCreate
.....(略)
```

只看到 **FailedCreate** 實在很難判定實際的問題到底是什麼，因此可以透過以下指令找出導致問題的詳細原因：

```bash
# 可看出每一個階段產生出的詳細 log 資訊
$ kubectl get deployment nginx-deployment -o yaml
.....(略)
status:
  availableReplicas: 2
  conditions:
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: Replica set "nginx-deployment-4262182780" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  - lastTransitionTime: 2016-10-04T12:25:42Z
    lastUpdateTime: 2016-10-04T12:25:42Z
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: 2016-10-04T12:25:39Z
    lastUpdateTime: 2016-10-04T12:25:39Z
    message: 'Error creating: pods "nginx-deployment-4262182780-" is forbidden: exceeded quota:
      object-counts, requested: pods=1, used: pods=3, limited: pods=2'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
  observedGeneration: 3
  replicas: 2
  unavailableReplicas: 2
.....(略)
```


Clean up Policy
===============

這個部份主要設定的是 revision history 保留的數量，這個設定可以針對單一的 Deployment 進行設定，透過設定 `.spec.revisionHistoryLimit` 就可以指定 k8s 要保留幾份 Deployment 的 revision history，預設值為 **10**。

> 若是設定為 0，將會喪失 rollback 的能力



如何撰寫正確的 Deployment
======================

以上介紹了很多 Deployment 的特性 & 功能，但如果希望某些屬性在定義時就宣告完成，就必須要知道 deployment spec 要如何撰寫，以下是一個比較詳細的範例：

```yaml
# apiVersion, kind, metadata 3個欄位是必備的
apiVersion: apps/v1
kind: Deployment
metadata:
  # deployment name
  name: nginx-deployment-spec
  labels:
    app: nginx
spec:
  strategy:
    # 用來指定當 new pod 要取代 old pod 時要如何進行
    # RollingUpdate: 會根據 maxUnavailable & maxSurge 的設定，確保有足夠的 pod 可以提供服務
    # 才會慢慢的將 old pod 換成 new pod (default)
    # Recreate: 會先移除所有的 old pod 後，才會開始產生 new pod
    type: RollingUpdate
    rollingUpdate:
      # 指定當 update 進行中時，可以失效(無法提供服務)的 pod 佔整體 pod 數量的比例(也可以是整數值)為多少
      # (default = 25%)
      maxUnavailable: 25%
      # 指定當 update 進行中時，pod 可以超過 desired status 定義數量的比例(也可以是整數值)
      # (default = 25%)
      maxSurge: 25%
  # 指定要建立多少 pod 副本(default = 1)
  # 實際情況少於此數字，則會增加 pod，反之則會殺掉 pod
  replicas: 3
  # 指定最長等待多少時間後，佈署依舊無法順利完成時，回報 "failed progressing" 的時間(秒)
  # (default = 600)
  progressDeadlineSeconds: 600
  # 設定若是沒有任何 pod crashing 的情況發生，被認為是可用狀態的最小秒數 (default = 0)
  minReadySeconds: 0
  # 設定 revision history 保留的數量
  # 建議可以根據 Deployment 更新的頻率來決定這個值要設定多少
  # 設定 0 將會無法進行 rollback
  # (default = 10)
  revisionHistoryLimit: 10
  # 用來指定要用來監控並進行管理的 pod label 設定
  # 必須要與下面的 pod label(.spec.template.metadata.labels) 相符合
  # 在 apps/v1 版本中，".spec.selector" 一旦設定後就無法變更了
  selector:
    matchLabels:
      app: nginx
  # .spec.template 其實就是 pod 的定義
  # 也是 .spec 中唯一必須要設定的欄位
  template:
    # pod metadata
    metadata:
      # 設定給 pod 的 label 資訊
      labels:
        app: nginx
    spec:
      # restart policy  在 Delpoyment 中只能設定為 Always
      restartPolicy: Always
      # 可看出這個 pod 只運行了一個 nginx container
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

> 設定 deployment 時，必須注意**不可以**佈署帶有相同 label selector 的 Deployment, ReplicaSet 或是 ReplicationController



References
==========

- [Deployments - Kubernetes](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

- [Deployment · Kubernetes Handbook - Kubernetes中文指南/云原生应用架构实践手册 by Jimmy Song(宋净超)](https://jimmysong.io/kubernetes-handbook/concepts/deployment.html)
