---
layout: post
title:  "[Prometheus] Service Discovery & Relabel "
description: "本篇文章介紹如何使用 Prometheus 本身提供的 service discovery 機制(kubernetes_sd_configs) 搭配 relabel 的功能，對 Kubernetes resource 進行監控"
date: 2019-05-27 16:50:00
published: true
comments: true
categories:
  - Prometheus
tags:
  - Prometheus
  - Kubernetes
  - Telemetry
  - Service Discovery
---


Preface
=======

Relabel 是在 Prometheus 中一個強大的功能，善用 relabel 可以讓資料真正進入到資料庫之前，根據需求完成一些前置處理，了解 relabel 功能怎麼用，在使用 Prometheus 上就可以有許多變化。

若是使用到 service discovery 的功能就會發現，動態的 instance 要用 static configuration 做到想要的設定是很困難的，而藉由 relabel 的功能就可以根據預先定義好的規則，將資料進行處理。

以下以 Kubernetes 為例(使用 `kubernetes_sd_configs`)，做一些 relabel 的示範 & 說明，而在 **kubernetes_sd_configs**，共有以下五種 role 的資料可以動態擷取，分別是：

- node

- service

- pod

- endpoints

- ingress


Node
====

一開始設定使用 `kubernetes_sd_configs` 取得 `node` 的資訊，設定如下：

```yaml
... (略)
scrape_configs:
- job_name: kubernetes-nodes-kubelet
  kubernetes_sd_configs:
  - role: node
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: true
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

套用到 Prometheus 之後，發現抓取到的資料似乎不太像是我們想要的，以下的圖是 relable 之前的樣子：

![Prometheus - k8s before relabel](/blog/images/prometheus/prometheus-k8s-nodes-before-relabel.png)

接著我們要做的是，將原本以 external IP 擷取 node status 的方式，改成透過 internal k8s API server 來取得資訊，而從 k8s cluster 內部要取得 node metric 的方式，是透過以下的網址：

> https://kubernetes.default.svc:443/api/v1/nodes/`[NODE_NAME]`/proxy/metrics/cadvisor

但目前的 endpoint 是 `https://10.103.9.x:10250/metrics`，因此以下要來進行一些 relabel 的動作：


## 修改 Endpoint IP

首先要將 Endpoint 中的 IP 改成 `kubernetes.default.svc:443`，可以透過以下設定：

```yaml
relabel_configs:
  # 要修改 endpoint 中的 `__address__` value，並非修改 Label 本身
  - target_label: __address__
    replacement: kubernetes.default.svc:443
```

首先必須了解 Endpoint 的值是由 `__scheme__` + `__address__` + `__metrics_path__` 所組成，而上面的設定就是要將 `__address__` 的部份換成 `kubernetes.default.svc:443`，因此以上的 relabel 設定就會變成以下結果：

![Prometheus - k8s after relabel](/blog/images/prometheus/prometheus-k8s-nodes-after-relabel-1.png)

但很明顯，這還沒到結束的程度，Endpoint 的部份還需要更多的加工。

> `/metrics` 是預設抓取 metric 的路徑，可以透過 `scrape_configs[].metrics_path` 調整

## 修改 metric path

接著要將 metric path 從 `/metrics` 變成 `/api/v1/nodes/[NODE_NAME]/proxy/metrics/cadvisor`，但麻煩的事情是，中間 `NODE_NAME` 的部份會隨著每個 node 而改變，因此這是動態的，那要怎麼解決?

答案是從現有的 Label metata 中找尋是否有可用的資訊，透過第一張圖可以發現，我們可以用 `__meta_kubernetes_node_name` 這個 label 取得每個 node 所使用的 node name。

有了以上的資訊後，可以使用以下設定來修改 metric path：

```yaml
relabel_configs:
    # 使用 Label "__meta_kubernetes_node_name" 的值
  - source_labels: [__meta_kubernetes_node_name]
    # 使用所有的值，不經過額外處理
    regex: (.+)
    # 取代 endpoint 中 "__metrics_path__" 的值
    target_label: __metrics_path__
    # ${1} 的部份會由 "__meta_kubernetes_node_name" 的值取代
    replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

透過以上的 relabel 設定後，結果會變成如下：

![Prometheus - k8s after relabel](/blog/images/prometheus/prometheus-k8s-nodes-after-relabel-2.png)

## 為資料標記更多 Label

從上圖中可以看到，在 Label 的部份目前只有一個 instance，因此之後在做資料分析 or 處理時，僅有 instance 可以作為依據來處理，這樣看起來似乎不足；此外 `kubernetes_sd_configs` 提供了許多 metadata 資訊可供使用，而且很多是富有意義的，例如：

- `__meta_kubernetes_node_label_beta_kubernetes_io_os`

- `__meta_kubernetes_node_label_beta_kubernetes_io_arch`

- `__meta_kubernetes_node_label_kubernetes_io_hostname`

有了上面的 metadata 資訊，我們就可以額外對資料根據 OS, architecture, hostname 進行分類，為了達到此目的，我們可以透過 `labelmap` 的方式，為每一筆 time series data 加入新的 label 資訊：

```yaml
relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
```

以上的設定會將 `__meta_kubernetes_node_label_` 開頭的 metadata，拆開後加入 Label 中，並會在之後的每一筆擷取到的資料中加上這些 label 的資訊，方便使用者進行分類查詢 or 使用。

以下是設定 labelmap 後的結果：

![Prometheus - k8s after relabel](/blog/images/prometheus/prometheus-k8s-nodes-after-relabel-3.png)

## 結論

經過上面 3 個 relabel 設定後，我們就可以在 k8s cluster 內部的 prometheus server，透過 k8s API server 來取得每個 node 的狀態，以下是完整 configmap 設定：

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: "prometheus-config"
  namespace: kube-system
data:
  prometheus.yml: |-
    global:
      scrape_interval: 15s
      external_labels:
        monitor: 'codelab-monitor'

    scrape_configs:
    - job_name: kubernetes-nodes-kubelet
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        insecure_skip_verify: true
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      
      relabel_configs:
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
```


Pod
===

一開始先開啟 pod scrape 的功能：

```yaml
- job_name: kubernetes-pods
  kubernetes_sd_configs:
  - role: pod
```

以下的圖是尚未做任何 relabel 設定的結果：

![Prometheus - k8s pod before relabel](/blog/images/prometheus/prometheus-k8s-pods-before-relabel.png)

![Prometheus - k8s pod before relabel](/blog/images/prometheus/prometheus-k8s-pods-before-relabel-2.png)

## 只留下需要監控的 Pod

在 k8s 中有一些 pod 其實不是我們想要關心的，真的想要關心的 pod 會在 YAML configuration 中設定 `metadata.annotations.prometheus.io/scrape` 設定為 `true`，prometheus 在擷取 pod 相關資訊時，就會取得 Label `__meta_kubernetes_pod_annotation_prometheus_io_scrape` 的值為 **true**，因此透過以下設定只取得我們想要留下的 pod 資訊：

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: "keep"
    regex: "true"
```

上面的設定會檢查 Label `__meta_kubernetes_pod_annotation_prometheus_io_scrape` 的值，只有是 `true` 的 target 才會被保留下來，結果很多 pod 就被過濾(drop)掉了：

![Prometheus - k8s pods after relabel](/blog/images/prometheus/prometheus-k8s-pods-after-relabel-1.png)

接著還要再移除掉 `calico-node`，為什麼呢? 因為 calico 有專門口以用來監控的工具 [felix](https://docs.projectcalico.org/v3.5/reference/felix/prometheus)，所以這邊就先拿 calico 的部份：

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_container_name]
    action: "drop"
    regex: "calico-node"
```

上面的設定會檢查 Label `__meta_kubernetes_pod_container_name` 的值，如果是 `calico-node`，就會被過濾掉：

![Prometheus - k8s pods after relabel](/blog/images/prometheus/prometheus-k8s-pods-after-relabel-2.png)

## 修改監控用的 port & metrics path

在預設的情況下，prometheus 擷取資料的 endpoint 為 `__scheme__` + `__address__` + `__metrics_path__`，但如果實際要擷取資料的位置不是這 3 個變數的組合呢? 就可以用 relabel 來調整。

透過 YAML 佈署 k8s pod 時，可以額外加上以下兩種資訊：

- `spec.template.metadata.annotations.prometheus.io/port`

- `spec.template.metadata.annotations.prometheus.io/path`

就會在 Target 中多出以下 Label 可以用：

- `__meta_kubernetes_pod_annotation_prometheus_io_port`

- `__meta_kubernetes_pod_annotation_prometheus_io_path`

利用以上的 Label 就可以用來調整實際擷取 metrics 資訊的位置，以下是調整範例：

```yaml
relabel_configs:
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    target_label: __address__
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: "(.+)"
```

## 為資料標記更多 Label

### 增加 pod Label 資訊

在佈署 pod 的過程中，我們通常會透過 Label(`spec.template.metadata.labels`) 的方式為 pod 加上一些 metadata，例如：`app: prometheus`, `app: node-exporter`，而這樣的資料，在 prometheus 中會自動生成相對應的 metadata Label 可用，以上面的例子來說：

- `app: prometheus` -> `__meta_kubernetes_pod_label_app="prometheus"`

- `app: node-exporter` -> `__meta_kubernetes_pod_label_app="node-exporter"`

而我們希望可以在每一筆 time series data 中可以加上這樣的 metadata，因此就可以透過 relabel 的方式來處理：

```yaml
relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
```

結果變成如下：

![Prometheus - k8s pods after relabel](/blog/images/prometheus/prometheus-k8s-pods-after-relabel-3.png)

### 增加 namespace 資訊

此外，由於 k8s cluster 中資源的 isolation 是用 namespace 實現的，因此我們希望每筆 time series data 可以同時帶著 namespace 的資訊，這時可以從 `__meta_kubernetes_namespace` 來著手，利用以下設定來完成：

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
```

結果就多了 `kubernetes_namespace` label：

![Prometheus - k8s pods after relabel](/blog/images/prometheus/prometheus-k8s-pods-after-relabel-4.png)

### 增加 pod name 資訊

同上，我們希望可以增加 pod name 資訊(資料來源為 `__meta_kubernetes_pod_name`)，可用下面的設定：

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
```

結果就多了 `kubernetes_pod_name` label：

![Prometheus - k8s pods after relabel](/blog/images/prometheus/prometheus-k8s-pods-after-relabel-5.png)

## 結論

經過上面的 relabel 設定後，我們就可以在 k8s cluster 內部的 prometheus server，取得我們所需要監控的 pod 資訊，以下是完整 configmap 設定：

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: "prometheus-config"
  namespace: kube-system
data:
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      external_labels:
        monitor: 'codelab-monitor'

    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: "keep"
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_container_name]
        action: "drop"
        regex: "calico-node"
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: "(.+)"
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
```


Service
=======

在 kubernetes service 的部份，我們希望可以透過 probe 的方式，檢查 service 是否會回應，首先先開啟 k8s 的 service scrape 的功能：

```yaml
scrape_configs:
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
    - role: service
```

以下是佈署後的結果結果：

![Prometheus - k8s service before relabel](/blog/images/prometheus/prometheus-k8s-services-before-relabel.png)

## 化繁為簡

此時假如需要監控的 service 有上百個呢? 同時對上百個 service probe 對 prometheus 來說也是挺辛苦的事情，此時就可以在 prometheus 與眾多 service 之間放一個 aggregator 的角色，這裡我們選用的是 [Blackbox Prober Exporter](https://github.com/prometheus/blackbox_exporter)。

> Blackbox Exporter 允許使用者透過 HTTP、HTTPS、DNS、TCP & ICMP .... 等協定來進行 probe

在 k8s 中套用以下設定以安裝 blackbox exporter：(使用的 namespace 為 `kube-system`)

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
  namespace: kube-system
spec:
  ports:
  - name: blackbox
    port: 9115
    protocol: TCP
  selector:
    app: blackbox-exporter
  type: ClusterIP

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: blackbox-exporter
  name: blackbox-exporter
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blackbox-exporter
  template:
    metadata:
      labels:
        app: blackbox-exporter
    spec:
      containers:
      - image: prom/blackbox-exporter
        imagePullPolicy: IfNotPresent
        name: blackbox-exporter
```

在設定 blackbox exporter 之前，我們要了解一下 blackbox exporter 的使用方式，首先要先說明架構，原本的架構如下：

> `Prometheus` --> `Service`

若是加入了 blackbox exporter 之後，會變成如下：

> `Prometheus` --> `Blackbox Exporter` --> `Service`

而正確的使用的方式呢? 要注意的地方有兩點：

1. endpoint 要改成 `http://[Blackbox_URL]:9115/probe`

2. 要額外給入 target，告知 blackbox exporter 要探測的對象

所以此時 prometheus 的設定會變成如下：

```yaml
scrape_configs:
  - job_name: 'kubernetes-services'
    metrics_path: /probe
    params:
      # 指定 blackbox exporter 使用 http_2xx module
      # 同時也有 http_post_2xx, tcp_connect, ping ... 等 module 可用
      module: [http_2xx]
    kubernetes_sd_configs:
    - role: service
    relabel_configs:
    # 將 service URL 的資訊新增到 target 中
    - source_labels: [__address__]
      target_label: __param_target
    # 將 endpoint 全部改成 blackbox exporter URL
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

以下是 relabel 後的結果：

![Prometheus - k8s services after relabel](/blog/images/prometheus/prometheus-k8s-services-after-relabel-1.png)


## 修正錯誤的 instance

從上圖可以看出，雖然我們已經將擷取資料的對象從多個 service 改成單一的 blackbox exporter，但同時也把 `instance` 變成了 blackbox exporter，如此一來 time series data 中就無法判斷資料是來自於哪一個 service，因此可以透過以下設定來修正這個問題：

```yaml
relabel_configs:
# 將 instace 的值修改成 target 的值
- source_labels: [__param_target]
  target_label: instance
```

以下是 relabel 後的結果：

![Prometheus - k8s services after relabel](/blog/images/prometheus/prometheus-k8s-services-after-relabel-2.png)

## 為資料標記更多 Label

但僅有 instance label 是不夠的，我們還可以用以下的設定加入更多的 label，以便於後續的資料處理：

```yaml
relabel_configs:
- action: labelmap
  regex: __meta_kubernetes_service_label_(.+)

- source_labels: [__meta_kubernetes_namespace]
  target_label: kubernetes_namespace

- source_labels: [__meta_kubernetes_service_name]
  target_label: kubernetes_service_name
```

以下是 relabel 後的結果：

![Prometheus - k8s services after relabel](/blog/images/prometheus/prometheus-k8s-services-after-relabel-3.png)

## 僅留下想要監控的 service

如果不是每個 service 都想要監控呢? 那可以在需要監控的 service YAML 定義加上 `metadata.annotations.prometheus.io/probe: 'true'`，並將以下的 relabel 設定在第一條規則：

```yaml
relabel_configs:
- source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
  action: keep
  regex: 'true'
```

## 結論

經過上面的 relabel 設定後，我們就可以在 k8s cluster 內部的 prometheus server，取得我們所需要監控的 service 資訊，以下是完整 configmap 設定：

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: "prometheus-config"
  namespace: kube-system
data:
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      external_labels:
        monitor: 'codelab-monitor'

    scrape_configs:
    - job_name: 'kubernetes-services'
      metrics_path: /probe
      params:
        module: [http_2xx]
      kubernetes_sd_configs:
      - role: service
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: 'true'
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.monitoring.svc.cluster.local:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_service_name
```


Endpoint & Ingress
==================

Endpoint & Ingress 在 relabel 部份的作法其實跟上面大同小異，甚至有很多重複的部份，以下僅列出範例的設定方式，使用者可以根據自己的監控需求來進行增減：

## Endpoint

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: "prometheus-config"
  namespace: kube-system
data:
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      external_labels:
        monitor: 'codelab-monitor'

    scrape_configs:      
    - job_name: 'kubernetes-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)(?::\d+);(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_service_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name
```

## Ingress

Ingress 同樣建議搭配 [Blackbox Prober Exporter](https://github.com/prometheus/blackbox_exporter) 一起使用

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: "prometheus-config"
  namespace: kube-system
data:
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      external_labels:
        monitor: 'codelab-monitor'

    scrape_configs:
    - job_name: 'kubernetes-ingress'
      kubernetes_sd_configs:
      - role: ingress
      relabel_configs:
      - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
        regex: (.+);(.+);(.+)
        replacement: ${1}://${2}${3}
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_ingress_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_ingress_name]
        target_label: kubernetes_name
```


References
==========

- [prometheus/blackbox_exporter: Blackbox prober exporter](https://github.com/prometheus/blackbox_exporter)

- [网络探测：Blackbox Exporter - prometheus-book](https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/exporter/commonly-eporter-usage/install_blackbox_exporter)

- [使用Prometheus的blackbox_exporter进行网络监控 — 青蛙小白](https://blog.frognew.com/2018/02/prometheus-blackbox-exporter.html)

- [Javascript Regular Expressions , 表示法](https://www.puritys.me/docs-blog/article-30-Javascript-Regular-Expressions-,-%E8%A1%A8%E7%A4%BA%E6%B3%95.html)

- [RegExr: Learn, Build, & Test RegEx](https://regexr.com/)

- [使用Prometheus完成Kubernetes集群监控 · 极术](https://jishu.io/kubernetes/kubernetes-monitoring-with-prometheus/)