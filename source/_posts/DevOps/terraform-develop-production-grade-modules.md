---
layout: post
title:  "[Terraform] Module 設計上的思考與原則"
description: "此篇文章是閱讀 \"Terraform Up and Running 2nd\" 中關於 module 設計時，將一些覺得重要的觀念 & 內容節錄下來的內容"
date: 2020-05-13 06:35:00
published: true
comments: true
categories:
  - Terraform
tags:
  - Terraform
  - DevOps
  - IaC
---


Module 設計上越小越好
===================

開發小的 module，而非大的，專注在單一且簡單的部份，大的 module 有不少缺點，列舉如下：

- module 太大，執行會很慢，會伴隨著很大的 state & 檢查，執行 `terraform plan` 可能會花上個數分鐘都有可能

- 若是把整個 infra 都寫進同一個 module，任何要修改的開發者都給予權限，而這權限就可以存取所有資源，這表示每個人必須是 admin，這完全了違反最小權限原則

- 任何一個小修改都可能會毀了很大一部份的現存系統

- module 不僅會讓人難以了解，很難協助做 review，且難以測試


設計可組合使用 Module
===================

為了設計出可組合搭配使用的 module，在設計時可以考慮使用類似一般程式語言的思考方式，將一個 module 視為一個有 input & output 的 function，而 module 只是中間處理 input 資訊後，輸出 output 給使用者，甚至是其他的 module。

因此幾個設計重點如下：

- 避免從外面讀取 state 資訊，而是改成由 input 取得 

- 避免將 state 資訊寫到外部，而是將需要回傳的資訊透過 output 傳出

> 上面兩個原則主要是為了避免 side effect，同時也讓 module 本身內容更為合理、更容易測試並重複使用

- 透過將 small module 組合的方式，來開發更為複雜的 module


設計可測試的 Module
=================

Module 被開發出來，確保可用是很重要的，大概有幾個方向可以參考：

- 為 module 撰寫相關的 RAEDME 說明，包含用途 & 使用方法

- 開發使用 module 的範例程式，若是可以是一個可直接執行的 script 最好(讓使用者不用寫太多的程式就可以執行測試)，可以作為確認可執行的參考

- 為範例程式撰寫相關的 README 說明

- 開發測試程式，也可以是一個作為 module 使用範例的內容

- 如果 module 設計上有考慮到在各種不同的情境下的處理，那就可以多寫幾個範例來作為說明，以下圖為例：

![Terraform module example structure](/blog/images/devops/terraform_module-example-structure.png)
> 同一個 `asg-rolling-deploy` module 就可以有各式不同的應用，可以搭配單一 EC2 instance，也可以加上 load balancer，或是自訂 tag

- 在開始開發 module 前，可以先試著寫 example，藉由範例思考 module 應該如何被使用，並嘗試描繪出 module 該有的 input & output，嘗試從使用者的角度來思考 module 應該如何被設計，在開發 module 時的方向就會明確的多
> 其實這樣的思考方式也是 TDD(Test-Driven Development) 的精神

- 在 module 中要鎖定 terraform & provider 的版本，因為 terraform 不同版本間並不相容，以下是範例：

```hcl
terraform {
  required_version = ">= 0.12, < 0.13"
}

provider "aws" {
  region = "us-east-2"

  # AWS provider 2.0 以上的版本皆可
  version = "~> 2.0"
}
```
> 若是針對 production 環境，可以考慮鎖定更細的版本號


Release Modules
===============

當 module 開發 & 測試到一個穩定階段，準備要釋出時，有幾項重點原則可以參考：

- 為每一個穩定的版本加上 tag，搭配 git，使用者就可以指定要使用哪個版本的 module

- 有落實版本控管的 module，即使版本更新發生問題了，也可以很快的恢復到上一個版本

- 若是泛用性很高的 module，也可以考慮發布到 [Terraform Registry](https://registry.terraform.io/) 開源給大家使用，目前 Terraform Registry 已經有很多現成的 module 可用了

- 要在 Terraform Registry 開源自己所開發的 module，其實是有不少規範要遵循的，例如：命名規格、檔案結構、版本控管 .... 等等

- 若要直接使用 Terraform Registry 上的 module，不一定要完整指定 Git Repo ＆ version 來達成，可以類似以下比較簡單的宣告方式：

```hcl
# 使用範本
module "<Name>" {
    source  = "<OWNER>/<REPO>/<PROVIDER>"
    version = "<VERSION>"
}

# 實際使用範例
module "vault" {
    source  = "hashicorp/vault/aws"
    version = "0.12.2"
}
```


References
==========

- [Terraform: Up and Running](https://www.terraformupandrunning.com/)