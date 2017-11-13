---
title:  "[OpenShift] Concept - Image Stream"
date: 2017-11-06 14:00:00
comments: true
tags: 
  - OpenShift
categories: 
  - OpenShift
---

What is Image Stream?
=====================

每一個 image stream 代表著一個 Docker-formatted container image；它其實只是一個在 OpenShift 中內部對於 docker image 的命名方式，讓系統可以使用指定的名稱找到正確的 docker image 來使用。(類似 Docker 中對每個 image 使用 tag 來命名)

也因為有自己內部的命名方式，因此 Image Stream 就可以包含以下來源的 image：

- OpenShift 內部的 container registry

- 其他的 image stream

- 外部的 image repository (例如：DockerHub, CoreOS Quay)

在 OpenShift 中，image stream 可與 Build & Deployment 搭配完成特定的自動化功能；由於 Build & Deployment 都可以監控特定 image stream，當 image stream 指向的 image 有新版產生時，可自動的進行特定的 build or deploy 的工作。


----------------------------


建立第一個 Image Stream
=====================

以下是一個 ImageStream 的定義範例：

```yml
---
kind: ImageStream
apiVersion: v1
metadata:
  name: "myubuntu:xenial"
spec:
  tags:
  - name: '16.04'
    from:
      kind: DockerImage
      name: ubuntu
    
```

透過以上的 image stream 定義，在 OpenShift Build or Deployment 中，就可以使用 `myubuntu:xenial` 指定外部 DockerHub 中的 `ubuntu:16.04` image。

當以上 ImageStream 被建立後，我們可以查詢到以下資訊：

```bash
$ oc get is
NAME       DOCKER REPO                                           TAGS      UPDATED
myubuntu   docker-registry.default.svc:5000/leon-test/myubuntu   16.04     2 seconds ago
```

若使用 `oc get is/myubuntu -o=yaml` 指令檢視 YAML 輸出，得到以下內容：

```yaml
apiVersion: v1
kind: ImageStream
metadata:
  annotations:
    openshift.io/image.dockerRepositoryCheck: 2017-11-06T04:12:59Z
  creationTimestamp: 2017-11-06T04:12:58Z
  generation: 2
  name: myubuntu
  namespace: leon-test
  resourceVersion: "9145705"
  selfLink: /oapi/v1/namespaces/leon-test/imagestreams/myubuntu
  uid: c59725ce-c2a8-11e7-a13b-faf564e56811
spec:
  lookupPolicy:
    local: false
  tags:
  - annotations: null
    from:
      kind: DockerImage
      name: ubuntu
    generation: 2
    importPolicy: {}
    name: "16.04"
    referencePolicy:
      type: Source
status:
  dockerImageRepository: docker-registry.default.svc:5000/leon-test/myubuntu
  tags:
  - items:
    - created: 2017-11-06T04:12:59Z
      dockerImageReference: ubuntu@sha256:152b4ccc429f6f28533aff625d8345baf1ba3808e9a99446e86b2bf3efa18571
      generation: 2
      image: sha256:152b4ccc429f6f28533aff625d8345baf1ba3808e9a99446e86b2bf3efa18571
    tag: "16.04"
```


----------------------------


Image Stream Image
==================

image stream image(簡稱 **isimage**) 是一種 virtual resource，讓使用者可以透過 isimage 從特定的 image stream 取得 image，isimage 以 `<image stream name>@<image name>` 的方式呈現，因此以上面的範例來看，image steam image 就會是：

> myubuntu@sha256:152b4ccc429f6f28533aff625d8345baf1ba3808e9a99446e86b2bf3efa18571

----------------------------


Image Stream Tag
================

image stream tag(簡稱 **istag**) 是一個指到上面 image stream image 的 name pointer，可指向 local or 外部的 image，此外 isiage 還包含了 image 內容變動的歷史紀錄，這樣的設計讓使用者可以在有需要的時候方便的進行 rollback。

istag 以 `<image stream name>:<tag>` 的方式呈現，因此以上面的範例來看， istag 就會是：

>　myubuntu:16.04

----------------------------


Conclusion
==========

有了 Image Stream(is), Image Stream Image (isimage), 以及 Image Stream Tag(istag) 的觀念之後，下一個階段將會介紹如何在 OpenShift 中管理 Image。


----------------------------


References
==========

- [Builds and Image Streams - Core Concepts \| Architecture \| OpenShift Container Platform 3.6](https://docs.openshift.com/container-platform/3.6/architecture/core_concepts/builds_and_image_streams.html)

- [Builds and Image Streams - Core Concepts \| Architecture \| OpenShift Enterprise 3.0](https://docs.openshift.com/enterprise/3.0/architecture/core_concepts/builds_and_image_streams.html)

- [OpenShift ImageStream Examples @GitHub](https://github.com/openshift/origin/tree/master/examples/image-streams)

- [Managing Images \| Developer Guide \| OpenShift Container Platform 3.6](https://docs.openshift.com/container-platform/3.6/dev_guide/managing_images.html)