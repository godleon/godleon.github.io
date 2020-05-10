---
layout: post
title:  "[Terraform] 使用上的小技巧整理"
description: "本篇文章是學習 Terraform 的過程中，將覺得值得記錄的一些注意事項 & 小技巧進行整理後所留下的筆記，後續若是有其他值得記錄的內容，會再繼續補上"
date: 2020-05-11 05:45:00
published: true
comments: true
categories:
  - Terraform
tags:
  - Terraform
  - DevOps
  - IaC
---


規劃 Provider Plugin Cache 存放位置
=================================

當執行 `terraform init` 時，terraform 會下載 provider plugin 並放在目前的目錄中，但如果專案中有相當多個目錄，就表示這樣的下載行為會被執行很多次，且會浪費不少空間。

因此為了解決這個問題，可以搭配 [CLI configuragion file](https://www.terraform.io/docs/commands/cli-config.html) 一起使用，只要在 `$HOME` 中放入 `.terraformrc` 檔案(only for non-Windows OS)，內容如下：(**指定目錄必須先建立好**)

```
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
```

這樣每次進行 `terraform init` 的時候，plugin cache 就會被放到同樣的地方囉!

> 若是設定環境變數 `TF_PLUGIN_CACHE_DIR` 也是可以有相同效果

> 另外比較需要注意的是，plugin cache 目錄不能與 third party plugin 目錄放在相同地方，會有問題，目前 terraform 還無法處理這樣的規劃安排


Terraform 的使用限制
==================

## count & for_each 無法與 resource output 搭配使用

原因是因為 terraform 在 plan 階段就必須預測 infra 建立後的樣子，但 resource output 卻必須等待 resource 產生後才可以取得，因此無法在 plan 階段預測結果，因此會失敗。

## count & for_each 無法用在 module 設定中

以下的設定會失敗：

```hcl
module "webserver_cluster" {
  source = "../../../../modules/services/webserver-cluster"

  count = 3

  ....
  ....
}
```


Zero Downtime Deployment 有其限制
================================

使用 **ASG(Auto Scaling Group)** 搭配 `create_before_destroy` policy 雖然可以達到 zero-download 的佈署，但並不會考慮到 auto scaling policy 的部份，因為 ASG 被產生後，只會使用 min_size 中所設定的值。

有兩個 workaround 可以解這個問題：

1. 將 **aws_autoscaling_schedule** 從 `0 9 * * *` 改成 `0-59 9-17 * * *`，若目前環境已經達到 schedule 的要求則不會變更，若是沒有，則一分鐘內就會套用規則

2. 開發 script，使用 [External Data Source](https://www.terraform.io/docs/providers/external/data_source.html) 的機制，搭配 `desired_capacity` 參數，也可以達成相同效果；但這樣會造成 terraform code 可攜性降低，也會變得較難維護


看起來可執行的 plan 還是有可能會壞掉
===============================

這通常是因為除了 terraform 之外，可能還使用了 CLI 之類的工具進行了資源上的管理，因此在佈署資源時產生了衝突。

而這樣的問題有兩種解法：

1. 全部用 terraform
> 讓 terraform state 可以完全符合現況

2. 若已經有現存的 infra 存在，則使用 import 命令產生 state


重構(Refactoring) 要很小心
========================

重構在一般程式語言的開發中是很常見的，但在 terraform 中可不是這樣子，在有些情況下，terraform 會出現非預期的結果，例如：

- 修改某個 resource name

- 修改 resource identifier

```hcl
# 將 instance 改為 cluster_instance
resource "aws_security_group" "instance" {.....}
```

然後 terraform 就會把原本的 resource 移除，重新建立一個新的，但這並非我們所想要的結果。

因此可用以下方式避免這個問題的發生：

- 先用 `terraform plan` 確認將要發生的事情，確保沒有任何 resource 會被不小心移除
> 特別是不小心改到無法變動的 resource parameter 時，就一定會造成 resource 被移除再重建，請務必小心

- 若是真的要進行 resource replace 的動作，可透過 `create_before_destroy` 先產生可用的 resource，再移除不需要的

- 調整 identifier 也同時需要調整 terraform state
> 建議不要手動修改 terraform state 內容，可搭配 `terraform state mv <ORIGINAL_REF> <NEW_REG>` 的方式進行調整


Eventual Consistency
====================

terraform 與 cloud provider API 是以非同步的方式進行通訊，因此即使 API 立即回傳了訊息，這訊息也不見得可以馬上拿來使用。

舉例來說，當建立一個 AWS EC2 instance，API 很快的回傳 **201 Created**，但實際上要馬上存取到相關資訊可能還是不行的，可能是以下原因造成的：

- 實際上 EC2 instance 並未完全建立成功

- 相關的資訊還沒有完全在整個 AWS region 中同步

因此，若是在 CI/CD 的自動化流程中，有透過 terraform 產生 resource，並立即要拿來使用的情境，中間就必須設定一些等待時間，或是持續的監測是否能取到所需要的資訊，才能確保自動化流程才可以每一次都順利完成。