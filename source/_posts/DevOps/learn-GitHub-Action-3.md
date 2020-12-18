---
layout: post
title:  "[GitHub Actions] 學習重點節錄(3) - 流程控制、timeout、matrix、container 的運用搭配"
description: "此篇文章是學習 Udemy 課程 \"The Complete GitHub Actions & Workflows Guide\" 時，將學習流程控制、timeout、matrix、container 的運用搭配...等內容的過程中整理出來的重要觀念 & 使用方式"
date: 2020-12-13 10:40:00
published: true
comments: true
categories:
  - DevOps
tags:
  - GitHub Actions
  - DevOps
---


流程 & 時間控制
=============

## continue on error

- 若是在 step 上設定 `continue-on-error: true`(預設為 `false`)，即使該 step 出現問題，後面的 step 也會繼續執行


## timeout 設定

- 預設的 job timeout 設定為 `360`(mins, 共 6 hours)

- 若要調整 job timeout 為 8 hours，可以在 job 中加上 `timeout-minutes: 480` (也可以降低)

- **timeout-minutes** 的設定同時也可以用在 `step` level 上



Matrix
======

## 調整程式執行環境

- 如果要變更 step 中的執行環境，可以額外加一個 step 並使用 [`actions/setup-node`](https://github.com/actions/setup-node) 來完成


## Matrix

若是要針對多種環境進行相同的 workflow，透過 [matrix](https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/managing-complex-workflows#using-a-build-matrix) 的方式可以省下很多複製貼上的功夫，以下是一個簡單範例：

```yaml
name: matrix test

on: push

jobs:
  node-version:
    strategy:
      matrix:
        os: [macos-latest, ubuntu-latest, windows-latest]
        node_version: [6, 8, 10]
      fail-fast: true
    runs-on: ${{ matrix.os }}
    steps:
      - name: display node version
        run: node -v
      - uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node_version }}
      - name: display node version after setup-node
        run: node -v
```

Matrix 在上方的設定中，一共有 9 個 workflow(3 x 3) 會被產生出來，但如果有以下兩個環境不需要怎麼辦：

- os: macos-latest + node_version: 6

- os: ubuntu-latest + node_version: 8

此時可以使用 `exclude` 來達成此功能，因此設定會變成以下這樣：

```yaml
name: matrix test

on: push

jobs:
  node-version:
    strategy:
      matrix:
        os: [macos-latest, ubuntu-latest, windows-latest]
        node_version: [6, 8, 10]
        # 多了以下這一段
        # 因此 workflow 的數量會變成 7 個 (3 x 3 - 2)
        exclude:
          - os: macos-latest
            node_version: 6
          - os: ubuntu-latest
            node_version: 8
      fail-fast: true
    runs-on: ${{ matrix.os }}
    steps:
      - name: display node version
        run: node -v
      - uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node_version }}
      - name: display node version after setup-node
        run: node -v
```

此外，若是在 **os: ubuntu-latest + node_version: 10** 的時候，會有個特別的參數 `is_ubuntu_10` 需要加入，那怎麼辦呢?

此時可以透過 `include` 關鍵字來達成，因此程式會變成以下這樣：

```yaml
name: matrix test

on: push

jobs:
  node-version:
    strategy:
      matrix:
        os: [macos-latest, ubuntu-latest, windows-latest]
        node_version: [6, 8, 10]
        include:
          - os: ubuntu-latest
            node_version: 10
            is_ubuntu_10: 'true'
        exclude:
          - os: macos-latest
            node_version: 6
          - os: ubuntu-latest
            node_version: 8
      fail-fast: true
    runs-on: ${{ matrix.os }}
    env:
      IS_UBUNTU_10: ${{ matrix.is_ubuntu_10 }}
    steps:
      - name: display node version
        run: node -v
      - uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node_version }}
      - name: display node version after setup-node
        run: |
          node -v
          echo "IS_UBUNTU_10 = $IS_UBUNTU_10"
```

> 只是務必要記得，`include` 並不是用來額外加上一組 workflow 組合，而是要**針對已經在 matrix 已經存在的組合，額外多加參數用的**



使用 Container
==============

GitHub Actions 在使用 container 的功能上，大概會有以下幾種情境：

1. job in container

2. service container

3. step in container

## job in container

這個情況是不要使用 VM 作為執行環境，而是使用指定的 container image 作為執行環境，以下是一個簡單範例：

```yaml
name: Container
on: push

jobs:
  node-docker:
    # 還是需要指定 ubuntu 作為 container runtime 運行的平台
    runs-on: ubuntu-latest
    # 定義 job 所要使用的 container image
    container: 
      image: node:10.23.0-alpine3.11
    steps:
      # 顯示 Alpine Linux v3.11
      - name: display node version
        run: |
          node -v
          cat /etc/os-release
      # 顯示 Alpine Linux v3.11
      # 表示 job 中的所有 step 都會使用相同的 container 環境
      - name: display OS
        run: |
          uname -r
          cat /etc/os-release
```


## [service container](https://docs.github.com/en/free-pro-team@latest/actions/guides/about-service-containers)

透過 `service` 關鍵字，指定要在 VM 上佈署的 multiple container 環境(也可以是 single container)，然後可以在 VM 中對服務進行測試，以下是一個簡單範例：

```yaml
name: Container-2
on: push

jobs:
  container-job:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis
        ports:
          - 6379:6379
    steps:
      - name: test redis connection
        run: |
          cat /etc/os-release
          telnet localhost 6379

```

由於實際 step 的運行是發生在 VM 端，因此下面還是要透過 localhost:6379 才可以找到服務；若是 service 中定義了多個 container，則 container 之間的溝通就可以直接透過 container name 來進行。


## step in container

```yaml
name: Container-3
on: push

jobs:
  container-job:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis
        # 在 step 中的 container 會與 service container 使用相同的 docker network
        # 因此就不需要開啟 port mapping 了
        # ports:
        #   - 6379:6379
    steps:
      - name: test redis connection
        uses: docker://redis
        with:
          entrypoint: '/usr/local/bin/redis-cli'
          args: '-h redis ping'
```

## 如何在 container 中執行 repo 中的程式?

假設有一個 shell script，位於 repo 的根目錄下，內容如下：

```bash
#!/bin/sh
echo $1
echo "this is a custom script"
```

問題是，如何在 container 執行這個 script 呢?

只要使用下面的語法定義 actions 即可：

```yaml
name: Container-4
on: push

jobs:
  container-job:
    runs-on: ubuntu-latest

    steps:
      # 這裡將 repo checkout 後，檔案會帶往下面的 steps
      - uses: actions/checkout@v1
      - name: display file list
        run: ls -a
      # 執行 repo 中的 script.sh 檔案
      - name: run a script
        uses: docker://alpine
        with:
          entrypoint: ./script.sh
          args: 'hello-world'
```



參考資料
=======

- [The Complete GitHub Actions & Workflows Guide | Udemy](https://www.udemy.com/course/github-actions/)

- [About service containers - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/guides/about-service-containers)

- [Creating a Docker container action - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/creating-actions/creating-a-docker-container-action)