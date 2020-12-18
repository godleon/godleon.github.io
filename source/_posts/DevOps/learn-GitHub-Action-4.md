---
layout: post
title:  "[GitHub Actions] 學習重點節錄(4) - Cache、Artifact、版本號管理 & Semantic Versioning 的運用搭配"
description: "此篇文章是學習 Udemy 課程 \"The Complete GitHub Actions & Workflows Guide\" 時，將學習Cache、Artifact、版本號管理 & Semantic Versioning ...等內容的過程中整理出來的重要觀念 & 使用方式"
date: 2020-12-16 06:00:00
published: true
comments: true
categories:
  - DevOps
tags:
  - GitHub Actions
  - DevOps
---


Cache & Artifacts
=================

## Cache

- 利用 cache 可以將特定的檔案(例如：nodejs modules)存入 cache，在下一次 workflow 執行時就可以不用重複下載之前已經下載過的套件

- 必須搭配 [`actions/cache`](https://github.com/actions/cache) 使用

- 可以參考 [actions/cache repository 中的範例](https://github.com/actions/cache/blob/main/examples.md)，裡面有各種程式語言的範例可以參考使用


## artifact

- 通常用來存放程式建置打包(build)後出來的檔案，需要透過 [`actions/upload-artifact`](https://github.com/actions/upload-artifact) 進行上傳

- 上傳後就可以在每個 action 中看到 artifact 可以被下載



Semantic Versioning & Conventional Commits
==========================================

- Semantic Versioning 是很多專案用來作為版本號管理時的規範之用

- 詳細的用法可以參考 [Semantic Versioning 2.0.0 | Semantic Versioning](https://semver.org/) 網站的說明

## Conventional Commits

- Conventional Commits 則是可以透過固定格式的 commit message，來自動產生版本號、release node ... 等這一類的資訊，甚至是關閉指定的 issue

- 詳細的用法可以參考 [Conventional Commits](https://www.conventionalcommits.org/) 網站的說明



自動化產生版本號 & Release Note
============================

- 透過 [semantic-release](https://github.com/semantic-release/semantic-release) 工具，搭配上面提到的 [Conventional Commits](https://www.conventionalcommits.org/)，可以全自動進行版本號的管理 & 套件的發佈

- 若要自動檢查 commit 是否有符合預先制定好的規則，可以使用 [commitlnt](https://github.com/conventional-changelog/commitlint) 這個工具來完成

- [Commitizen](https://github.com/commitizen/cz-cli) 則是一款協助用的工具，可以透過互動的方式，在 commit 過程中透過問答的方式填寫完所有需要的資訊



其他注意事項
==========

## trigger workflow in another workflow

- 無法在一個 workflow 中透過 `${{ secret.GITHUB_TOKEN }}` 去觸發另外一個 workflow(例如：CI workflow 中有發布 release，而有另外一個 release workflow 則用來送通知)

- 若需要達成上面觸發另外一個 workflow 的效果，需要新增到 `Developer settings` 新增一個 `Personal access token` 來作為發布 release 用的 token，如此一來另外一個 workflow 才會被觸發


## workflow status badge

- 若想要在 README 頁面增加一個 workflow 是否有完成的 badge 圖案，可以放一段 markdown 的圖片語法來達成，語法如下：
> `![](https://github.com/[USER_ACCOUNT]/[REPO_NAME]/workflows/[WORKFLOW_NAME]/badge.svg?branch=[BRANCH_NAME]&event=[EVENT_NAME])`

- 透過上面 badge 的連結可以發現，可以指定在特定的 branch & event 所對應到的結果來產生 badge



參考資料
=======

- [The Complete GitHub Actions & Workflows Guide | Udemy](https://www.udemy.com/course/github-actions/)

- [actions/cache](https://github.com/actions/cache)

- [actions/upload-artifact](https://github.com/actions/upload-artifact)

- [Semantic Versioning 2.0.0 | Semantic Versioning](https://semver.org/)

- [Conventional Commits](https://www.conventionalcommits.org/)

- [semantic-release/semantic-release: Fully automated version management and package publishing](https://github.com/semantic-release/semantic-release)