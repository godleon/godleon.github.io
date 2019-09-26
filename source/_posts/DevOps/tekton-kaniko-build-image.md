---
layout: post
title:  "[Tekton] 使用 Tekton + Kanibo 在 Kubernetes 上產生 container image"
description: "此篇文章介紹如何透過 Tekton + Kanibo 在 Kubernetes 上產生 container image，並上傳到 Docker Hub 上"
date: 2019-09-26 15:50:00
published: true
comments: true
categories:
  - DevOps
tags:
  - DevOps
  - Tekton
  - Kanibo
  - Kubernetes
---

Ovewview
========

Tekton 是個 k8s 原生框架，用來建置 CI/CD 系統，以下是官方說明：

> Tekton is a powerful yet flexible Kubernetes-native open-source framework for creating continuous integration and delivery (CI/CD) systems. It lets you build, test, and deploy across multiple cloud providers or on-premises systems by abstracting away the underlying implementation details.

相關的中文介紹可以在網路上找到很多，這邊就不寫了，有興趣的人可以參考以下文章：

- [谷歌开源Tekton：Kubernetes原生的CI/CD构建框架-InfoQ](https://www.infoq.cn/article/tZ6E1_lhsWeh26C9xUJf)

- [kubernetes原生CI/CD工具：Tekton探秘与上手实践 - 掘金](https://juejin.im/post/5d629c1a5188254628236b69)

這邊主要是因為未來要將 workload 往 k8s 上面移動，想說找看看跟 k8s 整合度比較好的新工具來用，就找到了 Tekton，而這工具也是 [Red Hat OpenShift](https://www.openshift.com/) 中標配的 CI/CD 工具。


此篇文章要介紹什麼?
================

這篇文章主要是介紹如何在 k8s 中，透過 [Tekton](https://github.com/tektoncd/pipeline) + [Kaniko](https://github.com/GoogleContainerTools/kaniko) 來 build container image 後，上傳到 Docker Hub 上。

原本要 build container image 需要有 root 或是跟 Docker daemon process 直接溝通的能力才有辦法，這樣的作法多少會有一些安全上的疑慮，因此 `Kaniko` 就是個可以讓使用者在 k8s 中，不需要什麼特別權限也可以 build container image 的工具。

有了以上需求後，就會衍生出一些問題：

- Kaniko 用來 build container image 的流程為何? 需要提供什麼資訊?

- 如何給入 Docker Hub credential


實作過程
=======

假設 k8s & [Tekton Pipelines](https://github.com/tektoncd/pipeline/blob/master/docs/install.md) & [Tekton dashboard](https://github.com/tektoncd/dashboard) 都已經已經佈署完成，繼續完成下面的步驟：

## 設定 Docker Hub credential

首先在本地端先用 `docker login` 在本地端成功登入 Docker Hub 後，會在家目錄中產生一個 docker config，完整路徑為 `~/.docker/config.json`，內容大概如下：

```json
{
	"auths": {
		"https://index.docker.io/v1": {
			"auth": "Z29...........................YzY="
		}
	}
}
```

此時可以用以下指令將 docker config 設定為 k8s secret：

> kubectl create secret generic docker-basic --from-file=.dockerconfigjson=~/.docker/config.json --type=kubernetes.io/dockerconfigjson

## 佈署 Tekton 相關資源

只要佈署以下的 YAML 就可以完成一個 build container image & push image to Docker Hub 的示範了：

以下是 ServiceAccount & Resource 的部份：

```yaml
---
# 需要一個 ServiceAccount 並帶上上面新增的 secret
apiVersion: v1
kind: ServiceAccount
metadata:
  name: robot-docker-basic
secrets:
  - name: docker-basic
imagePullSecrets:
  - name: docker-basic

---
# 設定 Tekton CRD "PipelineResource"
# 並指定其為特定的 Git repository
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git-tekton-test
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/GoogleContainerTools/skaffold


---
# 設定 Tekton CRD "PipelineResource"
# 指定要使用的 Docker Hub repository
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: image-tekton-test
spec:
  type: image
  params:
  - name: url
    value: godleon/tekton-test
```

以下是 Task & TaskRun 的 YAML 的定義：

```yaml
---
# 設定 Tekton CRD "Task"
# 定義 Task 的詳細步驟(step)，並包含 input, output & 相關參數
# 主要是定義 Kaniko build container image 的相關資訊
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  inputs:
    resources:
      - name: docker-source
        type: git
    params:
      - name: pathToDockerFile
        type: string
        description: The path to the dockerfile to build
        default: /workspace/docker-source/Dockerfile
      - name: pathToContext
        type: string
        description:
          The build context used by Kaniko
          (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
        # Kaniko executor image 會將 source code 預設放到 /workspace 目錄中
        default: /workspace/docker-source
  outputs:
    resources:
      - name: builtImage
        type: image
  steps:
    - name: build-and-push
      # 需要使用 Kaniko executor image
      image: gcr.io/kaniko-project/executor
      # 指定 Docker Hub credential 路徑
      # 若有指定帶有 Docker config secret 的 ServiceAccount
      # kaniko executor 會將 Docker config 放到 /builder/home/.docker/ 目錄下
      env:
        - name: "DOCKER_CONFIG"
          value: "/builder/home/.docker/"
      command:
        - /kaniko/executor
      # 使用 Kaniko 需要以下三個參數，實際值會從 TaskRun 傳過來
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(outputs.resources.builtImage.url)
        - --context=$(inputs.params.pathToContext)

---
# 設定 Tekton CRD "TaskRun"
# 這是以上面的 Task 為 template，傳入參數進行工作
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: build-docker-image-from-git-source-task-run
spec:
  serviceAccount: robot-docker-basic
  taskRef:
    name: build-docker-image-from-git-source
  inputs:
    resources:
      # 指到前面定義好的 Git repository
      # 其中的 name 會變成 /workspace 下的 source code root folder 名稱
      - name: docker-source
        resourceRef:
          name: git-tekton-test
    params:
      # 對應到上面 Task 中定義的參數
      - name: pathToDockerFile
        value: Dockerfile
      - name: pathToContext
        value: /workspace/docker-source/examples/microservices/leeroy-web
  outputs:
    resources:
      # 指到前面定義好的 docker image resource
      - name: builtImage
        resourceRef:
          name: image-tekton-test
```

將全部的 YAML 都套用後，就可以在 dashboard 上面看到運作的訊息：

![Tekton dashboard](/blog/images/devops/tekton-dashboard.png)

在 k8s 中 build 好的 container image 也都會送進 Docker Hub 囉!


References
==========

- [Tekton](https://github.com/tektoncd/pipeline)

- [Tekton Dashboard](https://github.com/tektoncd/dashboard)

- [Tektoncd usage and examples](https://sbr.pm/technical/tekton-usage.html)

- [Credential 問題的處理 - Creating a Secret](https://support.telefonicaopencloud.com/en-us/api/cce/en-us_topic_0035621944.html)

- [不需Root權限， Google釋出容器映像檔建立工具Kaniko | iThome](https://www.ithome.com.tw/news/122484)

- [Unable to use docker secrets to push image to dockerhub · Issue #1005 · tektoncd/pipeline](https://github.com/tektoncd/pipeline/issues/1005)

- [Tekton pipeline - Authentication](https://github.com/tektoncd/pipeline/blob/master/docs/auth.md)