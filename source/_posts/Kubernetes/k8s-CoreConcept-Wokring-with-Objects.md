---
layout: post
title:  "[Kubernetes] 如何管理 Resource Object"
description: "此篇文章介紹在 Kubernetes 中，提供了那些機制來管理各式各樣的 Resource Object"
date: 2018-08-29 09:45:00
published: true
comments: true
categories: [Kubernetes]
tags: [Kubernetes]
---

此篇文章介紹在 Kubernetes 中，提供了那些機制來管理各式各樣的 Resource Object


What are Kubernetes Objects?
=============================

k8s object 是存在於 k8s 系統中的 persisten entity，用來描述 k8s cluster 中的狀態，例如：

- 那一個 containerized application 正在運作?

- 承上，在哪些 worker nodes 上運行?

- 那些資源是可提供給 application 使用

- 套用在這些 application 上的規則，例如：restart policy, upgrade, fault-tolerance


每個 Object 都包含了 **spec** & **status** 兩個部份，其中 spec 是用來描述 object 要以什麼樣的形式(規格 or 內容)呈現，而 status 則是用來描述這個 object 希望由 K8s 所維持的狀態(**desired state**)。

以下舉個 deployment 的例子來說明：(可透過 YAML or JSON 呈現)

```yaml
# 指定使用那個版本的 API 來建立此 object
apiVersion: apps/v1beta1
# 建立什麼樣的 resource object
kind: Deployment
# 用來辨識此物件的資訊，可能包含"name", "UID", "namespace" .... 等資訊
metadata:
  name: nginx-deployment
# spec 則是用來描述物件被生成的細節
# 每個不同的 resource object 都有不同的 spec 定義
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

> 若是透過 [kubectl create -f FILENAME](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create) 建立物件，檔案的資料格式可以是 JSON or YAML 格式；但若是呼叫 API 直接建立物件，就只能使用 JSON 格式資料

上面是一個 deployment object 的定義，其中的 spec 部份則清楚的定義這個 deployment 的規格(使用 nginx:1.7.9 作為 image 來運行 container)；而 status 則是希望有 3 份 replica，因此 K8s 會協助維持 3 份 replica 的狀態。

在前一篇文章提到，在 K8s 上各式各樣(**kind**)的 resource，其實就是以一個一個 object 存在於 K8s ecosystem 中，而每個 object 都各自擁有自己的 spec & status。

以下提供兩種 resource object 的 spec 定義文件提供參考：

- [Deployment](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#deploymentspec-v1-apps)

- [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#podspec-v1-core)



Name
====

每個在 K8s cluster 上的 object 都可以用 Name or UID 來識別，而在同一時間，每一種 **kind(resource type)** 的 object name 不能重複，例如：不能有兩個 name 同為 nginx 的 pod。

而每個 resource 都可以透過 API 進行存取，以 pod 為例，resource url 可能像是 `/api/v1/pods/some-name`。當然，為 object 取名也是有學問的，若是要研究此部份可參考[官方文件](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/identifiers.md)。



Namespace
=========

## What is namespace ?

假設有很多不同的使用者同時使用相同的 K8s cluster，K8s 提供了 `namespace` 讓一個 K8s cluster 上可區分出多個不同的 virtual cluster。

> 通常用來區分不同 team 的 user 或是不同的專案

大部分的 resource 都會隸屬於某個 namespace，不同 namespace 的 resource name 可以相同，每個使用者可以自訂一個自己的 namespace，並在此 namespace 中自訂自己的 resource。

> 有些特殊的 resource(e.g., node, persistent volume) 不屬於任何 namespace

而管理者也可以透過 [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 來定義每個 namespace 可使用的實體資源總量，亦可以 namespace 為單位設定 ccontrol policy。

而若是要在同一個 namespace 為不同的 resource 進行分門別類，則可搭配 [Label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 來完成。


## 檢視 cluster 中的 namespace

此外，namespace 其實也是 K8s resource 的一種，只是透過不同 resource 的搭配，可以達成資源隔離的效果：

```bash
$ kubectl get ns
NAME          STATUS    AGE
default       Active    1d
kube-system   Active    1d
kube-public   Active    1d
```

當 K8s 安裝完成後，預設就會有以下 3 個 namespace：

- **default**：若沒有指定 namespace 的 resource object 則會被歸類在 default

- **kube-system**：這個 namespace 內的 resource object 則都是屬於 K8s 系統本身所擁有的

- **kube-public**：在這個 namespace 中的 resource object 是可以被所有使用者所存取(visible & readable)的(也包含未認証的使用者)，通常是在 cluster level 被使用


## 使用 namespace

使用 namespace 的方式也很簡單，以下是幾個範例：

```bash
# 取得指定 namespace 的 pod 清單
$ kubectl --namespace=<insert-namespace-name-here> get pods

# 修改使用 kubectl 時預設的 namespace (若不修改，預設為 default)
$ kubectl config set-context $(kubectl config current-context) --namespace=<insert-namespace-name-here>
```


## namespace & DNS 的關係

當 [service](https://kubernetes.io/docs/concepts/services-networking/service/) 被建立後，為了讓 cluster 內部可以透過 domain name 存取 service，相對應的 [DNS entry](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) 也同時會被建立，而這個 DNS entry 的格式如下：

> <service-name\>.<namespace-name\>.svc.cluster.local

其中 `cluster.local` 是當初建立 cluster 的時候給入的設定，可以修改

> 若是在同一個 namespace 中的 pod，可直接透過 **<service-name>** 就存取到 service，不需指定完整的 FQDN


## 不屬於任何 namespace 的 resource

雖然大部分的 resource 都有所屬的 namespace，但也有一些 resource 是例外，通常這些都是 cluster level 的 resource，例如：**namespace**、**ClusterRole**、**ClusterRoleBinding** .... 等等。

詳細的清單可以透過以下指令查詢：

> kubectl api-resources --namespaced=false



Labels and Selectors
====================

## Label

Label 是個 key/value 的組合，使用者可以隨意為 object 賦予附加上自訂的 label(一個 or 多個)，作為每個 object 的屬性，來達成類似群組 or 分類的效果；此外，還可以透過 selector 進行 object 的選取。

> 在 k8s 中有個 **annotation** 的機制，同樣也是 key/value 的型式 & 為 object 增加不同的屬性資料，但 annotation 可乘載的資料較為大量，甚至可以是結構性的資料或是一段 script，但也就無法透過 select 來進行屬性的過濾

以下是幾個 Label 設定的範例：

- "**release**" : "stable", "**release**" : "canary"

- "**environment**" : "dev", "**environment**" : "qa", "**environment**" : "production"

- "**tier**" : "frontend", "**tier**" : "backend", "**tier**" : "cache"

- "**partition**" : "customerA", "**partition**" : "customerB"

- "**track**" : "daily", "**track**" : "weekly"

> 一個 object 可以有多個 label，而同樣的 label 設定可以套在不同的 object 上面，因此 label & resource object 之間其實是多對多的關係，然後仰賴後續介紹的 selector 來進行 resource 的過濾


## Selector

select 顧名思義，就是用來過濾 label 之用的，那要如何使用呢? 以下也有幾個 cli 範例：

```bash
$ kubectl get pods -l environment=production,tier=frontend

$ kubectl get pods -l 'environment in (production),tier in (frontend)'

$ kubectl get pods -l 'environment in (production, qa)'

$ kubectl get pods -l 'environment,environment notin (frontend)'
```

若是透過 YAML 宣告：

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

最後，也可以指定 schedule pod 到特定的 node。(使用 [nodeSelector](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/))

以下有個簡單範例，將 workload 丟到有強力顯示卡的機器上執行：

```yaml
# 將這個 pod schedule 到帶有 "accelerator=nvidia-tesla-p100" 上的 node 上執行
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```



進階的 Label 用法
===============

## 針對 Application 管理的 Label 使用方式

除了 kubectl & dashboard 之外，若想要視覺化的管理 k8s object，透過套用 common set label 來為不同的 object 增加更多的 metadata & 管理資訊是相當有用的，特別是對那些想自己開發管理工具的人來說。

k8s 原生就不是一個 PaaS，因此在設計上就不是以 **Application** 的角度為主來進行架構設計 & 開發，因此若是要針對 Application 進行深入的管理，透過使用 **app.kubernetes.io** 作為 prefix 的 label or annotation 就是一種作法。

> 一般來說 label 是 private to user，使用 `app.kubernetes.io` 的原因就是確保這樣子的 label 不會影響到 custom user label


## 相關的 Label 列表

透過 `app.kubernetes.io` prefix，有以下的 label 組合可以使用：

| Key | Description | Example | Type |
|-----|-------------|---------|------|
| app.kubernetes.io/name | Application 名稱 | mysql | string |
| app.kubernetes.io/instance | Application Instance 的識別用名稱 | wordpress-abcxzy | string |
| app.kubernetes.io/version | Application目前的版本號 | 5.7.21 | string |
| app.kubernetes.io/component | 在 Application 架構中的元件 | database | string |
| app.kubernetes.io/part-of | 更上層的 Application 名稱 | wordpress | string |
| app.kubernetes.io/managed-by | 用來管理 Application 佈署 & 操作的工具 | helm | string |

使用上面表格中的資訊，轉換成可設定在 k8s resource object 中的 YAML 格式，就會變成如下：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: "mysql"
    app.kubernetes.io/instance: "wordpress-abcxzy"
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: "database"
    app.kubernetes.io/part-of: "wordpress"
    app.kubernetes.io/managed-by: "helm"
```


## 當 Application 被多次佈署到 Kubernetes Cluster 時

以上述的例子來說，一個 wordpress application 可以被佈署多次到 k8s cluster 中，每一次的佈署都稱為 **Application Instance**，雖然每個 application name 都是 **wordpress**，但每個 application instance 都會有各自不同的 instance ID，例如：

- app.kubernetes.io/instance: "wordpress-abcxzy"

- app.kubernetes.io/instance: "wordpress-defstu"

透過不同的 instance 名稱，就可以知道同一個 application，可能被不同的使用者佈署，或是在不同的專案同時被使用


## 範例

以下是使用 Helm 安裝 wordpress(Web + DB) 時，label 設定的範例：

```yaml
# Web 以 Deployment 的形式佈署
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress


# 用來提供外部連線處理的 Service，串到後面的 Web 服務
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress

# DB(MySQL) 則是以 StatefulSet 的形式佈署
# 並可看出與那個 application instance 相關
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/version: "5.7.21"

# 用來提供 DB 連線介面給 Web(wordpress) 的 service
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/version: "5.7.21"
```

從上面 Label **app.kubernetes.io/instance** 設定的都是相同的 value 來看，就可以很明確的看出，這幾個 resource object 構成了一個完成的 Wordpress 服務。



Annotations
===========

## What is Annotations ?

Annotations 跟 上面提到的 label 相同，是以 key/value 的型式儲存資料，但不同的是，Annotations 並不是儲存用來辨識 resource object 用的，主要就是作為 metadata 來當類似"**註解**"的方式使用，大概有幾個特點：

- 並非用來辨識 resource object 之用

- 註解的資料量通常是較大的

- 資料可以是較為複雜的結構化資料(structured data)，當然也可以是非結構化的資料

- tool or library 等 client 都可以存取此 metadata


## 如何設定 Annotations

以下是簡單一點的 JSON 格式範例：

```json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

也可以是複雜的資料格式：(以 YAML 格式示範)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress-with-annotations
  annotations:
    nginx.org/proxy-connect-timeout: "30s"
    nginx.org/proxy-read-timeout: "20s"
    nginx.org/client-max-body-size: "4m"
    nginx.org/location-snippets: |
        if ($ssl_client_verify = SUCCESS) {
            set $auth_basic off;
        }
        if ($ssl_client_verify != SUCCESS) {
            set $auth_basic "Restricted";
        }
        auth_basic $auth_basic;
        auth_basic_user_file "/var/run/secrets/nginx.org/auth-basic-file";
    nginx.org/server-snippets: |
        ssl_verify_client optional;
spec:
....... (以下略)
```

然而，Annotations 的設定不一定是由使用者套用到 resource object 上的，也可以是透過 auto-sizing or auto-scaling system 來自動生成相關 metadata 並套用到 object 上，讓 control manager 透過監控這一類的資訊來調整 object 狀態的變更。


## 那些資訊適合用 Annotation 呈現?

以下幾個例子說明：

- auto-sizing or auto-scaling system 自動生成的資訊

- build, release 或是與 image 相關的資訊，例如：timestamp, release ID, Git branch, PR numbers, image hash, registry address ... 等等

- 與 client library/tool 相關且用來 debug 用的資訊，例如：name, version, build information

- tool/system 的來源資訊，例如：URL

- 負責人聯絡資訊，例如：電話, E-Mail ....等等



Field Selectors
===============

## What is Field Selectors?

k8s 除了提供了上面使用 label 的方式搜尋對應的 resource object 之外，還另外支援了另外一種稱為 "**Field Selectors**" 的搜尋方式。

透過 Field Selectors，使用者可以根據 object 中的 `Field` 作為搜尋的依據，以下是幾個範例：

- metadata.name=my-service

- metadata.namespace!=default

- status.phase=Pending

實際操作的範例如下：

```bash
# 搜尋目前 status 為 running 的 pod
$ kubectl get pods --field-selector status.phase=Running
```

## 那些 Field 支援搜尋功能

每種不同的 resource type 支援不同的 Field，但所有的 resource type 都支援以下兩種：

1. metadata.name

2. metadata.namespace

若是使用者操作時針對不支援的 Field 做搜尋，就會出現類似下面的錯誤訊息：

```bash
$ kubectl get pod -n kube-system --field-selector foo.bar=baz
No resources found.
Error from server (BadRequest): Unable to find {"" "v1" "pods"} that match label selector "", field selector "foo.bar=baz": field label not supported: foo.bar
```

## 進階搜尋

### Supported operators

除了上面示範的 **=**(or **==**) 之外，同時也支援了 **!=**，但也僅此兩種而已，不像 label 支援 subset 的搜尋

### Chained selectors

還可以將多個搜尋條件作結合，就是類似 **AND** 的效果，例如：

```bash
# 搜尋 kube-system namespace 中，狀態為 running & restartPolicy 為 Always 的 pod
$ kubectl get pods -n kube-system --field-selector=status.phase==Running,spec.restartPolicy=Always
```

### Multiple resource types

也可以一次針對多個 resource type 進行搜尋，例如：

```bash
# 同時搜尋名稱為 myweb 的 pod & service
$ kubectl get pod,service --field-selector metadata.name=myweb
```
 


References
==========

- [Understanding Kubernetes Objects - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

- [Names - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)

- [Namespaces - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

- [Labels and Selectors - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

- [Annotations - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

- [Field Selectors - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)

- [Recommended Labels - Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)