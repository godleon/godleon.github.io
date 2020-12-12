---
layout: post
title:  "[GitHub Actions] 學習重點節錄(1)"
description: "此篇文章是學習 Udemy 課程\"The Complete GitHub Actions & Workflows Guide\" 時，將重要的使用觀念 & 方法節錄整理而成，並非完整的教學文章"
date: 2020-12-12 11:00:00
published: true
comments: true
categories:
  - DevOps
tags:
  - GitHub Actions
  - DevOps
---


發生問題時如何追蹤 & 查詢
=====================

當 GitHub Action 執行過程中發生問題了，有幾個方式可以查詢：

1. 上 GitHub Actions 頁面查看 log

2. 若需要詳細一點的 log，可以下載 log archive

3. 若希望可以有 debug level 的 log，則可以到 repository 的 `Settings >> Secrets` 中，加入 **`ACTIONS_RUNNER_DEBUG`** & **`ACTIONS_STEP_DEBUG`** 兩個環境變數，並設定為 **`true`**
> 參考資料：[Enabling debug logging - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/managing-workflow-runs/enabling-debug-logging)



關於 Action 的引用
================

- 可以使用已經公開的 public action repository

- 可以透過 branch、tag、commit 來指定 action 的版本



Event
=====

## pull_request

- `on: [pull_request]` 在預設情況下，只會在 pull request 狀態是 `open`、`synchronize`, `reopened` 的時候被觸發，其他狀態並不會被觸發(例如：`closed`)

- 若要指定在其他狀態也同時要觸發，可以使用 `types` 關鍵字來明確定義 (參考下方文件，搜尋 `types:` 就可以找到相關資料了)

## 參考資料

- [Events that trigger workflows - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows)


Scheduler
=========

- 可以使用 [crontab guru](https://crontab.guru/) 來設計一個精準的 crontab 設定

- 設定的時間會以 `UTC` 時間為基準

- **最短僅能設定到每 5 分鐘執行一次**

- crontab 設定內容要用`單引號`包起來，用雙引號會沒作用....(這個真的很雷...)



從外部觸發 workflow
=================

當自動化相關程式寫好後，需要從外部觸發其實是稀鬆平常的事情，而 GitHub Actions 則提供了 [workflow_dispatch](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows#workflow_dispatch) & [repository_dispatch](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows#repository_dispatch) 兩種方式來達成外部觸發 workflow 的目的。

- [官網參考資訊](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows#manual-events)

- 由於一個 repository 中可以有多個 workflow；如果只要特定的 workflow，使用 `workflow_dispatch`；但若要觸發所有的 workflow，那就使用 `repository_dispatch`

- 要送出一個標準的 HTTP POST request 觸發 repository dispatch(送到 `https://api.github.com/repos/[USER_NAME]/[REPO_NAME]/dispatches`)，需要帶上三個 header 資訊，如下圖
  - **Accept**：`application/vnd.github.v3+json`
  - **Content-Type**：`application/json`
  - **Authorization**：`Bearer [GITHUB_TOKEN]`
> 其中 Authorization 的部份也可以換成 `Basic [bases64 編碼後的內容]`(編碼的內容使用 `Username:[GITHUB_TOKEN] | base64` 語法產生，不過要把最後面的 `o=` 換成 `==`)

- 透過 `event_type` 關鍵字(`{ "event_type": "build" }`)，可以把 type 的資訊送入 GitHub Action 中，藉由比對在 github action YAML 設定檔中的 type 參數，來觸發對應的 workflow

- 若要額外帶上其他 key/value 的參數，可以透過在 request body 中加上 `client_payload` 來完成(不能使用其他名稱，詳細說明可參考[官網文件說明](https://docs.github.com/en/free-pro-team@latest/rest/reference/repos#create-a-repository-dispatch-event))，在 action 中可以透過 `${{ github.event.client_payload.[DATA_KEY] }}` 取得



過濾指定目標所發生的事件
====================

- 除了可以指定各種 event 發生時觸發 workflow 外，也可以指定在特定的目標上發生變更的時候才觸發 workflow，而這目標可以是 `branches`、`tags`、`paths` ... 等等

- 也可以透過 `branches-ignore`、`tags-ignore`、`paths-ignore` ... 等等來做反向的設定

- 細部的 filter 條件可以參考[官網文件的 cheat sheet](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#filter-pattern-cheat-sheet) 說明



參考資料
=======

- [The Complete GitHub Actions & Workflows Guide | Udemy](https://www.udemy.com/course/github-actions/)

- [Workflow syntax for GitHub Actions - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions)

- [Repositories - create a repository dispatch event - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/rest/reference/repos#create-a-repository-dispatch-event)

- [Events that trigger workflows - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/reference/events-that-trigger-workflows)

- [lambci/serverless-actions: Serverless GitHub Actions](https://github.com/lambci/serverless-actions)
