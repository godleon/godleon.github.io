---
layout: post
title:  "[GitHub Actions] 學習重點節錄(2)"
description: "此篇文章是學習 Udemy 課程 \"The Complete GitHub Actions & Workflows Guide\" 時，將學習 Environment Variables、Encryption、Expressions、Context 等內容的過程中整理出來的重要觀念 & 使用方式"
date: 2020-12-12 17:20:00
published: true
comments: true
categories:
  - DevOps
tags:
  - GitHub Actions
  - DevOps
---


環境變數(Environment Variable)的存取
==================================

- Environment Variable 可以分為 `workflow`、`job`、`step` 三個 scope，使用的方式同樣都是透過關鍵字 `env` 的方式定義，存取的範圍也會依據定義的位置而有所不同

- 若要讓環境變數更安全，可以到 repository 的 `settings > Secrets` 頁面中，新增 **repository secret**，這樣在 action 執行過程中就不會以明碼顯示；而存取的方式則是透過 `${{ secrets.[SECRET_NAME] }}` 

- repository secret 有最大長度 64KB 的限制

- 若是要在 action 中使用大型的加密檔案(> 64KB)，可以透過 GPG 工具加密後，傳到 repo 中，再將密碼存入 repository secret，然後在 action 中再次透過 GPG 工具解密後使用



關於在 Workflow 中授權機制
=======================

在上一篇文章中有提到，若是要從外部觸發 workflow，需要有一個有足夠權限的 token，而這個 token 需要到開發者帳號的 `settings >> Developer settings` 頁面中取得。

但是，如果假設執行的環境已經在 action 內部了，那還需要申請這樣一個 token 嗎?

答案是不用的，只要透過 `${{ secrets.GITHUB_TOKEN }}` 就可以取得一個合法的 token，而這 token 也正式代表了執行目前這個 action 的身份，而這個 token 具體有哪些權限，可以參考[官方文件的說明](https://docs.github.com/en/free-pro-team@latest/actions/reference/authentication-in-a-workflow#permissions-for-the-github_token)。

而 GITHUB_TOKEN 可以拿來做什麼用呢? 可能會有以下應用場景：

- 在自動化過程中，建立 issue

- 在 action 執行過程，產生某些檔案，再重新 commit/push 回原本的 repository

總之 GITHUB_TOKEN 可以做蠻多事情的，若是有需要對原本的 repository 做一些異動，是可以透過 GITHUB_TOKEN 作為認證身份來完成這些事情的。



Expression & Context
====================

## Context 

Context 在 GitHub Actions 的文件中是這麼介紹的：
> Contexts are a way to access information about workflow runs, runner environments, jobs, and steps. Contexts use the expression syntax.

因此表示，要存取 workflow 的任何相關資訊，或是與 workflow 的相關設定，就必須要認識有哪些 context 可用，可透過[此官網文件連結](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#contexts)取得最新的 context 列表，而 context 的用法則是 `${{ <context> }}`


## Expression

Expression 一般來說有兩個用途：

1. 針對變數做一些程式化的調整 & 修改，這會在 `${{ }}` 中發生，而且通常會搭配 [function](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#functions) 使用，資料來源很多時候是來自於 [context](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#contexts)

2. 使用 `if` 關鍵字來調整 workflow 在特定情況下才能執行，而資訊來源很多時候也是來自於 [context](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#contexts)

以下舉幾個變數搭配 function 的範例：(詳細說明可參考 [GitHub Actions 官網文件](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions#functions))

- `contains(github.event.issue.labels.*.name, 'bug')`針對 array 中的每個 **name** 值來判斷是否有 bug 這個字串存在

- `contains('Hello world', 'llo')`

- `startsWith('Hello world', 'He')`

以下是幾個 if 使用的例子：

- if 不僅可用在 job level 上，也可以用在 step level 上
> 例如：`if: github.event_name == 'push'` (github context 提供了很多資訊可以用來在 if 條件上作判斷，**event_name** 只是其中一種而已)

- 使用 `if: success()` or `if: failure()` or `if: always()` 來根據前一個 job or step 的結束狀態來決定要不要執行特定工作



參考文件
=======

- [The Complete GitHub Actions & Workflows Guide | Udemy](https://www.udemy.com/course/github-actions/)

- [Authentication in a workflow - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/reference/authentication-in-a-workflow)

- [Context and expression syntax for GitHub Actions - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/actions/reference/context-and-expression-syntax-for-github-actions)