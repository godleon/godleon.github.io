---
layout: post
title:  "[Terraform] Input Variables "
description: "本篇文章介紹 Terraform Input Variables 如何使用 & 使用時的注意事項"
date: 2019-10-15 15:15:00
published: true
comments: true
categories:
  - Terraform
tags:
  - Terraform
  - DevOps
  - IaC
---


Overview
========

input variable 通常用來作為 module 的 parameter，而且在 root module 中定義好的 variable，可以根據需求透過 CLI or 環境變數來覆蓋。


變數宣告
=======

要宣告一個變數檔，基本上就是新增一個副檔名為 `tf` 的文字檔，放入類似以下內容：

```bash
# 變數名稱為 "image_id"，型態為 string
variable "image_id" {
  type = string
  # 可透過 "description" 參數對變數加上額外說明
  description = "The id of the machine image (AMI) to use for the server."
}

# 變數名稱為 "availability_zone_names"，型態為 list(string)
variable "availability_zone_names" {
  type    = list(string)
  # 可指定預設值
  default = ["us-west-1a"]
}

# 變數名稱為 "docker_ports"，型態為 list(object)
variable "docker_ports" {
  type = list(object({
    internal = number
    external = number
    protocol = string
  }))
  default = [
    {
      internal = 8300
      external = 8300
      protocol = "tcp"
    }
  ]
}
```

基本上變數的名稱可以隨意取(只要遵守[命名規則](https://www.terraform.io/docs/configuration/syntax.html#identifiers)即可)，但不可使用 `source`, `version`, `providers`, `count`, `for_each`, `lifecycle`, `depends_on`, `locals` ... 等關鍵字。


如何使用 input variable?
=======================

使用方式很簡單，一個簡單的範例如下：

```bash
# ami 設定中使用變數 image_id
resource "aws_instance" "example" {
  instance_type = "t2.micro"
  ami           = var.image_id
}
```


變數型態
=======

input variable 可以有很多型態，以下是目前支援的型態：

- `string`

- `number`

- `bool`

- `list(<TYPE>)`

- `set(<TYPE>)`

- `map(<TYPE>)`

- `object({<ATTR NAME> = <TYPE>, ... })`

- `tuple([<TYPE>, ...])`

> tuple 中每個元素可以是不同的型態，例如 `["a", 15, true]`

詳細的使用規範 & 範例可以參考[官網文件](https://www.terraform.io/docs/configuration/types.html)。


如何傳入 variable values ?
=========================

定義了 variable 之後，就會面臨到如何傳入 value 的問題。

以下的說明是將 value 傳入到 root module 中，若是要傳入到 child module 中，則是要在程式中自行處理。

## 透過 command line

透過 `-var` 關鍵字搭配 `terraform plan` or `terraform apply` 命令使用：

```bash
$ terraform apply -var="image_id=ami-abc123"

$ terraform apply -var='image_id_list=["ami-abc123","ami-def456"]'

$ terraform apply -var='image_id_map={"us-east-1":"ami-abc123","us-east-2":"ami-def456"}'
```

> `-var` 可以在單一指令中使用多次，來傳入多個 variable value

## 使用變數定義檔(`.tfvars`)

可以撰寫以 `.tfvars` or `.tfvars.json` 為副檔名的變數定義檔，以下是一個簡單的範例：

```bash
image_id = "ami-abc123"

availability_zone_names = [
  "us-east-1a",
  "us-west-1c",
]
```

以下是 json 格式的範例：

```json
{
  "image_id": "ami-abc123",

  "availability_zone_names": ["us-west-1a", "us-west-1c"]
}
```

可以透過類似以下的語法帶入變數定義檔：

> terraform apply -var-file="testing.tfvars"

而 Terraform 也可以自動將變數定義檔載入，只要符合以下任何一種命名規範即可：

- 檔案名稱為 `terraform.tfvars` or `terraform.tfvars.json`

- 任何檔名，副檔名為 `.auto.tfvars` or `.auto.tfvars`

> 必須放在執行 `terraform apply` or `terraform plan` 時所在的目錄中

## 用 Environment Variables 傳入

除了上面兩種外，Terraform 也會自動去使用環境變數開頭為 `TF_VAR_` 的環境變數來作為 variable value。

假設變數名稱為 `image_id`，只要將變數值設定到 `TF_VAR_image_id` 環境變數中，terraform 就會自己載入了!


傳入變數值的優先權
===============

上面提到幾個方法可以傳入變數值，那如果同時有多個方法被使用，哪個會被優先採用呢?

基本上順序是這樣的：

1. 首先會使用目錄中的 `terraform.tfvars` or `terraform.tfvars.json` 檔案

2. 若目錄中有 `*.auto.tfvars` or `*.auto.tfvars.json` 檔案，則會覆蓋掉上一個檔案中的變數值

3. 若在 command line 中透過 `-var` or `-var-file` 指定變數值或是檔案，則會覆蓋掉上面兩個檔案中提供的變數值



References
==========

- [Input Variables - Configuration Language - Terraform by HashiCorp](https://www.terraform.io/docs/configuration/variables.html)

- [Type Constraints - Configuration Language - Terraform by HashiCorp](https://www.terraform.io/docs/configuration/types.html)
