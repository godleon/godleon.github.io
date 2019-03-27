---
layout: post
title:  "[Prometheus] Basic Concept memo"
description: "這篇文章單純是研究 Prometheus 時隨手記下的筆記...."
date: 2019-03-24 17:00:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Prometheus
  - Monitoring
---


這篇文章單純是研究 [Prometheus](https://prometheus.io/) 時隨手記下的筆記....


關鍵名詞
=======

## Sample

sample 表示實際的時間序列資料，包含以下內容：

- 一個精度為 float64 的值

- 一個以 ms 為最小單位的 timestamp

## Instance

泛指 Prometheus 擷取監控資料的 HTTP(s) endpoint，一般會是一個運行中的 process。

## Job

一群 instance 的集合，稱為一個 Job，一般會是用來收集相同目的的資料，例如：因為 scalability & reliability 而讓同一個 process 跑在多台機器上，然後同時監控多台機器中的該 process。


Data Model
==========

Prometheus 中的 multiple dimension 是透過 `metric` + `label` 所呈現出來的，同樣的 metric 資料，但透過不同的 label 搭配可以呈現出不同的維度的資料。

## 如何表示資料

資料的表示需要 `metric` & `label` 兩種資訊，範例如下：

> <metric name>{<label name>=<label value>, ...}

以下是個實際範例：

> api_http_requests_total{method="POST", handler="/messages"}

以上的資料表示法可以在 Prometheus console 的查詢畫面中使用。


Jobs & Instances
================

Prometheus 除了收集 instance metrics 外，還會有其他自動產生並帶上相對應 label 的資料，例如：

- `up{job="<job-name>", instance="<instance-id>"}`：`1` 表示 instance 目前很健康，可以正常取得 metric 資料；若 `0` 則反之

- `scrape_duration_seconds{job="<job-name>", instance="<instance-id>"}`：擷取 metric 資料的時間間隔

- `scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}`：有多少個 sample

- `scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}`


Service Discovery
=================

在一般 Prometheus 的 `scrape_configs` 設定中，會看到 `static_configs` 的設定，用來指定 metric endpoint，使用者可以根據需要設定 endpoint list。

但若是 metrics endpoint 時常變動時怎麼辦? 例如在 OpenStack or Kubernetes 的環境中，VM or Pod 可能都是來來去去，IP address 都是會一直變動的，此時 `static_configs` 根本無法正確指定要擷取資料的 instance。

為了解決這個問題，就要透過 **service discovery** 的方式來動態取得 instance list，而 Prometheus 支援很多種 service discovery 的機制，例如：

- [file_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#file_sd_config)

- [kubernetes_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)

- [openstack_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#openstack_sd_config)

除了上面的幾個之外，Prometheus 還支援了很多種其他的服務，也包含了多個 public cloud，詳細的清單可以參考[官網](https://prometheus.io/docs/prometheus/latest/configuration/configuration)。


以下蒐集了一些幾個 service discovery 的範例供參考：

## consul_sd_configs

- [Prometheus中的服务发现和relabel | I'm Yunlong](http://ylzheng.com/2018/01/17/prometheus-sd-and-relabel/)

- [Consul+Prometheus系统监控之注册发现 - 柒's Blog](https://blog.52itstyle.vip/archives/2084/)

## kubernetes_sd_config

- [Prometheus在Kubernetes下的监控实践 | I'm Yunlong](http://ylzheng.com/2017/07/04/prometheus-kubernates/)

- [监控Kubernetes集群 - prometheus-book](https://yunlzheng.gitbook.io/prometheus-book/part-iii-prometheus-shi-zhan/readmd/use-prometheus-monitor-kubernetes)



在 Kubernetes 上佈署 Prometheus 的流程
====================================

若是 Prometheus 佈署在 k8s cluster 之外，從外面與 API server 溝通，就必須要準備好正確的憑證資訊。

但如果將 Prometheus 放到 k8s cluster 內部，就可以使用 k8s 提供的一些方法來取得正確的憑證資訊，流程如下：

1. 建立存放 Prometheus 服務所需要的 namespace (optional)

2. 建立 Prometheus 所使用的 ServiceAccount (每個 ServiceAccount 會帶有一個 token)

3. 設定 RBAC，並賦予 ServiceAccount 足夠的存取權限

4. 佈署 Prometheus 指定使用上一個步驟所設定好的 ServiceAccount

5. 在 Prometheus 設定檔中，進行以下設定：
    - `scrape_configs.tls_config.ca_file`: `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`
    - `scrape_configs.bearer_token_file`: `/var/run/secrets/kubernetes.io/serviceaccount/token`

如此一來，Prometheus pod 就可以跟 k8s API server 通訊了!


Rules
=====

Prometheus 中提供了兩種 rule type，分別是：

- [Recording Rule](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)：用來協助預先處理 or 計算某些 metric 的值，透過 recording rule，將較為複雜的需求先計算完成，後續就可以直接使用，會比起每次都重新查詢 & 計算來的快的多且省資源

- [Alerting Rule](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)：讓使用者可以透過 rometheus expression language expressions 自訂需要發送 alarm 的條件

rule 在使用上有幾點需要注意：

- rule 可以獨立成一個個單獨的檔案並 include，不一定要全部設定在主設定檔 `prometheus.yml` 上。

- 透過送給 prometheus process `SIGHUP` 的訊號，可以在執行期間載入 rule


## Recording Rule 範例

```yaml
groups:
  - name: example
    rules:
    - record: job:http_inprogress_requests:sum
      expr: sum(http_inprogress_requests) by (job)
```

## Alerting Rule 範例

```yaml
groups:
- name: example
  rules:
  - alert: HighErrorRate
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    # alert 成立後，進入 pending 狀態，持續超過 10 mins 則進入 firing 
    for: 10m
    # 為 alert 標記 tag
    labels:
      severity: page
    # 可儲存較為複雜的 metadata 資訊
    annotations:
      summary: High request latency
```

與 [Console Templates](https://prometheus.io/docs/visualization/consoles/) 搭配使用：

```yaml
groups:
- name: example
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: InstanceDown
    expr: up == 0
    for: 5m
    labels:
      severity: page
    annotations:
      summary: "Instance {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."

  # Alert for any instance that has a median request latency >1s.
  - alert: APIHighRequestLatency
    expr: api_http_request_latencies_second{quantile="0.5"} > 1
    for: 10m
    annotations:
      summary: "High request latency on {{ $labels.instance }}"
      description: "{{ $labels.instance }} has a median request latency above 1s (current value: {{ $value }}s)"
```


儲存系統
=======

可參考以下文章：

- [Prometheus 儲存系統解析 - KaiRen's Blog](https://k2r2bai.com/2018/06/27/devops/prometheus-storage/)

- [Prometheus高可用(1)：理解本地存储 | I'm Yunlong](http://ylzheng.com/2018/03/06/promethus-local-storage/)

- [Prometheus高可用(2)：理解远端存储 | I'm Yunlong](http://ylzheng.com/2018/03/07/promethues-remote-storage/)


References
==========

- [Prometheus 介紹與基礎入門 - KaiRen's Blog](https://k2r2bai.com/2018/06/10/devops/prometheus-intro/)

- [Prometheus 儲存系統解析 - KaiRen's Blog](https://k2r2bai.com/2018/06/27/devops/prometheus-storage/)

- [了解 Prometheus Federation 功能 - KaiRen's Blog](https://k2r2bai.com/2018/06/29/devops/prometheus-federation/)

- [Prometheus 高可靠實現方式 - KaiRen's Blog](https://k2r2bai.com/2018/07/01/devops/prometheus-ha/)

- [Kubernetes - Prometheus Add-on](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/prometheus)
