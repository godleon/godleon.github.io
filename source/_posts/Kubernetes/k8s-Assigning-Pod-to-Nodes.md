---
layout: post
title:  "[Kubernetes] Assigning Pods to Nodes"
description: "本篇文章的主題在介紹在 Kubernetes 中，如何透過 nodeSelector, labelSelector, affinity ... 等機制，將 Pod 分派到特定的 worker node 上"
date: 2018-10-27 05:20:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - CKA
  - CKA Scheduling
---


前言
===

k8s 其中一個很大的優點在於很強大的 scheduler，在絕大多數時的情況下，使用者只要提交自己想要佈署的服務，k8s 就會自動的找到合適的 worker node 來運行，完全不用擔心 woker node 之間工作負載不平衡的狀況發生。

但有些時候，我們可能會希望可以人為介入 pod scheduling 的工作，例如：

- 希望讓 pod 運作在某些帶有某些特定硬體(例如：SSD)的 worker node 上

- 希望某些 pod 可以被固定放在一起

k8s 也提供了一些機制可以讓我們自己決定 job scheduling 是如何進行的，而**這些機制基本上都是建立在 label select 的基礎上完成的**，以下來逐一介紹。



nodeSelector
============

nodeSelector 是目前 k8s 提供的方式中最簡單的一個，只要在 pod spec 上指定所希望的 key/value pair 作為 nodeSelector，k8s 就會協助找到有相同 label 的 worker node 來接手工作。

但 nodeSelector 要怎麼用呢? 以下是簡單的流程說明：

## 為 Worker Node 設定 Label

上面提到可以指定 pod 到帶有特定 key/value pair 的 worker node 上，那當然 worker node 上一定要有特定的 key/value pair 才行，以下是為 worker node 指定 key/value pair 的方式

> kubectl label nodes/**\<node-name>** **\<label-key>**=**\<label-value>**

以下是實際示範過程：

```bash
# 檢視目前每個 node 所帶的 label
$ kubectl get nodes --show-labels
NAME              STATUS   ROLES    AGE   VERSION   LABELS
... (略)
leon-k8s-node03   Ready    node     12d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node03,node-role.kubernetes.io/node=true
leon-k8s-node04   Ready    node     12d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node04,node-role.kubernetes.io/node=true
leon-k8s-node05   Ready    node     12d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node05,node-role.kubernetes.io/node=true

# 新增一個 label(disk_type=ssd) 到 node05
$ kubectl label node/leon-k8s-node05 disk_type=ssd
node/leon-k8s-node05 labeled

# 可以看出 node05 已經有新的 label 出現了
$ kubectl get nodes --show-labels
NAME              STATUS   ROLES    AGE   VERSION   LABELS
... (略)
leon-k8s-node03   Ready    node     12d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node03,node-role.kubernetes.io/node=true
leon-k8s-node04   Ready    node     12d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node04,node-role.kubernetes.io/node=true
leon-k8s-node05   Ready    node     12d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk_type=ssd,kubernetes.io/hostname=leon-k8s-node05,node-role.kubernetes.io/node=true
```

> 從上面的 label 資訊可以看出，其實 k8s 剛安裝好後，就會為每個 node 自動加上一些預設的 label，這些 label 可能會在未來作為識別 or scheduling 時的一個依據，通常 public cloud provider 上佈署的 k8s，也會根據需求為每個 node 加上預設的 label 資訊

## 為 Pod spec 設定 nodeSelector

接著以下就是新增一個帶有 nodeSelector 的 pod，範例如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  # 設定的地方在 pod.nodeSelector
  nodeSelector:
    disk_type: ssd
```

套用以上設定後，檢視一下 pod 的內容：

```bash
# 檢視 pod 運作細節
$ kubectl describe pod/nginx
Name:               nginx
Namespace:          default
Priority:           0
PriorityClassName:  <none>
# 這裡可以看出 pod 有確實被分派到 node05 來執行
Node:               leon-k8s-node05/10.107.13.15
Start Time:         Wed, 24 Oct 2018 19:38:28 +0000
Labels:             env=test
Status:             Running
... (略)
# 從此欄位可以看出我們所指定的 nodeSelector 參數
Node-Selectors:  disk_type=ssd
... (略)
```



Built-in Node Labels
====================

剛安裝好 k8s 後，如果有曾經檢查過 node status 的話，可能會發現其實每個 node 都會有一些內建的 label set，以下是剛安裝好 v1.12.1 後的結果：

```bash
kubectl get nodes --show-labels
NAME              STATUS   ROLES    AGE   VERSION   LABELS
leon-k8s-node00   Ready    master   14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node00,node-role.kubernetes.io/master=true
leon-k8s-node01   Ready    master   14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node01,node-role.kubernetes.io/master=true
leon-k8s-node02   Ready    master   14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node02,node-role.kubernetes.io/master=true
leon-k8s-node03   Ready    node     14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node03,node-role.kubernetes.io/node=true
leon-k8s-node04   Ready    node     14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node04,node-role.kubernetes.io/node=true
leon-k8s-node05   Ready    node     14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk_type=ssd,kubernetes.io/hostname=leon-k8s-node05,node-role.kubernetes.io/node=true
```

可以看出內建的 label set 有以下幾項：

- **beta.kubernetes.io/arch**

- **beta.kubernetes.io/os**

- **kubernetes.io/hostname**

- **node-role.kubernetes.io/master**

- **node-role.kubernetes.io/node**

以上是在本地端環境安裝 k8s 後的結果，若是在 public cloud 上，為了管理方便，可能會有用來表示 Zone or AZ 這一類資訊 label 存在。

而以上這些內建的 node label，也會跟下面提到 affinity 機制在運行的時候會有相關，而這些 node label 則會被作為 `topologyKey`(下面會看到) 來使用。

> 為何這些 node label 要稱為 `topologyKey`? 因為從人類的角度來看，這些 node 的規劃透過以 topology 的型式來呈現，會比較容易理解


Node Affinity & anti-affinity
=============================

上述的 nodeSelector 提供了一個簡單的方式讓使用者可以將 pod 分派到帶有特定 label 的 worker node 上，但 nodeSelector 有時候並無法滿足現實世界的複雜需求，因此 **affinity/anti-affinity** 的功能就應運而生了。(目前在 v1.12 中還是屬於 beta feature)

而 affinity/anti-affinity 主要加強了幾個地方：

1. 可用更彈性的方法來指定多個 label 時的組合，而不再只能用 **AND**

2. 以往只能設定是否完全符合條件，現在可以用 `preference(希望可以有，但沒有也沒關係)` 的方式來設定

3. 可以指定跟帶有某些 label 的 `pod` 放在一起(or 不要放在一起)，而不是只能指定 worker node label：這樣有助於讓某些 pod 可以被放在同一個 worker node(或不被放在一起)

從上面幾個加強的地方可以看出，**affinity/anti-affinity** 這個 feature 共分為兩大類，分別是：

- node affinity

- inter-pod affinity/anti-affinity

nodeSelector 目前還是可以正常使用，但最終會被 node affinity 取代，因為 node affinity 可以完全取代 nodeSelector 的功能之外，還可以提供更多的其他功能，就像 RepplicationController & ReplicaSet 的關係是相同的。(ReplicaSet 取代了 RepplicationController)


## Node affinity/anti-affinity

node affinity 在 v1.2 之後所提供的功能，類似 nodeSelector ，有以下兩種類型：

- `requiredDuringSchedulingIgnoredDuringExecution`

- `preferredDuringSchedulingIgnoredDuringExecution`

上面的設定可以拆開三個部份來看，分別是：

- `requiredDuringScheduling`：一定要 node 符合條件，scheduler 才會把 pod 分派到上面去跑

- `preferredDuringScheduling`：會儘量嘗試找尋合適條件的 node，但不強制

- `IgnoredDuringExecution`：表示當 pod 已經正在運作中了，即使 node 的 label 在之後遭到變更，也不會影響正在運作中的 pod

從上面的說明就可以知道，兩種組合的搭配所得到的結果會是如何。

> 之後還會推出 `requiredDuringSchedulingRequiredDuringExecution` 的功能，而且同樣的，看字面的意思就可以很清楚了解這個設定會有什麼樣的效果

以下是一個 node affinity 的範例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      # 一定要滿足以下條件才可作 pod scheduling
      requiredDuringSchedulingIgnoredDuringExecution:
        # nodeSelector 的條件定義
        nodeSelectorTerms:
        - matchExpressions:
          # node 一定有帶有 以下任何一種 label 才可以
          # "kubernetes.io/e2e-az-name=e2e-az1"
          # "kubernetes.io/e2e-az-name=e2e-az2"
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      # 儘量滿足以下條件即可作 pod scheduling
      preferredDuringSchedulingIgnoredDuringExecution:
      # 這是屬於 prefer 的權重設定(1-100)，符合條件就會得到此權重值
      # pod 會被分配到最後加總數值最高的 node
      - weight: 1
        preference:
          matchExpressions:
          # 儘量尋找帶有 label "another-node-label-key=another-node-label-value" 的 node
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

除了上面範例中的 `In` 之外，k8s 還支援了其他的 operator 像是 `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` ... 等等，可以根據實際需求使用。

此外，node affinity 在使用上還有一些其他規則的存在，例如：

- 若同時設定 `nodeSelector` & `nodeAffinity`，那 scheduler 就會尋找同時滿足兩個條件要求的 node

- 如果在 `nodeAffinity` 中設定多個 `nodeSelectorTerms`，那就只要滿足任何一個 **nodeSelectorTerms** 的要求即可

- node affinity 目前只有在 pod scheduling 的時候會有用途，當 pod 已經在 node 上運行後，即使 node label 變更了也不會影響正在上面運行中的 pod，除非之後 `requiredDuringSchedulingRequiredDuringExecution` 的功能有推出

- 在 prefer 的設定中，有個 `weight` 的權重值(1-100)可以設定，而 scheduler 在決定前，還會加上其他 node priority function 來進行綜合考量，最後 pod 會被分配到數值計算結果最高的 node 上去


## Inter-pod affinity/anti-affinity

為什麼需要 pod affinity/anti-affinity?

> 這功能有時候挺有用的，特別是跟 ReplicaSets, StatefulSets 或是 Deployments 一起搭配的時候；例如：我希望把 workload 分派到特定的 topology 的運行(例如：同一個 node)

pod affinity 的功能是在 v1.4 的時候提出，其實這跟 node affinity 很類似，只是 scheduler 要尋找的是目前有哪些正在運行中的 **pod** 帶有符合條件的 label set，而不是 node。

規則的設定原則應該是`幫忙尋找符合設定條件的 Pod & 所在的 node 在哪裡，並把 Pod 放到該 node 上面運行`

而且 pod affinity 跟 node affinity 相同，也是有以下兩種設定類型可以使用：

- `requiredDuringSchedulingIgnoredDuringExecution`

- `preferredDuringSchedulingIgnoredDuringExecution`

而兩種設定的運作規則當然也跟 node affinity 相同，但通常 `preferredDuringSchedulingIgnoredDuringExecution` 常與 pod anti-affinity 搭配使用，而通常這兩者的搭配，大多是要達成**儘量將 pod 分散配置**的目的才會這樣使用。

當實際要設定 pod affinity/anti-affinity，有兩個條件可以用來進行設定：

- pod label set：這個部份其實很清楚，跟 node label set 其實是一樣的東西

- node topology key：這個部份通常就是 k8s 的 build-in node label (表示希望檢查 worker node 是否也符合條件)
> topologyKey 的設定用意在於，如果有多個 pod 需要分配，並指定了 topologyKey，那 scheduler 在分配時就**不可以**(注意! 是不可以!)將多個 pod 放到帶有相同 value 的topologyKey(Label) 的 node 上

以下用一個實際的例子來說明：

假設 k8s cluster 中有 3 個 worker node(node03, node04, node05)，且要在裡面安裝一個三份 replica 的 web application，而這個 web application 使用到 redis 作為 cache，為了平均的分散 work load，希望可以將 pod 佈署成：

- `node03`: redis + web-app

- `node04`: redis + web-app

- `node05`: redis + web-app

也就是每個 worker 都帶有一份 redis + web app，這要如何透過 pod affinity/anti-affinity 來完成呢? 可以參考以下步驟：

### 確認 topologyKey (node Label)

```bash
# 從上面可以看出，每個 worker node 中的 "kubernetes.io/hostname" 都帶有不同的值，可以拿來作為 topologyKey
$ kubectl get node --show-labels
NAME              STATUS   ROLES    AGE   VERSION   LABELS
... (ignore master nodes)
leon-k8s-node03   Ready    node     14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node03,node-role.kubernetes.io/node=true
leon-k8s-node04   Ready    node     14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=leon-k8s-node04,node-role.kubernetes.io/node=true
leon-k8s-node05   Ready    node     14d   v1.12.1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk_type=ssd,kubernetes.io/hostname=leon-k8s-node05,node-role.kubernetes.io/node=true
```

### 佈署 redis，並確保分散在不同的 node

接著要來佈署 redis 作為 cache service，但由於要將 redis 分散到不同的 node，因此進行了以下設定：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        # 確保 pod 不會落在帶有 app in "store" label 的 pod 所在的 worker node 上
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            # pod 在分配時要確保 worker node 帶有名稱為 "kubernetes.io/hostname" 的 label
            # 但不能有相同的 value
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

套用完以上設定後，可以看到 k8s 成功的將 redis cache 分散到不同的 worker node 上

```bash
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE              NOMINATED NODE
redis-cache-7d87c677b5-wg7rb   1/1     Running   0          11s   10.233.76.3      leon-k8s-node05   <none>
redis-cache-7d87c677b5-wn8s2   1/1     Running   0          11s   10.233.103.193   leon-k8s-node04   <none>
redis-cache-7d87c677b5-zjbwd   1/1     Running   0          11s   10.233.102.130   leon-k8s-node03   <none>
```

### 佈署 web app，並跟 redis 放一起

為了提升效能，佈署 web application 的時候，每個 web application 就要搭配一個 redis cache，可以透過以下設定完成：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        # 跟佈署 redis 相同，不要讓 web app 分派到同樣的 worker node 上
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        # 這是跟 redis cache 可以一對一放在一起並分散到不同 worker node 的關鍵
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          # 在這裡設定 redis cache 的 label set
          # 指定要找到 redis 所在的 worker node 來放 web app pod
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            # 但是要分散到不同的 worker node 上
            # (需要帶有 kubernetes.io/hostname label 的 worker node，但 value 不能相同)
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

套用完以上設定後，就可以看到 k8s 系統中已經將 web app 跟 redis 一對一對的分派到不同的 worker node 上：

```bash
$ kubectl get pod -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE              NOMINATED NODE
redis-cache-7d87c677b5-wg7rb   1/1     Running   0          18m   10.233.76.3      leon-k8s-node05   <none>
redis-cache-7d87c677b5-wn8s2   1/1     Running   0          18m   10.233.103.193   leon-k8s-node04   <none>
redis-cache-7d87c677b5-zjbwd   1/1     Running   0          18m   10.233.102.130   leon-k8s-node03   <none>
web-server-68844db94f-72mhj    1/1     Running   0          13s   10.233.76.4      leon-k8s-node05   <none>
web-server-68844db94f-ldp9s    1/1     Running   0          13s   10.233.102.131   leon-k8s-node03   <none>
web-server-68844db94f-m58p9    1/1     Running   0          13s   10.233.103.194   leon-k8s-node04   <none>
```

> 因此從上面的例子可以看到，若不要讓 pod 分配到同一個 node 上，只要設定 `podAntiAffinity` + `topologyKey: "kubernetes.io/hostname"` 即可 (因為 `kubernetes.io/hostname` 是 built-in 的 label，並且都會帶上不同的 value)


## 使用 affinity 的注意事項

官網的文件中有提到，由於 inter-pod affinity/anti-affinity 需要消耗大量的計算資源來作比對，因此若是 cluster node 數大於 700 的情況下，不建議使用這個功能，會大大降低整個 cluster 的運作速度。

不過是說一般人要用到超過 700 個 node 的 k8s cluster 似乎也是沒什麼機會倒是真的....



References
==========

- [Assigning Pods to Nodes - Kubernetes](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/)

- [Node affinity and NodeSelector - Kubernetes Design Doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/nodeaffinity.md)

- [Inter-pod topological affinity and anti-affinity - Kubernetes Design Doc](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md)