---
layout: post
title:  "[Elasticsearch] API CheatSheet"
description: "Elasticsearch 常用 API 的整理，方便未來操作 Elasticsearch 時的查詢用途"
date: 2020-11-17 18:40:00
published: true
comments: true
categories:
  - Elasticsearch
tags:
  - Elasticsearch
---


cluster 狀態查詢
===============

- [GET _cluster/health/<target>](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html)：查詢 cluster 健康狀況

- [GET /_cat/nodes](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-nodes.html)：回傳 cluster node 相關資訊

- [GET /_cat/shards/<target>](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-shards.html)(`GET /_cat/shards`)：顯示 shard 詳細資訊，包含哪些是 primary/replica，儲存多少 document，花費多少空間，位於哪個 data node 上



Document CRUD
=============

- [Index API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html)：將 document 新增置 index 使其變成可被搜尋，若是資料已經存在，則 document 會刪除，並重新進行索引的過程，且版本號碼(version)的值會增加
  - `PUT /<target>/_doc/<_id>`
  - `POST /<target>/_doc/`
  - `PUT /<target>/_create/<_id>`
  - `POST /<target>/_create/<_id>`

- [Get API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-get.html)：從 index 中取得指定資料
  - `GET <index>/_doc/<_id>`
  - `HEAD <index>/_doc/<_id>`
  - `GET <index>/_source/<_id>` (只取得 document 本身內容)
  - `HEAD <index>/_source/<_id>` 

- [Update API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update.html)：更新指定的 document (不會刪除原本的 document，可以更新部份屬性的值) **[使用方法須詳閱官網文件]**
  - `POST /<index>/_update/<_id>`



搜尋(Search/Query)
===================

- [Search API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html)：查詢資料
  - `GET /<target>/_search`
  - `GET /_search`
  - `POST /<target>/_search`
  - `POST /_search`

## URI Query 查詢語法

### 指定欄位 v.s. 泛查詢

- 指定欄位：`q=title:2012`

- 泛查詢：`q=2012`

### Term Query v.s. Phrase Query

- `Beautiful Mind` == `Beautiful OR Mind` (term query)

- `"Beautiful Mind"` == `Beautiful AND Mind` (phrase query)

### 分組 & 引號

- `title:(Beautiful AND Mind)` == `title="Beautiful Mind"`

### Boolean 查詢

使用 `AND`/`OR`/`NOT` 或是 `&&`/`||`/`!`

- 必須是大寫

- 範例：`title:(matrix NOT reloaded)`

### 分組

- `+` 表示 **must**

- `-` 表示 **must_not**

- 範例：`title:(+matrix -reloaded)`

### 範圍查詢

- `year:{2019 TO 2018}`

- `year:[* TO 2018]`

### 算術符號

- `year:>2010`

- `year:(>2010 && <=2018)`

- `year:(+>2010 +<=2018)`

### 萬用字元/正規表示式/模糊查詢 & 近似查詢

- 萬用字元：`title:mi?d`、`title:be*`

- 正規表示式：`title:[bt]oy`

- 模糊查詢 & 近似查詢：`title:beautifl~1`(輸入一個錯誤字母還是可以找到)、`title:"lord rings"~2`(可以找到 **load of the rings**)


## Request Body 查詢語法

### 一般查詢範例

```bash
# ignore_unavailable=true，可以忽略嘗試訪問不存在的 index “404_idx” 導致的錯誤
# 查詢 movies index
# 開啟 profile
POST /movies,404_idx/_search?ignore_unavailable=true
{
  "profile": true,
	"query": {
		"match_all": {}
	}
}

# 透過 from & size 達到分頁效果
POST /kibana_sample_data_ecommerce/_search
{
  "from":10,
  "size":20,
  "query":{
    "match_all": {}
  }
}

# 對資料排序，使用 sort 參數
POST kibana_sample_data_ecommerce/_search
{
  "sort":[{"order_date":"desc"}],
  "query":{
    "match_all": {}
  }

}

# source filtering
# 當某些 document 很大時，僅針對特定的幾個 term 做查詢
POST kibana_sample_data_ecommerce/_search
{
  "_source":["order_date", "category.keyword"],
  "from": 10,
  "size": 5, 
  "query":{
    "match_all": {}
  }
}
```

### Scripted Field(腳本字段) Query

```bash
# 透過 ES 中 painless script 算出新的 field value
# 搜尋結果中都會加上 hello 作為結尾
GET kibana_sample_data_ecommerce/_search
{
  "script_fields": {
    "new_field": {
      "script": {
        "lang": "painless",
        "source": "doc['order_date'].value+'hello'"
      }
    }
  },
  "from": 10, 
  "size": 5,
  "query": {
    "match_all": {}
  }
}
```

### 使用查詢表達式 - Match

```bash
# 預設為 "last OR christmas"
POST movies/_search
{
  "query": {
    "match": {
      "title": "last christmas"
    }
  }
}

# 可透過 operator 改為 "last AND christmas"
POST movies/_search
{
  "query": {
    "match": {
      "title": {
        "query": "last christmas",
        "operator": "AND"
      }
    }
  }
}
```

### 使用查詢表達式 - Match Phrase

```bash
# 必須按照順序出現
POST movies/_search
{
  "query": {
    "match_phrase": {
      "title":{
        "query": "one love"
      }
    }
  }
}

# 加上 slop，中間可以有一個其他的 term 插入
POST movies/_search
{
  "query": {
    "match_phrase": {
      "title":{
        "query": "one love",
        "slop": 1

      }
    }
  }
}
```