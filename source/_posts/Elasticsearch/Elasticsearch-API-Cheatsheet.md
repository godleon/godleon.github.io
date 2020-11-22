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

## Cluster APIs

cluster API 處理的部份當然就屬於 cluster level 部份的查詢 & 操作，Elasticsearch 提供了相當多的 cluster API，列表如下：
![Elasticsearch - cluster API list](/blog/images/Elasticsearch/es_cluster-API-list.png)

以下列出常用的 cluster API：

- [`GET /_cluster/health/<target>`](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html)：查詢 cluster 健康狀況



資源使用狀況查詢
=============

## cat APIs

cat APIs 是以 human readable 形式提供出來，方便使用者閱讀，主要是用來查詢 cluster 中各個元件的狀態、資源使用...等資訊用，目前 Elasticsearch 支援的 cat API 有相當多種：
![Elasticsearch - cat API list](/blog/images/Elasticsearch/es_cat-API-list.png)

以下列出常用的 cat API：

- [`GET /_cat/nodes`](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-nodes.html)：回傳 cluster node 相關資訊

- [`GET /_cat/shards/<target>`](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-shards.html)(`GET /_cat/shards`)：顯示 shard 詳細資訊，包含哪些是 primary/replica，儲存多少 document，花費多少空間，位於哪個 data node 上



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

```json
//ignore_unavailable=true，可以忽略嘗試訪問不存在的 index “404_idx” 導致的錯誤
//查詢 movies index
//開啟 profile
POST /movies,404_idx/_search?ignore_unavailable=true
{
  "profile": true,
	"query": {
		"match_all": {}
	}
}

//透過 from & size 達到分頁效果
POST /kibana_sample_data_ecommerce/_search
{
  "from":10,
  "size":20,
  "query":{
    "match_all": {}
  }
}

//對資料排序，使用 sort 參數
POST /kibana_sample_data_ecommerce/_search
{
  "sort":[{"order_date":"desc"}],
  "query":{
    "match_all": {}
  }

}

//source filtering
//當某些 document 很大時，僅針對特定的幾個 term 做查詢
POST /kibana_sample_data_ecommerce/_search
{
  "_source":["order_date", "category.keyword"],
  "from": 10,
  "size": 5, 
  "query":{
    "match_all": {}
  }
}
```

### Scripted Field(腳本欄位) Query

```json
//透過 ES 中 painless script 算出新的 field value
//搜尋結果中都會加上 hello 作為結尾
GET /kibana_sample_data_ecommerce/_search
{
  "script_fields": {
    "new_field": {
      "script": {
        "lang": "painless",
        "source": "doc['order_date'].value+'_hello'"
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

```json
//預設為 "last OR christmas"
POST /movies/_search
{
  "query": {
    "match": {
      "title": "last christmas"
    }
  }
}

//可透過 operator 改為 "last AND christmas"
POST /movies/_search
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

```json
//必須按照順序出現
POST /movies/_search
{
  "query": {
    "match_phrase": {
      "title":{
        "query": "one love"
      }
    }
  }
}

//加上 slop，中間可以有一個其他的 term 插入
POST /movies/_search
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

## Query String Query 

```json
//可指定 default field(DF)
//可指定 operrator
POST users/_search
{
  "query": {
    "query_string": {
      "default_field": "tool",
      "query": "Docker AND Kubernetes"
    }
  }
}

//可指定多個 field
//搭配 AND/OR/NOT 可以有非常多的條件組合
POST users/_search
{
  "query": {
    "query_string": {
      "fields":["tool","about"],
      "query": "(Docker AND Kubernetes) OR (Java AND Elasticsearch)"
    }
  }
}
```


## Simple Query String Query

```json
//在 query 欄位中所有 AND/OR/NOT，會被視為搜尋內容
//operator 若需要指定，則必須透過 "default_operator" 欄位
POST users/_search
{
  "query": {
    "simple_query_string": {
      "query": "Docker Kubernetes",
      "fields": ["tool"],
      "default_operator": "AND"
    }
  }
}
```


## 單字符串多字段查詢

### Bool Query & Disjunction Max Query

```json
//新增多筆資料
POST /blogs/_bulk
{ "index": { "_id": 1 } }
{ "title": "Quick brown rabbits", "body":  "Brown rabbits are commonly seen." }
{ "index": { "_id": 2 } }
{ "title": "Keeping pets healthy", "body":  "My quick brown fox eats rabbits on a regular basis." }

//預期應該會是 id=2 的 document 相關性比較高
//但結果卻是 id=1 的相關度較高
//因為 id=1 的 document 在 title & body 都有 brown
POST /blogs/_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "Brown fox" }},
        { "match": { "body":  "Brown fox" }}
      ]
    }
  }
}

//若是欄位屬於競爭關係
//透過 Disjunction Max Query 就可以讓單一欄位最匹配的得到最高分數
POST /blogs/_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "title": "Brown fox" }},
        { "match": { "body":  "Brown fox" }}
      ]
    }
  }
}

//透過加上 tie_breaker 的方式，改變計算分數的方法
//讓實際出現較多查詢條件的 document 取得較高的算分
POST /blogs/_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "title": "Quick pets" }},
        { "match": { "body":  "Quick pets" }}
      ],
      "tie_breaker": 0.2
    }
  }
}
```

### Multi Match Query

#### Best Field

```json
//新增多筆資料
POST /blogs/_bulk
{ "index": { "_id": 1 } }
{ "title": "Quick brown rabbits", "body":  "Brown rabbits are commonly seen." }
{ "index": { "_id": 2 } }
{ "title": "Keeping pets healthy", "body":  "My quick brown fox eats rabbits on a regular basis." }

//Disjunction Max Query 搜尋的盲點
//兩個 document 都會取得相同的分數
POST /blogs/_search
{
  "query": {
    "dis_max": {
      //"tie_breaker": 0.2,
      "queries": [
        { "match": { "title": "Quick pets" }},
        { "match": { "body":  "Quick pets" }}
      ]
    }
  }
}

//multi_match 卻沒搭配 tie_break or minimum_should_match
//效果跟 dis_max 是相同的
//但適時的加上 tie_break or minimum_should_match 可以提高搜尋的準確度
POST /blogs/_search
{
  "query": {
    "multi_match": {
      "tie_breaker": 0.2,
      "minimum_should_match": "20%", 
      "type": "best_fields",
      "query": "Quick pets",
      "fields": ["title","body"]
    }
  }
}
```

#### Most Field

```json
//額外增加一個 "std" field，並使用 standard analyzer
PUT /titles
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "english",
        "fields": {"std": {"type": "text","analyzer": "standard"}}
      }
    }
  }
}

//新增資料
POST titles/_bulk
{ "index": { "_id": 1 }}
{ "title": "My dog barks" }
{ "index": { "_id": 2 }}
{ "title": "I see a lot of barking dogs on the road " }

//將 query.multi_match.type 改成 "most_fields"
GET /titles/_search
{
  "query": {
    "multi_match": {
      "query":  "barking dogs",
      "type":   "most_fields",
      "fields": [ "title", "title.std" ]
    }
  }
}

//也可以根據需要加入 boosting 的設定(範例中的 ^10 就是)
GET /titles/_search
{
  "query": {
    "multi_match": {
      "query":  "barking dogs",
      "type":   "most_fields",
      "fields": [ "title^10", "title.std" ]
    }
  }
}
```

#### Cross Field

```json
//新增一筆地址資料
PUT address/_doc/1
{
  "street": "5 Poland Street", 
  "city": "London", 
  "country": "United Kingdom", 
  "postcode": "W1V 3DG"
}

//透過 cross_field" 進行 multi field 資料搜尋
//若是 type 改為 "most_fields" 搭配 AND operator 就會找不到資料
POST address/_search
{
  "query": {
    "multi_match": {
      "query": "Poland Kingdom W1V",
      "type": "cross_fields", 
      "operator": "and",
      "fields": ["street", "city", "country", "postcode"]
    }
  }
}
```