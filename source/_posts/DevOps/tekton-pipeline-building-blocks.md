---
layout: post
title:  "[Tekton] Tekton Pipeline 的基本組成元件介紹"
description: "此篇文章介紹 Tekton pipeline 的基本組成元件，包含 PipelineResource, Task, TaskRun, Pipeline, PipelineRun ... 等等，並包含一些簡單的使用範例"
date: 2019-10-03 16:15:00
published: true
comments: true
categories:
  - DevOps
tags:
  - DevOps
  - Tekton
  - Kubernetes
---

Overview
========

在 Tekton pipeline 中有以下幾個主要的組成要素，分別是：

- PipelineResource

- Task & ClusterTask

- TaskRun

- Pipeline

- PipelineRun


PipelineResource
================

`PipelineResource` 可以作為 task 的 input or output，而每個 task 可以有多個 input & output。

## Syntax

PipelineResource 的定義中會有以下必要資訊：

- `apiVersion`：目前固定是 `tekton.dev/v1alpha1`

- `kind`：因為這是 CRD，所以是 `PipelineResource`

- `metadata`：用來辨識此 TaskRun 用的資訊，例如 **name**

- `sepc`：使用 resource 的詳細資訊(例如：路徑、位址)

- `type`：用來指定 resource type，目前支援 `git`, `pullRequest`, `image`, `cluster`, `storage`, `cloudevent` ... 等等

其他選填項目：

- `params`：不同的 resource type 可能會有的不同額外參數資訊


## Resource Type

有了以上概念後，接著要知道的是 [PipelineResources][PipelineResources] 共有以下幾種類型：

- Git Resource

- Pull Request Resource

- Image Resource

- Cluster Resource

- Storage Resource

- Cloud Event Resource

以下就針對比較常用的 Git & Image resource 說明，其他的部份可以參考[官網的詳細文件](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md)。


### Git Resource

一般的 git repository，作為 task input 時，Tekton 執行 task 前會將程式碼 clone 回來，因此這邊就必須注意 git repository 存取的權限問題，若是 private repository 就要額外提供 credential 資訊才可以正常運作

以下是一個標準的 Git PipelineResource 的定義：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-git
  namespace: default
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang.git
    # 可用 branch, tag, commit SHA or ref
    # 沒指定就會拉 master branch
    - name: revision
      value: master
      # value: some_awesome_feature
      # value: refs/pull/52525/head
```

### Image Resource

image resource 就是存放在某個遠端 image registry 的 image，通常作為 task output，表示該 task 過程中會 build image 並上傳到指定的 image resource。

以下是一個標準 Image PipielineResource 的定義：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: kritis-resources-image
  namespace: default
spec:
  type: image
  params:
    - name: url
      value: gcr.io/staging-images/kritis
```


## 使用上的一些小技巧

### 調整 resource 掛載的路徑

實際上工作執行在 container 中，使用 resource 的方法就是會先將其掛載到 container 的檔案系統中。

使用者可以透過 `targetPath` 參數來調整 resource 被掛載的路徑，在預設沒有設定 `targetPath` 的情況下，resource 會被掛載到 `/workspace` 目錄中，若是有指定則會改到 `/workspace/targetPath` 目錄中，以下是一個設定 targetPath 的範例：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: task-with-input
  namespace: default
spec:
  inputs:
    resources:
      - name: workspace
        type: git
        # 額外指定了 targetPath，會影響實際上在 container 的掛載路徑
        targetPath: go/src/github.com/tektoncd/pipeline
  steps:
    - name: unit-tests
      image: golang
      command: ["go"]
      args:
        - "test"
        - "./..."
      # 因為設定了 targetPath，因此 workingDir 也要有對應的設定才可以正確執行
      workingDir: "/workspace/go/src/github.com/tektoncd/pipeline"
      env:
        - name: GOPATH
          value: /workspace/go
```

### 在 pipeline 的 task 之間傳遞 resource

這樣的應用可以透過 `paths` 設定來達成：

- 若是在 input resource 設定 paths，表示要改從 paths 指定的路徑取得 resource 內容

- 若是在 output resource 設定 paths，表示要將 resource 複製到 paths 所指定的路徑

通常 `paths` 的設定常用在 Pipeline 中 Task 之間的 resource 傳遞，以下是個簡單範例：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: volume-task
  namespace: default
spec:
  inputs:
    resources:
      - name: workspace
        type: git
  # 原本是要 push 回 git resource
  outputs:
    resources:
      - name: workspace
  steps:
    - name: build-war
      image: objectuser/run-java-jar
      command: jar
      args: ["-cvf", "projectname.war", "*"]
      volumeMounts:
        - name: custom-volume
          mountPath: /custom

---
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: volume-taskrun
  namespace: default
spec:
  taskRef:
    name: volume-task
  inputs:
    resources:
      - name: workspace
        resourceRef:
          name: java-git-resource
  outputs:
    resources:
      - name: workspace
        # 在 output 設定 paths，會將 output resource 複製到以下路徑中
        # 所以 output resource 會存在於 volume 上
        # 而不是被送回 Git repository
        paths:
          - /custom/workspace/
        resourceRef:
          name: java-git-resource
  # 額外定義一個 volume
  volumes:
    - name: custom-volume
      emptyDir: {}
```


Task & ClusterTask
==================

`Task`(& `ClusterTask`) 中包含了一連串的 step，通常是使用者要用來執行 CI flow，而這些工作會在單一個 pod 中以多個 container 的形式逐一完成。

> Task & ClusterTask 兩者的不同在於 Task 是屬於 namespace level，而 ClusterTask 是屬於 cluster level

而在 Task 的定義中，最重要的部份有以下三個項目：

- Input

- Output

- Steps

以下是一個 task 的標準定義內容：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: deploy-using-kubectl
spec:
  inputs:
    resources:
      - name: source
        type: git
      - name: image
        type: image
    params:
      - name: path
        type: string
        description: Path to the manifest to apply
      - name: yamlPathToImage
        type: string
        description:
          The path to the image to replace in the yaml manifest (arg to yq)
  steps:
    # step 中可以定義多個執行工作，會依照順序執行
    - name: replace-image
      image: mikefarah/yq
      command: ["yq"]
      args:
        - "w"
        - "-i"
        - "$(inputs.params.path)"
        - "$(inputs.params.yamlPathToImage)"
        - "$(inputs.resources.image.url)"
    - name: run-kubectl
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(inputs.params.path)"
  # 此 volume 在 task 中沒有用到，只是一個範例而已
  # 用以表示可以在 task 中定義 volume 並使用
  volumes:
    - name: example-volume
      emptyDir: {}
```

以下接著說明上面提到的 **Input** / **Output** / **Steps**

## Input

若是有需要，可以在 task 中定義 `input`，而 input 可以有 `Parameters` & `Input resources` 兩種：

### Parameters

這個部份可以當作是簡單的字串類型參數，可以拿來作為程式編譯時的參數 or 用來命名 artifacts 之用；也可以使用 *array** 之類較為複雜的參數。

以下是一個簡單 Task 的範例：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: task-with-parameters
spec:
  inputs:
    params:
      - name: flags
        type: array
      - name: someURL
        type: string
  steps:
    - name: build
      image: my-builder
      args: ["build", "$(inputs.params.flags)", "url=$(inputs.params.someURL)"]
```

透過以上的 Task template，可以用以下的 TaskRun 來使用：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: run-with-parameters
spec:
  taskRef:
    name: task-with-parameters
  inputs:
    params:
      - name: flags
        value: 
          - "--set"
          - "arg1=foo"
          - "--randomflag"
          - "--someotherflag"
      - name: someURL
        value: "http://google.com"
```

### Input resources 

這裡的 input resource 即為上面所介紹的 [PipelineResources][PipelineResources]。

上面有提到如何定義 PipelineResources，以下是個簡單在 Task 中使用 PipelineResources 的範例：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: deploy-image
  namespace: default
spec:
  # 以下使用到 Git, Image, Cluster 三種 PipelineResource
  inputs:
    resources:
      - name: workspace
        type: git
      - name: dockerimage
        type: image
      - name: testcluster
        type: cluster
  steps:
    - name: deploy
      image: image-wtih-kubectl
      command: ["bash"]
      args:
        - "-c"
        - kubectl --kubeconfig
          /workspace/$(inputs.resources.testCluster.name)/kubeconfig --context
          $(inputs.resources.testCluster.name) apply -f /workspace/service.yaml'
```


## Output

基本上 `Output` 會以 [PipelineResources][PipelineResources] 的形式存在，在使用上有一些必須注意到的部份：

- 在一個 Pipeline 中的 output 定義，output resource 會以共享的形式存在於多個 task 之間，並放在 `/workspace/output/resource_name/` 路徑下

- 由於每個 task 的工作都會在獨立的 pod 完成，因此共享的形式想當然爾就是用 PV & PVC 的方式來達成

- 承上，所以沒有額外 storage 相關設定的情況下，使用者必須確認 k8s cluster 中有預設的 StorageClass 可用

以下是一個簡單的範例：

```yaml
resources:
  outputs:
    name: storage-gcs
    type: gcs
steps:
  - image: objectuser/run-java-jar
    command: [jar]
    args:
      ["-cvf", "-o", "/workspace/output/storage-gcs/", "projectname.war", "*"]
    env:
      - name: "FOO"
        value: "world"
```

上面的工作完成後，檔案會自動被上傳到 `storage-gcs` resource 指定的 GCS 上。

以下的示範則是在一個 task 中的多個 step 之間對 output resource 進行處理：

```yaml
resources:
  inputs:
    # 指定 input resource 掛載到 /workspace/customworkspace 目錄中
    name: tar-artifact
    targetPath: customworkspace
  # input & output 相同
  # 因此處理完後會將 /workspace/customworkspace 目錄中的檔案上傳到 resource 指定的位置
  outputs:
    name: tar-artifact
steps:
  # 解壓縮來自 input 的檔案(rules_docker-master.tar)到 /workspace/tar-scratch-space/ 目錄中
  - name: untar
      image: ubuntu
      command: ["/bin/bash"]
      args: ['-c', 'mkdir -p /workspace/tar-scratch-space/ && tar -xvf /workspace/customworkspace/rules_docker-master.tar -C /workspace/tar-scratch-space/']
  # 對壓縮檔中的某個檔案進行編輯
  - name: edit-tar
      image: ubuntu
      command: ["/bin/bash"]
      args: ['-c', 'echo crazy > /workspace/tar-scratch-space/rules_docker-master/crazy.txt']
  # 將檔案重新打包並放回 /workspace/customworkspace 目錄中
  - name: tar-it-up
    image: ubuntu
    command: ["/bin/bash"]
    args: ['-c', 'cd /workspace/tar-scratch-space/ && tar -cvf /workspace/customworkspace/rules_docker-master.tar rules_docker-master']
```


## Steps

在 task 中 `steps` 的項目，會依照以下方式運作：

- 可以定義多個項目，都會成為一個獨立的 container 執行

- 同一個 task 中的 steps 定義的項目所對應到的 container 都會存在同一個 pod 內

- 會依照 steps 中定義的順序執行

而 step 的使用方式就像是在 k8s 中 [container](https://kubernetes.io/docs/concepts/containers/) 的定義一樣，從上面 input & output 的例子多少可以看出，以下再來一個簡單的範例：

```yaml
stepTemplate:
  env:
    - name: "FOO"
      value: "bar"
steps:
  - image: ubuntu
    command:
    - env
  - image: ubuntu
    command:
    - - env
    env:
      - name: "FOO"
        value: "baz
```


TaskRun
=======

定義了 task 之後，Tekton 並不會主動執行任何 task，這時候就必須要搭配 TaskRun 才可以讓 task 真正的執行指定工作。

## Syntax

TaskRun 的定義中會有以下必要資訊：

- `apiVersion`：目前固定是 `tekton.dev/v1alpha1`

- `kind`：因為這是 CRD，所以是 `TaskRun`

- `metadata`：用來辨識此 TaskRun 用的資訊，例如 **name**

- `sepc`：一般會放上 `taskRef`，指定已經定義好的 Task

其他選填資訊：

- `serviceAccount`: 讓使用者可以使用自訂的認證資訊來執行工作

- `input`：用來指定 input parameter or resource

- `output`：用來指定 output resource

- `timeout`：指定工作執行的 timeout 時間，若設定為 0 則表示沒有 timeout (預設的 timeout 時間為 `60 mins`)

- `podTemplate`：所有的 TaskRun 都是以 Pod 的形式執行，此參數可以在 PodSpec 中額外加入指定的 subset 資訊

以下是一個標準的 TaskRun 定義：

```yaml
# 以下幾個(apiVersion, kind, metadata, spec)是必要資訊
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: build-docker-image-from-git-source-task-run
spec:
  serviceAccount: robot-docker-basic
  # 指定到已經預先定義好的 Task
  taskRef:
    name: build-docker-image-from-git-source
  inputs:
    resources:
      - name: docker-source
        # 指定到已經預先定義好的 PipelineResource
        resourceRef:
          name: git-tekton-test
    params:
      - name: pathToDockerFile
        value: Dockerfile
      - name: pathToContext
        value: /workspace/docker-source/examples/microservices/leeroy-web
  outputs:
    resources:
      - name: builtImage
        resourceRef:
          name: image-tekton-test
```

## Pod Template

透過 Pod Template 的設定，可以提供額外的資訊讓 Task(Pod) 執行時改變某些行為，目前支援的設定如下：

- `nodeSelector`：可用來指定 task 要在哪個 node 上執行

- `tolerations`：允許讓 task 執行上帶有特定 **taint** 設定的 node 上

- `affinity`：可限制 task 在帶有某些 label 的 node 上執行

- `securityContext`：額外指定 pod-level 的安全屬性，或像是 `runAsUser` or `selinux` 的 container 設定

- `volumes`：指定可讓 container 掛載的 volume 清單

以下是一個 pod template 的範例：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: myTask
  namespace: default
spec:
  steps:
    - name: write something
      image: ubuntu
      command: ["bash", "-c"]
      args: ["echo 'foo' > /my-cache/bar"]
      volumeMounts:
        - name: my-cache
          mountPath: /my-cache
---
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: myTaskRun
  namespace: default
spec:
  taskRef:
    name: myTask
  podTemplate:
    # 額外指定 securityContext
    securityContext:
      runAsNonRoot: true
    # 額外指定 volume
    volumes:
    - name: my-cache
      persistentVolumeClaim:
        claimName: my-volume-claim
```


Pipeline
========

Pipeline 其實可以把它簡單思考為前面 Task 的集合，有順序性的排列，並透過之後介紹的 PipelineRun 來運作。

## Syntax

Pipeline 的定義中會有以下必要資訊：

- `apiVersion`：目前固定是 `tekton.dev/v1alpha1`

- `kind`：因為這是 CRD，所以是 `Pipeline`

- `metadata`：用來辨識此 TaskRun 用的資訊，例如 **name**

- `sepc`：這裡就會透過 `tasks` 來指定已經定義好的 Task list 了

其他選填項目：

- `resources`：設定在 Task 中會使用到的 PipelineResource

- `tasks.conditions`：用來設定在特定的情況下 Task 才需要執行，詳細內容可參考[官方文件](https://github.com/tektoncd/pipeline/blob/master/docs/conditions.md)


## 定義 Tasks & Resource & Parameter

在 Task 也有 Resource & Parameter 的定義，Input 中可以同時有 Resource & Parameter，但 Output 中僅可以有 Resource。

那 Pipeline 中的 Resource 就不一樣了，沒有 Input & Output 的差別，就僅僅是 Resource & Parameter 的定義而已，因為實際上需要 Input & Output 的是 Task 本身，Pipeline 中的 Resource & Parameter 定義只是用來滿足 Task 的需要而已，以下是一個簡單範例：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: list-files-pipeline
spec:
  # 即為 PipelineResource
  resources:
    - name: source-repo
      type: git
  params:
    - name: "path"
      default: "README.md"
```

而 Parameter 在使用上還有一些額外需要了解的：

- 每個 Parameter 都需要指定 `type` 資訊，沒指定就預設為 string

- 若需要同時傳入多個 string，則可以將 type 設定為 `array`

以下是一個 Pipeline & PipelineRun 的組合設定：

```yaml
---
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: pipeline-with-parameters
spec:
  # parameter 定義
  params:
    - name: context
      type: string
      description: Path to context
      default: /some/where/or/other
  # 在 pipeline 中要執行的 task
  tasks:
    - name: build-skaffold-web
      # 指到之前定義好的 task
      taskRef:
        name: build-push
      params:
        - name: pathToDockerFile
          value: Dockerfile
        - name: pathToContext
          value: "$(params.context)"

---
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: pipelinerun-with-parameters
spec:
  pipelineRef:
    name: pipeline-with-parameters
  # 根據實際狀況，調整 parameter 預設值
  params:
    - name: "context"
      value: "/workspace/examples/microservices/leeroy-web"
```

## from / runAfter / retries / conditions  關鍵字的使用

### from

有時候我們需要在不同的 task 之間傳遞資訊，這時候就必須使用 `from`，以下是一個範例：

```yaml
# 執行順序："build-app" -> "deploy-app"
- name: build-app
  taskRef:
    name: build-push
  resources:
    outputs:
      - name: image
        resource: my-image
- name: deploy-app
  taskRef:
    name: deploy-kubectl
  resources:
    inputs:
      - name: image
        resource: my-image
        # from 的設定通常會指到上一個 Task
        # 而被指到的 Task 會有一個 output 來對應這個 input
        from:
          - build-app
```

> 由於在 `deploy-app` 有設定 **from** 指到 `build-app`，因此無論實際上設定檔如何定義順序，執行時都會先以 `build-app -> deploy-app` 的順序來執行

### runAfter

有時候 Task 之間沒有上到下的 input & output 關係，但又要明確控制執行順序時，可使用 `runAfter`關鍵字，以下是一個簡單範例：

```yaml
# 執行順序："test-app" -> "build-app"
- name: test-app
  taskRef:
    name: make-test
  resources:
    inputs:
      - name: workspace
        resource: my-repo
- name: build-app
  taskRef:
    name: kaniko-build
  # 確保此 task 會在 "test-app" 結束後再執行
  runAfter:
    - test-app
  resources:
    inputs:
      - name: workspace
        resource: my-repo
```

> 由於 runAfter 會明確定義 Task 執行順序，因此以上面的範例來看，即使 "build-app" 擺在 "test-app" 之前也不會改變執行順序

### retries

有時候可能會因為網路問題，或是其他因素造成 Task 無法正確執行，但可能過一陣子就會正常；在這樣的情況下，就會有 Task retry 的需求，以下是設定範例：

```yaml
tasks:
  - name: build-the-image
    # 此 Task 最多會嘗試執行兩次，兩次都失敗就算失敗
    retries: 1
    taskRef:
      name: build-push
```

### conditions

有時候某些 Task 只有在某些條件滿足的情況下才需要執行，這時候就需要 [conditions](https://github.com/tektoncd/pipeline/blob/master/docs/conditions.md) 的設定。

基本上，conditions 也是一個 CRD，以下是一個 condition 的定義範例：

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Condition
metadata:
  name: file-exists
spec:
  params:
    - name: "path"
      type: string
  resources:
    - name: workspace
      type: git
  # condition 中要進行的檢查
  check:
    image: alpine
    command: ["/bin/sh"]
    args: ['-c', 'test -f $(resources.workspace.path)/$(params.path)']

---
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: list-files-pipeline
spec:
  resources:
    - name: source-repo
      type: git
  params:
    - name: "path"
      default: "README.md"
  tasks:
    - name: list-files-1
      taskRef:
        name: list-files
      conditions:
        # 要滿足以下條件才會執行此 Task
        - conditionRef: "file-exists"
          params:
            - name: "path"
              value: "$(params.path)"
          resources:
            - name: workspace
              resource: source-repo
      resources:
        inputs:
          - name: workspace
            resource: source-repo
```

## Ordering

目前可用來控制 Task 運作順序的，就是 `from` & `runAfter` 兩種設定：

- `from` 用來指定從上一個 Task output resource 作為目前 Task 的 input resource

- `runAfter` 用來指定目前的 Task 在某個 Task 執行完成後才開始

透過以上兩種設定，可以將 Pipeline 設計成 DAG(Directed Acyclic Graph) 的流程，以下是官網範例：

```yaml
# 與 test-app 同時執行
- name: lint-repo
  taskRef:
    name: pylint
  resources:
    inputs:
      - name: workspace
        resource: my-repo
# 與 lint-repo 同時執行
- name: test-app
  taskRef:
    name: make-test
  resources:
    inputs:
      - name: workspace
        resource: my-repo
# 在 test-app 完成後開始執行
- name: build-app
  taskRef:
    name: kaniko-build-app
  runAfter:
    - test-app
  resources:
    inputs:
      - name: workspace
        resource: my-repo
    outputs:
      - name: image
        resource: my-app-image
# 在 test-app 完成後開始執行
- name: build-frontend
  taskRef:
    name: kaniko-build-frontend
  runAfter:
    - test-app
  resources:
    inputs:
      - name: workspace
        resource: my-repo
    outputs:
      - name: image
        resource: my-frontend-image
# 當 build-app & build-frontend 都完成後才會執行
- name: deploy-all
  taskRef:
    name: deploy-kubectl
  resources:
    inputs:
      - name: my-app-image
        resource: my-app-image
        from:
          - build-app
      - name: my-frontend-image
        resource: my-frontend-image
        from:
          - build-frontend
```

上面的 pipeline 設定會變成以下流程：

```txt
        |            |
        v            v
     test-app    lint-repo
    /        \
   v          v
build-app  build-frontend
   \          /
    v        v
    deploy-all
```

而上面的 Pipeline 設定，要 `lint-repo` & `deploy-all` 兩個都執行完後，才算全部完成


PipelineRuns
============

在 Pipeline 中定義了需要執行哪些 Task & 執行順序，但是要讓工作實際上執行起來，就必須要建立 PipelineRun。

## Syntax

Pipeline 的定義中會有以下必要資訊：

- `apiVersion`：目前固定是 `tekton.dev/v1alpha1`

- `kind`：因為這是 CRD，所以是 `PipelineRun`

- `metadata`：用來辨識此 TaskRun 用的資訊，例如 **name**

- `sepc`：這裡就會透過 `pipelineRef` 來指定已經定義好的 Pipeline 了 (也可以使用 `pipelineSpec` 並包含 Pipeline 的設定，但不建議，因為這樣做就無法 reuse)

其他選填項目：

- `resources`：設定在 PipelineRun 中會使用到的 PipelineResource

- `serviceAccount`: 讓使用者可以使用自訂的認證資訊來執行工作

- `serviceAccounts`：可以定義多組 PipelineTask + ServiceAccount 的組合，來設定每個 Task 使用不同的 service account 來執行

- `tasks.conditions`：用來設定在特定的情況下 Task 才需要執行，詳細內容可參考[官方文件](https://github.com/tektoncd/pipeline/blob/master/docs/conditions.md)


以下是一個 PipelineRun 的實際範例：(完整程式碼可以參考[官網範例](https://github.com/tektoncd/pipeline/blob/master/examples/pipelineruns/pipelinerun.yaml))

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: demo-pipeline-run-1
spec:
  # 指定 pipeline
  pipelineRef:
    name: demo-pipeline
  # 指定執行工作時用的 service account
  serviceAccount: 'default'
  # 這段設定是額外加的，並不在原本的 source code 內
  # 用來表示 Task "build-push" 要用 service account "sa-for-build" 的身份執行
  serviceAccounts:
    - taskName: build-push
      serviceAccount: sa-for-build
  # 指定 pipeline resources
  resources:
  - name: source-repo
    resourceRef:
      name: skaffold-git
  - name: web-image
    resourceRef:
      name: skaffold-image-leeroy-web
  - name: app-image
    resourceRef:
      name: skaffold-image-leeroy-app
```


結語
===

以上內容(PipelineResource, Task, TaskRun, Pipeline, PipelineRun) 是 Tekton 中執行工作的必要元素，實際上執行的 CI/CD 工作都會與這幾個部份有關。

Tekton 將所有的基本元素拆分成一個一個的 k8s CRD(Custom Resource Definition)，如果是稍微複雜一點的 CI/CD 工作，可能就會需要定義不少個 CRD 才能完成，而且在設計上相對於其他的 CI server(例如：GitLab CI, Drone CI)可能不是這麼直覺；但這樣的設計提供了以下優點：

1. 原生的 k8s 使用經驗，不需要額外學習其他語法

2. 定義好的 CRD(PipelineResource, Task, Pipeline) 可以被重複利用

3. 原生整合 k8s

若是未來有考慮 workload 都跑在 k8s 上的使用者，在選擇 CI/CD 的工具時或許可以將 Tekton 考慮進行。

接著可能會面臨到的問題可能是，如果希望作到 GitOps，光是以上項目好像不夠，因為要執行工作，目前看起來似乎都是需要手動的；因此接下來要處理的問題就是，如何在使用者進行了某些行為時，自動觸發產生 PipelineRun 來執行 CI/CD 工作，這還需要額外加上一些 event trigger 的設定才有辦法完成，而這個部份將會由 [Tekton Trigger](https://github.com/tektoncd/triggers) 來進行，未來有空的話會繼續探討這個部份。


References
==========

- [pipeline/examples at master · tektoncd/pipeline](https://github.com/tektoncd/pipeline/tree/master/examples)

- [pipeline/resources.md at master · tektoncd/pipeline](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md)

- [pipeline/tasks.md at master · tektoncd/pipeline](https://github.com/tektoncd/pipeline/blob/master/docs/tasks.md)

- [Tekton - Task Examples](https://github.com/tektoncd/pipeline/blob/master/docs/tasks.md#examples)

- [pipeline/taskruns.md at master · tektoncd/pipeline](https://github.com/tektoncd/pipeline/blob/master/docs/taskruns.md)

- [Tekton - TaskRun Examples](https://github.com/tektoncd/pipeline/tree/master/examples/taskruns)

- [pipeline/pipelines.md at master · tektoncd/pipeline](https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md)


## Trigger

- [tektoncd/triggers: Event triggering with Tekton!](https://github.com/tektoncd/triggers)


[PipelineResources]: https://github.com/tektoncd/pipeline/blob/master/docs/resources.md "PipelineResources"