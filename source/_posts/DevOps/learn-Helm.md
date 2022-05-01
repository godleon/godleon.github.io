---
layout: post
title:  "Helm 學習筆記"
description: "此篇文章是在學習 Udemy 線上課程(HELM - Package Manager for Kubernetes Complete Master Course)的過程中所留下的學習筆記"
date: 2022-05-01 15:15:00
published: true
comments: true
categories:
  - Helm
tags:
  - Helm
  - DevOps
  - Kubernetes
---


常用指令(置頂)
============

以下將常用指令置頂：

## 開發階段

```bash
# 可以不安裝的方式顯示 release manifest (適合用來 debug 用)
$ helm install --debug --dry-run [RELEASE_NAME] [CHART_PATH]

# 直接透過指令複寫原本 values.yaml 中的特定內容
# 透過此方式可以針對特定的 release 進行客製化修改
＄helm install --debug --dry-run --set [VALUES_KEY]=[VALUES_VALUE] [RELEASE_NAME] [CHART_PATH]

# 列出已經安裝的 helm chart
$ helm list
```

## 安裝後狀態檢查

```bash
# 顯示指定 release 的 YAML 內容
$ helm get manifest [RELEASE_NAME]

# 顯示指定 release 所使用的 values 內容
$ helm get values [RELEASE_NAME]

# 顯示指定 release 的 notes 資訊
$ helm get notes [RELEASE_NAME]

# 顯示指定 release 使用了那些 hook
$ helm get hooks [RELEASE_NAME] 

# 顯示指定 release 所有的內容
$ helm get all [RELEASE_NAME] 

# 顯示指定 release 目前的狀態
$ helm status [RELEASE_NAME] 
```


## 管理 Helm Chart Repository

安裝 S3 plugin：

```bash
# 安裝最新版 S3 plugin
$ helm plugin install https://github.com/hypnoglow/helm-s3.git

# 安裝指定版本 S3 plugin
$ helm plugin install https://github.com/hypnoglow/helm-s3.git --version 0.10.0

# 顯示目前安裝的 plugin
$ helm plugin list
NAME	VERSION	DESCRIPTION                                                                                
s3  	0.10.0 	The plugin allows to use s3 protocol to upload, fetch charts and to work with repositori...
```

初始化 S3 bucket：(需要有[足夠的 AWS S3 存取權限](https://github.com/hypnoglow/helm-s3#configuration))

```bash
# 初始化 Helm Chart Repo (假設 helm chart 要放在 `/test` prefix 下，階層可以自訂沒問題)
$ helm s3 init s3://my-helm-charts/test

# 驗證是否有正確產生 index.yaml 檔案(Helm Chart index file)
$ aws s3 ls s3://my-helm-charts/test/
```

加入 Helm Chart Repository：

```bash
# 加入 S3 bucket (名稱為 "stable-myapp")
$ helm repo add stable-myapp s3://my-helm-charts/test

# 加入標準的 HTTP repository
$ helm repo add another-repo https://example.com/charts

# 列出目前加入的 Helm Chart Repo
$ helm repo list

# 更新 Helm Chart Repo (所有)
$ helm repo update

# 搜尋指定 repository 有哪些 Helm Chart 可用
# or 哪些 Helm Chart 帶有特定關鍵字
$ helm search repo [SEARCH_KEYWORD]
```

## 打包 Helm Chart 並上傳

```bash
# 先更新 Helm Chart repo，取得最新的 index.yaml
$ helm repo update

# 打包 Helm Chart
$ helm package [HELM_CHART_FOLDER]

# push 到 S3 bucket (會協助更新 index.yaml)
$ helm s3 push [HELM_CHART_NAME]-0.1.0.tgz [REPO_NAME]

# push 到 HTTP Repo (會協助更新 index.yaml)
$ helm push [HELM_CHART_NAME]-0.1.0.tgz [REPO_NAME]
```

## 安裝/升級/退版 Helm Chart 安裝

```bash
# 從指定的 Helm Chart Repo 安裝指定最新版的 Helm Chart
$ helm install [RELEASE_NAME] [CHART_REPO_NAME]/[CHART_NAME]

# 升級指定的 RELEASE (若是有更新版的 Helm Chart 被送進 repo)
$ helm upfrade [RELEASE_NAME] [CHART_REPO_NAME]/[CHART_NAME]

# 查詢指定 release 安裝歷史
$ helm history [RELEASE_NAME]

# 退版到指定版本
$ helm rollback [RELEASE_NAME] [REVISION_NO]
```

## 測試 & 驗證 Helm Chart

```bash
# 檢查 Helm Chart 是否有語法上的錯誤
$ helm lint [CHART_FOLDER]
```



Templating 的應用
================

## Go template 相關資訊

- Go templating 的語法，可以參考 [template · pkg.go.dev](https://pkg.go.dev/text/template)

- 若要查詢 template function(例如：**quote**, **upper**)，可到 [sprig](http://masterminds.github.io/sprig/) 網站


## pipeline 應用

可以透過 pipeline 的方式對 value 套用多個 function：

```yaml
# 一般使用方式
projectCode1: {{ upper .Values.projectCode }}
# 使用 pipeline 套用多個 function
projectCode2: {{ .Values.projectCode | upper | quote }}

# 中間的 date function 其實只是指定時間的格式而已，不會修改 now 的結果
now: {{ now | date "2006-01-02" | quote }}

# 若是 values.yaml 中沒有 contact 的值，就使用預設的
contact: {{ .Values.contact | default "1-123-456-7890" | quote }}
```


## Flow control - if/else

以下是 if/else 的基本格式：

```yaml
{{ if PIPELINE }}
  # do something
{{ else if PIPELINE }}
  # do something else
{{ else }}
  # default case
{{ end }}
```

以下情況下 if 會被判定為 false：

- Boolean false

- 數字為 0

- 空字串

- nil (空白 or null)

- 空集合 (map, slice, tuple, dict, array)

需要注意的地方：

- indent 的問題需要注意，錯誤就會造成 YAML 文件有問題無法使用

- whitespace & newline 可以透過 `{{-` or `-}}` 過濾掉，但同時擺上 `{{- if PIPELINE -}}` & `{{- end -}}` 可能會讓結果不如預期 (可參考[官網詳細說明](https://helm.sh/docs/chart_template_guide/control_structures/#controlling-whitespace))


## 使用 with

- 使用 `with` 可用來限定變數的使用範圍，在寫 template 可以不用寫得很冗長的變數完整階層，但在 with 中無法使用其他變數，即使是 built-in object(例如：`Release`) 也不行
> 使用 `range` 也會有相同限制


## 自訂 variable

- 自訂變數用，可自行組合，例如搭配 built-in object 的值，語法是 `{{- $relname := .Release.Name -}}`

- 自訂的 variable 可以放進 `with` 內，因此搭配自訂 variable，就可以在 `with` 中使用 built-in object 的值
> range 也可以透過自動 variable 的方式將指定變數之外的內容放進來

- `$` 符號代表 [Global chart value](https://helm.sh/docs/chart_template_guide/subcharts_and_globals/#global-chart-values)，有一些內建的 context 可用，例如：`Release`(`.Release.Name`, `.Release.Service` ... etc) & `Chart`(`Chart.Version`, `Chart.AppVersion` ... etc)


## 自訂 template

- 透過 define 關鍵字可以自訂一個 template，並在 template 其他的地方引用
> 自訂的 template 可以定義在獨立的檔案，也可以與引用它的檔案放在一起

```yaml
{{- define "mychart.workload_labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
data:
  myvalue: "Hello World"
```

- 自訂的 template 建議放在獨立的檔案，一般會定義在 `_helpers.tpl` 檔案中

- 在自訂的 template 中使用 built-in object value，引用時會發現 value 是空白的，原因是因為沒有指定 scope，在引用 template 時指定好 scope 就可以解決此問題

```yaml
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}

# 若是這樣引用 template(沒有提供 scope)，global chart value 的地方就會空白
{{- template "mychart.labels" }}

# 正確的引用方式("." 即為 scope)
{{- template "mychart.labels" . }}

# 也可以使用 "$" 作為 scope，其實效果跟 "." 是相同的
{{- template "mychart.labels" $ }}
```


## 使用 include 來放入 template 資訊

- 若是 template 被引用是在 chart 中各種不同的位置，甚至是其他 chart 時，縮排(indent)的處理就至關重要，光用 `template` 關鍵字引用 template 可能會遇到 indent 錯誤的問題，此時就是使用 `include` 來解決問題
> 基本上，`template` & `include` 功能是相同的，都是用來引用透過 `define` 定義的 template 

- 僅有 `include` 可以搭配 `indent` 使用，用法範例為 `{{ include "mychart.version" . | indent 4 }}`


## Notes

透過撰寫 `NOTES.txt` 檔案的內容，可以在使用者透過 chart 佈署特定資源後，告知使用者如何連線到此服務、如何使用此服務之類的訊息；對於提供使用者如何正確的使用服務有很大的幫助。


## sub chart

- sub chart 顧名思義就是在一個 Helm chart 中再包一個 Helm chart 的概念

- 每一個 sub chart 可以定義自己的 values，但 parent chart 中可以 override sub chart 中設定的值

- 在 parent chart 中可以定義 Global variable，讓 parent chart & sub chart 都可以使用
> 使用關鍵字 `global`，並放在 root level


Repository Management
=====================

## Overview

基本上，最簡單的 Helm Chart Repository，只要符合兩個條件即可：

1. 支援 HTTP(S) 協定

2. 提供 Helm Chart package & 對應的 index 檔案(`index.yaml`) 即可

目前在自架方案中，比較常用的 solution 是 [ChartMuseum](https://chartmuseum.com/)

但由於 Helm CLI 有 plugin 的機制，這使得也可以使用像是 AWS S3 作為 Helm Chart Repository (可參考䅮面的常用指令操作)

另外 Helm CLI 也支援使用提供 OCI(Open Container Initiative) 標準的 Helm Chart Repository，因此像是 [AWS ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/push-oci-artifact.html) 就可以作為 Helm Chart Repository 其中一個選項之一


## GitHub as Helm Repository

- 由於 GitHub Repository 本身並不是設定用來存放 Helm Chart，因此沒有相關功能，所以用來表示 Helm Chart 的 index file `index.yaml` 就必須自行產生後再 push 到 GitHub

```bash
# 可以根據目前目錄下的 helm package 產生 index.yaml
$ helm repo index .
```

- 若相關檔案已經正確上傳到 GitHub Repository，就可以透過類似下面格式用來加入 Helm Chart Repo 並使用
> https://raw.githubusercontent.com/[USER_NAME]/[REPO_NAME]/[BRANCH_NAME]


Helm Chart Hook
===============

Helm Chart 有 [Chart Hooks](https://helm.sh/docs/topics/charts_hooks/) 的機制設計，可以在 Helm Chart 的生命週期各階段根據需求執行不同的 hook job，從官網的說明中，有以下幾個 Chart Hook 可使用：

- `pre-install`: Executes after templates are rendered, but before any resources are created in Kubernetes

- `post-install`: Executes after all resources are loaded into Kubernetes

- `pre-delete`: Executes on a deletion request before any resources are deleted from Kubernetes

- `post-delete`: Executes on a deletion request after all of the release's resources have been deleted

- `pre-upgrade`: Executes on an upgrade request after templates are rendered, but before any resources are updated

- `post-upgrade`: Executes on an upgrade request after all resources have been upgraded

- `pre-rollback`: Executes on a rollback request after templates are rendered, but before any resources are rolled back

- `post-rollback`: Executes on a rollback request after all resources have been modified

- `test`: Executes when the Helm test subcommand is invoked ( view test docs)

撰寫方式其實也很簡單，只要將要執行的 Hook Job 加上 Annotations 的內容即可，範例如下：(以下是個 **post-install** 的範例)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: "{{ .Release.Name }}"
  labels:
    app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
    app.kubernetes.io/instance: {{ .Release.Name | quote }}
    app.kubernetes.io/version: {{ .Chart.AppVersion }}
    helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
  annotations:
    "helm.sh/hook": post-install
    # weight 可用來設定執行順序(預設為 0)，越小越優先，但必須以 string 指定
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: "{{ .Release.Name }}"
      labels:
        app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
        app.kubernetes.io/instance: {{ .Release.Name | quote }}
        helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
    spec:
      restartPolicy: Never
      containers:
      - name: post-install-job
        image: "alpine:3.3"
        command: ["/bin/sleep","{{ default "10" .Values.sleepyTime }}"]
```

若是同一個 job 同時在不同的 Chart Hook 中需要，也可以這樣寫：

```yaml
annotations:
  "helm.sh/hook": post-install,post-upgrade
```

## 與 ArgoCD 整合

若是環境中有使用 ArgoCD 搭配 Helm，需要注意的是，並非所有 Chart Hook 都有支援，因此若是有利用到 Chart Hook 時，就需要留意一下。

相關的詳細說明可以參考 [ArgoCD 文件說明](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#helm-hooks)。


參考資料
=======

- [HELM - Package Manager for Kubernetes Complete Master Course | Udemy](https://www.udemy.com/course/helm-package-manager-for-kubernetes-complete-master-course/)

- [hypnoglow/helm-s3: Helm plugin that allows to set up a chart repository in AWS S3.](https://github.com/hypnoglow/helm-s3)

- [Set up a Helm v3 chart repository in Amazon S3 - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/set-up-a-helm-v3-chart-repository-in-amazon-s3.html)

- [https://helm.sh/docs/topics/charts_hooks/](https://helm.sh/docs/topics/charts_hooks/)

- [Helm - Argo CD - Declarative GitOps CD for Kubernetes](https://argo-cd.readthedocs.io/en/stable/user-guide/helm)