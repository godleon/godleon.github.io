---
layout: post
title: "[OpenShift] 如何客製化 Jenkins Image"
description: "This article introduces how to customize OpenShift built-in Jenkins image"
date: 2017-11-07 15:20:00
comments: true
published: true
tags: 
  - OpenShift
  - Jenkins
categories: 
  - OpenShift
---

本文章的主題在於如何將客製化 OpenShift 原生提供的 Jenkins image


前言
===

Openshift 在 DevOps 的方面的確是下了不少功夫，使用 OpenShift 的確可以讓 DevOps engineer 省下不少工，但相對的也有蠻多新東西要學的! XD

在 OpenShift 建立第一個 BuildConfig(pipeline) 時，它會自動幫使用者佈署一個用來驅動 build pipeline 的 Jenkins server(此 container 會由 OpenShift 本身納管，因此掛了會自己再起一個新的)，佈署的行為如下：

1. 使用 `openshift` namespace 中的 template `jenkins-ephemeral` 為範本產生 Jenkins server

2. 此 tempate 使用的 image stream 為 `jenkins:latest`

3. Image Stream `jenkins:latest` 則是指到 Red Hat 官方所提供的 Jenkins image(**registry.access.redhat.com/openshift3/jenkins-2-rhel7**)

當 Jenkins server 佈署好之後，就會自動的與 OpenShift 本身的認証機制進行整合，完全不用使用者進行額外的設定，相關的 credential 也都會設定在 Jenkins server 內部，使用者需要做的就是開始設計 pipeline 就對了!


----------------------------

Jenkins image for OpenShift
===========================

由於 Jenkins 並不是一個很容易入門上手的系統，因此為了提供使用者能更方便的使用 Jenkins，OpenShift 將 Jenkins 系統打包成 docker image，並加入了許多常用的 plugin 以及與 OpenShift 整合所需要的 plugin，還很貼心的做好相關整合的設定，讓使用者在建立 pipeline 時就會同時產生相對應的 Jenkins server 來使用，省略許多繁複且容易造成錯誤的設定過程。

目前使用者可能比較容易用到的 Jenkins image for OpenShift 可能會有以下兩種：

- Red Hat 版本：`registry.access.redhat.com/openshift3/jenkins-2-rhel7`
本文章的主題在於如何將客製化 OpenShift 原生提供的 Jenkins image


前言
===

Openshift 在 DevOps 的方面的確是下了不少功夫，使用 OpenShift 的確可以讓 DevOps engineer 省下不少工，但相對的也有蠻多新東西要學的! XD

在 OpenShift 建立第一個 BuildConfig(pipeline) 時，它會自動幫使用者佈署一個用來驅動 build pipeline 的 Jenkins server(此 container 會由 OpenShift 本身納管，因此掛了會自己再起一個新的)，佈署的行為如下：

1. 使用 `openshift` namespace 中的 template `jenkins-ephemeral` 為範本產生 Jenkins server

2. 此 tempate 使用的 image stream 為 `jenkins:latest`

3. Image Stream `jenkins:latest` 則是指到 Red Hat 官方所提供的 Jenkins image(**registry.access.redhat.com/openshift3/jenkins-2-rhel7**)

當 Jenkins server 佈署好之後，就會自動的與 OpenShift 本身的認証機制進行整合，完全不用使用者進行額外的設定，相關的 credential 也都會設定在 Jenkins server 內部，使用者需要做的就是開始設計 pipeline 就對了!


----------------------------

Jenkins image for OpenShift
===========================

由於 Jenkins 並不是一個很容易入門上手的系統，因此為了提供使用者能更方便的使用 Jenkins，OpenShift 將 Jenkins 系統打包成 docker image，並加入了許多常用的 plugin 以及與 OpenShift 整合所需要的 plugin，還很貼心的做好相關整合的設定，讓使用者在建立 pipeline 時就會同時產生相對應的 Jenkins server 來使用，省略許多繁複且容易造成錯誤的設定過程。

目前使用者可能比較容易用到的 Jenkins image for OpenShift 可能會有以下兩種：

- Red Hat 版本：`registry.access.redhat.com/openshift3/jenkins-2-rhel7`

- CentOS 版本：`openshift/jenkins-2-centos7` (位於 DockerHub 上)

由於新版的 OpenShift pipeline plugin for Jenkins 目前僅在 CentOS 版本的 Jenkins image 支援，因此後面的範例將會以 CentOS 版本為主，並提供將原有的 Jenkins image 從 Red Hat 版本改成 CentOS 版本的方法。


----------------------------

安裝額外的 Jenkins plugin
=======================

由於 OpenShift 提供的 Jenkins image 本身就是一個具有 s2i 功能的 docker image，因此我們可以透過 s2i 的流程，將所需要安裝的 plugin 以 source code injection 的方式指定進來並安裝。

以下的範例將會進行以下的客製化：

1. 安裝 **redmine**, **gitlab-plugin**, **testlink** 三個 plugin (詳細版本可參考[原始碼](https://github.com/godleon/learning_openshift/blob/master/jenkins_customization/plugins.txt))

2. 預先加入 **scriptApproval.xml** 檔案，讓某些 method 可在 sandbox 的環境中執行 (詳細清單可參考[原始碼](https://github.com/godleon/learning_openshift/blob/master/jenkins_customization/configuration/scriptApproval.xml))

```bash
# 取得原始碼
$ git clone https://github.com/godleon/openshift-jenkins-customization.git

# 建立 ImageStream(custom-jenkins-2-centos7) BuildConfig(custom-jenkins-build)
$ oc create -f openshift-jenkins-customization/openshift-objects/

$ oc -n openshift start-build custom-jenkins-build

$ oc -n openshift get pods
NAME                           READY     STATUS      RESTARTS   AGE
custom-jenkins-build-1-build   0/1       Completed   0          22m

# 檢視 build log
$ oc -n openshift logs -f custom-jenkins-build-1-build
```

> 使用者必須先安裝 [OpenShift CLI tool](https://github.com/openshift/origin/releases) 才可以執行上述的 oc 指令

在最後一行指令中檢視 build log 時，就可以看見 OpenShift 透過 s2i 流程將我們在程式碼中所指定的 plugin 都已經安裝完成，此時我們就可以透過 ImageStream `custom-jenkins-2-centos7`(定義於 **Jenkins_Customization/bc_custom-jenkins-build.yml** 中) 來作為啟動 Jenkins server 的 default image。


----------------------------

更換原有的 Jenkins Image
======================

今天在測試設計較為複雜的 Jenkins pipeline 時，發現 [OpenShift 在 GitHub 提供的範例](https://github.com/openshift/origin/tree/master/examples/jenkins/pipeline) 無法正常的使用，會出現 openshift class 不存在的錯誤，後來仔細的查了一下，發現原來裡面的範例需要搭配 [OpenShift Jenkins Pipeline (DSL) Plugin](https://github.com/openshift/jenkins-client-plugin) 一起使用。

**問題是，Red Hat 版本的 Jenkins image 並沒有內建這一個 plugin，此 plugin 目前僅有內建在 Centos 版本的 Jenkins image 中。**

而在上一個步驟中，我們使用了 CentOS 版本的 Jenkins image，並安裝了額外的 plugin 作為後續使用，因此以下便使用已經客製化完成的 Jenkins image 來作為啟動 Jenkins server 的 image。

為了更改自動佈署的 Jenkins server 所使用的 docker image，需要調整 `jenkins-ephemeral` 的內容，將 image 從 Red Hat 版本改到在上一個步驟完成的客製化 CentOS 版本，執行以下指令：

```bash
$ oc edit template/jenkins-ephemeral -n openshift
```

找到 `parameters` --> `JENKINS_IMAGE_STREAM_TAG`，將 **value** 從 `jenkins:latest` 改為 `custom-jenkins-2-centos7:latest`，存檔即可。

經過了以上的設定，後面自動佈署出來的 Jenkins server 都會是 CentOS7 的版本，也會同時預載好 [OpenShift Jenkins Pipeline (DSL) Plugin](https://github.com/openshift/jenkins-client-plugin) 以及在上一個步驟額外安裝好的 plugin。


----------------------------

References
==========

- [OpenShift V3 Plugin for Jenkins (based on Kubernetes plugin)](https://github.com/jenkinsci/openshift-pipeline-plugin)

- [OpenShift Jenkins Pipeline (DSL) Plugin (newly design)](https://github.com/openshift/jenkins-client-plugin)

- [Using Jenkins Pipelines with OpenShift @GitbHub](https://github.com/openshift/origin/tree/master/examples/jenkins/pipeline)

- [Jenkins - Other Images \| Using Images \| OpenShift Container Platform 3.6](https://docs.openshift.com/container-platform/3.6/using_images/other_images/jenkins.html)

- [Script approvals needed for changes to build config Jenkinsfile · Issue #57 · openshift/jenkins-sync-plugin](https://github.com/openshift/jenkins-sync-plugin/issues/57)

- [fabric8io/jenkins-docker: docker file for a jenkins docker image](https://github.com/fabric8io/jenkins-docker)