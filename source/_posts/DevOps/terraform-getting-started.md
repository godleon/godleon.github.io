---
layout: post
title:  "[Terraform] 入門學習筆記"
description: "本篇文章是在 Terraform 官網教學文件中學習時，把一些個人認為的重要觀念 & 過程記錄起來，需要詳細的說明可以參考官網教學"
date: 2019-05-22 15:15:00
published: true
comments: true
categories:
  - Terraform
tags:
  - Terraform
  - DevOps
---


Installing Terraform
====================

> [官網連結](https://learn.hashicorp.com/terraform/getting-started/install)

這邊沒什麼太特別，就下載 zip 檔之後，解壓縮到 `/usr/bin` 目錄中就可以用了。(我使用的是 Ubuntu 18.04 Desktop)

而放到其他路徑，並修改 `$PATH` 也是可以的。



Build Infrastructure
====================

> [官網連結](https://learn.hashicorp.com/terraform/getting-started/build)

## Configuration

以下給一個簡單範例：

```bash
provider "aws" {
  access_key = "ACCESS_KEY_HERE"
  secret_key = "SECRET_KEY_HERE"
  region     = "ap-northeast-1"
}

# resource <resource_type> <resource_name>
resource "aws_instance" "example" {
  ami             = "ami-0ccdbc8c1cb7957be"
  instance_type   = "t2.micro"
}
```

- `provider`：此區塊用來指定所要使用的 provider，上述的例子是 **aws**，而 [terraform 支援的 provider 有相當多](https://www.terraform.io/docs/providers/index.html)，而 provider 琳琅滿目，不一定是 infra 類，也有很多是純軟體服務類

- `resource`：此區塊則是定義透過指定的 provider，所要建立的資源為何，而 resource 可以是一台 VM，也可以是個 app

- `<resource_type>`：每個 provider 底下有很多不同的 resource type 可以用，上面的範例是 aws_intance，而這個 resource type 也會主動告訴 terraform 所要使用的 provider 為 **aws**

- `<resource_name>`：這就按照個人喜好 or 需求去定義

## Initialization

這個步驟是執行 `terraform init` 指令，而 terraform 會根據在當前的目錄下產生一些本地端設定，並根據上面的 configuration 下載相對應的 provider binary，並放到 `.terraform` 目錄中

## Apply Changes

有了 provider binary 後，就可以執行 `terraform apply` 來實際產生相對應的資源了；而執行了 `terraform apply`，terraform 並不會馬上動作，而是會先列出一串執行列表，告訴使用者將會做哪些事情，並詳細列出建立該 resource 所會使用到的參數(資訊)；此外，若是看到 `+` 符號，表示此 resource 將會被建立。

terraform 執行完後，會在當前目錄產生 `terraform.tfstate`，此檔案相當重要，包含了透過 terraform 產生出來的 resource 的詳細資訊，而 terraform 就是依靠此檔案來進行 resource 的追蹤 & 維護。

> 因為 **terraform.tfstate** 包含了所有 resource 的狀態，因此執行 terraform 指令時，都必須要有這個檔案在，確保 terraform 可以正確的掌握 resource 的狀況，而此需求可以考慮使用 [remote state](https://www.terraform.io/docs/state/remote.html) 來達成

透過 `terraform show` 就可以檢視目前 resource 的詳細狀態

## Provisioning

這部份則是當 instance 被建立之後，所接著要進行的初始化 or 額外安裝的工作。

由於上面的範例是透過標準的 AMI 產生出來的 instance，因此還需要使用者額外安裝服務上去，若是透過自訂的 images 來產生的 instance，可以直接將 service 所需要的程式 & library 通通包好，此時可以考慮搭配 [Packer](https://www.packer.io/) 來協助建立 image。



Change Infrastructure
=====================

> [官網連結](https://learn.hashicorp.com/terraform/getting-started/change)

若有變更也相當容易，以上面的例子來說，假設換一個 AMI，就修改 AMI ID 之後，再執行一次 `terraform apply` 即可，而修改 AMI ID 的話，原本的 instance 會被移除，並使用新的 AMI 重新再建立一個新的 instance。

> 同樣的，輸入 `terraform apply` 可以看到執行列表同時有 `+` & `-`，表示原本的 instance 會被移除，並重新建立



Destroy Infrastructure
======================

> [官網連結](https://learn.hashicorp.com/terraform/getting-started/destroy)

這沒什麼訣竅...只要 configuration & `terraform.tfstate` 資訊有正確，輸入 `terraform destroy` 即可



Resource Dependencies
=====================

若是要同時建利多個 resource 時，以下面的 configuration 為例：

```bash
provider "aws" {
  access_key = "ACCESS_KEY_HERE"
  secret_key = "SECRET_KEY_HERE"
  region     = "ap-northeast-1"
}

# resource <resource_type> <resource_name>
resource "aws_instance" "example" {
  ami             = "ami-0ccdbc8c1cb7957be"
  instance_type   = "t2.micro"
}

resource "aws_eip" "ip" {
  instance = aws_instance.example.id
}
```

由於我們指定 EIP 要綁定 instance，因此 terraform 會自動先建立 instance，取得 instance id 後，再建立 EIP。

透過 `aws_instance.example.id` 的設定，Terraform 可以自動判定 resource 之間的相依性，而使用者在撰寫 terraform configuration 時，則必須儘可能的將這一類的資訊給定清楚。

除了上面的範例可以明確讓 terraform 知道 resource dependency 之外，也可以透過 `depends_on` 關鍵字來強制 resource 按照特定的順序來產生，以下是個範例：

```bash
provider "aws" {
  access_key = "ACCESS_KEY_HERE"
  secret_key = "SECRET_KEY_HERE"
  region     = "ap-northeast-1"
}

resource "aws_s3_bucket" "example" {
    bucket = "godleon-learn-terraform"
    acl    = "private"
}

# resource <resource_type> <resource_name>
resource "aws_instance" "example" {
    ami             = "ami-0ccdbc8c1cb7957be"
    instance_type   = "t2.micro"

    depends_on      = [ aws_s3_bucket.example ]
}

resource "aws_eip" "ip" {
  instance = aws_instance.example.id
}
```

按照上面的 configuration，terraform 會按照以下順序建立 resource：

1. aws_s3_bucket.example

2. aws_instance.example

3. aws_eip.ip



Provision
=========

若是使用 image-based infrastructure，那就可以忽略 provision 這一段，而轉去看看 [Packer](https://www.packer.io/) 是否可以提供更多的協助。

但如果需要進行一些初始化工作的話，provisioner 可以讓使用者上傳檔案、執行 shell script，或是呼叫其他 configuration management tool 等等。

以下是一個簡單的 local provisioner 範例：

```bash
resource "aws_instance" "example" {
    ami             = "ami-0eb48a19a8d81e20b"
    instance_type   = "t2.micro"
    key_name        = "godleon"

    depends_on = [ aws_s3_bucket.example ]

    provisioner "local-exec" {
        command = "echo ${aws_instance.example.public_ip} > /tmp/ip_address.txt"
    }
}
```



Input Variables
===============

有些敏感 or 重要的資訊可能不適合放在 version control system 作管理，例如：密碼, access key ... 等等，而 terraform 也提供了一些方法讓某些資訊可以透過 variable 的方式輸入。

## 定義 variables

我們可以建立一個 `variables.tf` 的檔案(檔名無所謂，只要副檔名為 tf 即可，terraform 會把當前目錄的 *.tf 全部拉進來用)，內容如下：

```bash
variable "access_key" {}

variable "secret_key" {}

# 可以設定預設值
variable "region" {
  default = "ap-northeast-1"
}
```

有了以上的設定後，在 resource configuration 可以改成如下：

```bash
provider "aws" {
  access_key    = var.access_key
  secret_key    = var.secret_key
  region        = var.region
}
```

完成以上設定後，在執行 `terraform apply` 時，會以互動式的方式詢問變數值，但如果不想用互動式的方式指定變數，可以透過以下的語法來指定變數：

> terraform apply -var 'access_key=<YOUR_ACCESS_KEY>' -var 'secret_key=<YOUR_SECRET_KEY>'

## 其他傳入 input variable 的方式

### 使用 variable file

若想要更偷懶讓 terraform 自動載入 variable file，可以在當前目錄建立一個名稱為 `terraform.tfvars`(或是 `*.auto.tfvars`) 的檔案，並設定內容如下：

```ini
access_key = "<YOUR_ACCESS_KEY>"
secret_key = "<YOUR_SECRET_KEY>"
```

若 variable file 想要用其他名稱，那就在執行 `terraform apply` 時使用 `-var-file` 參數指定 variable file

### 使用環境變數(Environment Variable)

也可以透過環境變數的方式傳入 input value，terraform 會自動以 `TF_VAR_name` 的型式讀取目前的環境變數，因此若要傳入 **key1=value1**，只要設定好 `TF_VAR_key1 = value1` 即可。


## 複雜資料結構變數

terraform 還支援了較為複雜的資料結構變數，分別是 **List** & **Map**，以下是使用範例：

### List

```bash
# 宣告 cidrs 變數為 list 格式
variable "cidrs" { default = [] }

# 也可以明確的宣告其變數型態
variable "cidrs" { type = list }

# 在 terraform.tfvars 中可用下面方式給入值
cidrs = [ "10.0.0.0/16", "10.1.0.0/16" ]
```

### Map

由於在不同的 region，同樣的 AMI 會有不同的 ID，因此可以用 Map 來解決：

```bash
# 宣告 Map 型態的變數
variable "amis" {
  type = "map"
  default = {
    "us-east-1" = "ami-b374d5a5"
    "us-west-2" = "ami-4b32be2b"
  }
}

# 使用方式
# 其中 var.amis 是 map 變數，var.region 則是 key
resource "aws_instance" "example" {
  ami           = "${lookup(var.amis, var.region)}"
  instance_type = "t2.micro"
}
```



Output Variables
================

執行 terraform 指令時，除了 input 重要外，output 也同樣重要，透過 output 可以讓使用者取得關於 resource 的真正重要資訊(例如：load balancer IP, VPN address ... 等等)，特別是當環境很複雜時，設計良好的 output 肯定是必要的。

而 output 除了會在 `terraform apply` 執行後顯示，以可以在後續透過 `terraform output` 來查詢。

以下範例可以取得 [AWS Elastic IP](https://www.terraform.io/docs/providers/aws/d/eip.html) 的 public ip address：

```bash
# https://www.terraform.io/docs/providers/aws/d/eip.html
output "ip" {
  value = aws_eip.ip.public_ip
}
```

上面的範例中，output 名稱為 `ip`(**名稱必須唯一**)，內容為 AWS Elastic IP 的 public IP address，當執行完 `terraform apply` 之後，就會自動出現 output，但之後也可以透過以下指令取得單純的內容輸出：

> terraform output ip



Modules
=======

當 infra 越來越大時，原本將所有 resource, variable 等定義檔放在同一個目錄下的方式就顯的很不夠用，且無法重複利用，因此 terraform 就提供了一個稱為 `module` 的機制來管理這些 configuration，將複雜的 infra 設定拆開成一個個可以獨立管理的小模組。(類似 Ansible Role)

## 使用 module

目前 [Terraform Registry](https://registry.terraform.io/) 已經有不少個已經開發完成可以直接使用的 module，若使用者要建立的 infra 中剛好有對應的 module 可以協助建立其中一部份，可以直接拉下來使用。

以下是一個建立 AWS Consul cluster 的範例：

```bash
provider "aws" {
  access_key = "AWS ACCESS KEY"
  secret_key = "AWS SECRET KEY"
  region     = "us-east-1"
}

module "consul" {
  source = "hashicorp/consul/aws"

  num_servers = "3"
}
```

從以上的內容，加上目前已經學到的 terraform 相關知識，可以歸納出以下重點 & 結論：

- `source` 定義的是 module 所在的位置，這邊的定義方式是告訴 terraform 此 module 位於 terraform registry 中，因此必須透過 `terraform init` 來自動取得

- 由於是 public module，因此[官網可以查詢到可用的 input 有哪些](https://registry.terraform.io/modules/hashicorp/consul/aws/0.6.1?tab=inputs)

- 每個 module 基本上也都會有一些 output 資訊可以取得([上面的範例](https://registry.terraform.io/modules/hashicorp/consul/aws/0.6.1?tab=outputs))

- module 本身也可以來自於 Git repository, HTTP, 或是本地端檔案

- 透過 `terraform apply` + module 產生出來的 resource 的名稱，都會帶有 module 名稱作為 prefix

- 要將 resource 清除同樣也是可以使用 `terraform destroy`

## Module Output

在前面的範例中有指定 input `num_servers` 的值，而 output 也可以比照原本 terraform 的處理方式比照辦理，由於幾乎每個 module 都會有 output，因此[以上面的 consul module 為例]（https://registry.terraform.io/modules/hashicorp/consul/aws?tab=outputs），我們也可以使用 consul module output 來自訂我們想要的 output，範例如下：

```bash
# 取得 auto-scaling group 的名稱
output "consul_server_asg_name" {
  value = module.consul.asg_name_servers
}
```

接著取得 output 的方式就跟上面相同囉!



Remote State Storage
====================

前面有提到使用 terraform 時很重要的一個課題就是如何管理 state，若是自己私下練習可以放在本機就好，但是當 infra 有很多人同時維護時，可以就需要放在遠端可透過 Internet 存取的地方；terraform 提供了很多 remote state storage babckend 的支援，像是：s3, consul, etcd, terraform cloud ... 等等。

由於目前將 state 放到 terraform cloud 是免費的(此外在 user, workspace 都沒有使用數量的限制，加上官方宣稱使用 [Vault](https://www.vaultproject.io/) 作管理加密)，因此這邊就使用 terraform cloud 來做測試。

首先要取得存取 terraform cloud 的 token，步驟如下：

1. 到 [terraform cloud](https://app.terraform.io/signup) 官網註冊新帳號

2. 將 terraform 升級到 0.11.13 以上

3. 在 [terraform cloud](https://app.terraform.io/app/organizations/new) 建立新的 organization

4. 到 [terraform cloud user settings](https://app.terraform.io/app/settings/tokens) 中建立一個新的 token，並 copy token 內容

5. 在本地端建立 `~/.terraformrc` 檔案，並填入以下內容：

```bash
# 將上一個步驟的 token 內容取代下方的 REPLACE_ME
credentials "app.terraform.io" {
     token = "REPLACE_ME"
}
```

6. 在當前目錄下建立一個名稱為 `remote-storage-state.tf` 的檔案(檔名可隨意，副檔名是 `tf` 即可)，內容如下：

```bash
# 將 organization 填入
# workspace name 可以隨意取，terraform 會自動在 terraform cloud 上建立下面指定的 workspace 
terraform {
  # 其中 "remote" 的意思表示使用 terraform cloud
  # 若是 remote storage 使用 etcdv3，則會是 "etcv3"
  backend "remote" {
    organization = "<ORGANIZATION NAME>"

    workspaces {
      name = "<WORKSPACE NAME>"
    }
  }
}
```

7. 執行 `terraform init`，取得存取 remote storage 所需要的檔案（若本地端有存在 state 檔案，terraform 會詢問要不要移到 remote storage）

8. 接著後續的 `terraform apply` 就會自動將 state 存到 terraform cloud 了，還會有 version control 的功能喔!

另外一點比較重要的是，如果之後決定要將 state 從 remote 移回本地端，那就移除剛剛上面所新增的 `remote-storage-state.tf` 檔案，並再執行一次 `terraform init`，tetrraform 就會詢問要不要將 state 從 remote 移回本地端了!


References
==========

- [Terraform Curriculum - HashiCorp Learn](https://learn.hashicorp.com/terraform/)

- [Creating Modules - Terraform by HashiCorp](https://www.terraform.io/docs/modules/index.html)

- [Documentation - Terraform by HashiCorp](https://www.terraform.io/docs/index.html)

- [Example Configurations - Terraform by HashiCorp](https://www.terraform.io/intro/examples/index.html)

- [Import - Terraform by HashiCorp](https://www.terraform.io/docs/import/index.html)

- [Configuration Language - Terraform by HashiCorp](https://www.terraform.io/docs/configuration/index.html)