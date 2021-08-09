
---
layout: post
title:  "[Elasticsearch] 深入搜索"
description: "此篇文章是在極客時間學習 Elasticsearch 課程時留下的一些學習筆記，主要內容包含 Elasticsearch 深入搜索"
date: 2019-11-09 08:00:00
published: true
comments: true
categories:
  - Elasticsearch
tags:
  - Elasticsearch
---


Conclusion
==========

以下的內容其實不少，先把重點抓到這裡來：

- term query 用於數據化結構的查詢

- Full text 使用 match 查詢

- bool query 是種複合查詢，可以結合 term query & full text query

- multi-match + best field = disjunction max query



Term Query & Full Text Query
============================

## 重點整理

- Term Query 不會做分詞處理，而 Full Text Query 會做分詞處理

- 要做精準搜尋，使用 `[FIELD_NAME].keyword` 欄位

- 透過 Constant Score query(將關鍵字 `term` 改成 `constant_score` + `filter` + `term`)，可以將查詢轉換為 filter，跳過算分(scoring)步驟並可利用 cache 來加速查詢效能
> **若是要快速將不需要的資料過濾掉，constant score query 是很好的一個方式**


## [Term Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html)

- Term 是表達語意的最小單位，搜尋或是利用自然語言進行處理時都需要處理 term

- 查詢的語法中，只要指定 `term`(**query >> term**) 關鍵字，就表示使用 term query

- Term Level Query 包含 **Term Query** / **Range Query** / **Exists Query** / **Prefix Query** / **Wildcard Query**

- 當 ES 接收到 term query 時，對輸入不會進行分詞，而是將輸入當作一個整體在 inverted index 中找到精確的詞項，並使用相關度算分的機制來計算分數

- 若是不需要算分，則可以利用 `constant score` 將查詢轉換成 `filter`，並利用 cache 來提高效能

以下使用範例說明 term 查詢所需要注意的事項：

```json
//新增 index
PUT products
{
  "settings": {
    "number_of_shards": 1
  }
}

//新增三筆資料
POST /products/_bulk
{ "index": { "_id": 1 }}
{ "productID" : "XHDK-A-1293-#fJ3","desc":"iPhone" }
{ "index": { "_id": 2 }}
{ "productID" : "KDKE-B-9947-#kL5","desc":"iPad" }
{ "index": { "_id": 3 }}
{ "productID" : "JODL-X-1937-#pV7","desc":"MBP" }

//檢視 index 的 setting & mapping 資訊
GET /products

//搜尋 "iPhone" => 找不到資料，因為預設的 index analyzer 分詞後會將每個單字轉小寫；
//但 term query 不會經過 analyzer，因此查詢時沒有轉小寫
//搜尋 "iphone" => 改成小寫，順利找到資料
POST /products/_search
{
  "query": {
    "term": {
      "desc": {
        "value": "iPhone"
        //"value":"iphone"
      }
    }
  }
}

//改成在 "desc.keyword" field 含有大寫字母的 "iPohne"，可以找到資料
//因為 keyword field 會完整保留原始資料
//反而使用小寫 "iphone" 是無法找到資料的
POST /products/_search
{
  "query": {
    "term": {
      "desc.keyword": {
        "value": "iPhone"
        //"value":"iphone"
      }
    }
  }
}

//跟上面的範例相同，大寫的搜尋條件無法找到資料
//因為預設的 analyzer 會將內容根據 `-` 切開後再作小寫處理
POST /products/_search
{
  "query": {
    "term": {
      "productID": {
        "value": "XHDK-A-1293-#fJ3"
      }
    }
  }
}

//使用 keyword field 就可以正確使用大寫的搜尋條件找到資料了
POST /products/_search
{
  //"explain": true,
  "query": {
    "term": {
      "productID.keyword": {
        "value": "XHDK-A-1293-#fJ3"
      }
    }
  }
}

//透過 "constant_score" 將查詢轉換成 filter 來提昇查詢效率
//因為算分過程被忽略，所有搜尋結果都是 1 分
POST /products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "productID.keyword": "XHDK-A-1293-#fJ3"
        }
      }

    }
  }
}
```

## [Full Text Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/full-text-queries.html)

- Full text query 可以是 match query, match phrase query, query string query ... 等等，詳細的列表可以參考[官網資料](https://www.elastic.co/guide/en/elasticsearch/reference/current/full-text-queries.html)

- index & query 都會進行分詞，查詢字串會先傳給一個合適的 analyzer，並生成用來查詢的 term list

- 分詞後的 term list 會被逐一拿來查詢，並將最後結果合併後，為每個 document 計算出一個分數

以下是幾個簡單範例：

```json
//若只有 query，預設的 operator 是 or，因此會找到多筆資料
//搭配 operator or minimum_should_match，可以讓查詢結果更準確
POST /movies/_search
{
  "query": {
    "match": {
      "title": {
        //"operator": "and", 
        //"minimum_should_match": 2,
        "query": "Matrix reloaded"
      }
    }
  }
}

//也可以使用 "match_phrase" 搭配 "slop" 讓搜尋結果更準確
POST /movies/_search
{
  "query": {
    "match_phrase": {
      "title": {
        //"slop": 1,
        "query": "Matrix reloaded"
      }
    }
  }
}
```


## Match Query 的查詢過程

- 屬於 Full Text Qeury：包含了 `Match Query`、`Match Phrase Query`、`Query String Query`

- Document 被索引 & 搜尋時都會經過分詞處理，然後產生提供查詢的詞項列表

- 查詢時會根據詞項列表逐一查詢，再將結果合併後，為每一個 document 進行算分(scoring)

![Elasticsearch - Match Query progress](/blog/images/Elasticsearch/es_match-query-progress.png)



結構化搜尋
=========

## 結構化數據

- 像是 date, boolean, number ... 這一類的數據都是屬於結構化的

- 有些 text 也屬於結構化的，例如：顏色(red, green, blue)、tag(distributed, search)、特定編碼....只要有遵守規定產生的 text，都可以算是結構化的格式

> 而結構化搜尋就是對結構化的數據進行搜尋

- 結構化的 text 可以做精確比對(term query) or 部份比對(prefix query)

- 結構化的搜尋結果只會有 "true" or "false" 兩種值，並且可以根據需求來決定是否做 scoring 的行為

以下是一些簡單範例：

```json
//結構化搜索，精確匹配
DELETE products

//加入資料
//並不是所有資料都有 date 欄位
POST /products/_bulk
{ "index": { "_id": 1 }}
{ "price" : 10,"avaliable":true,"date":"2018-01-01", "productID" : "XHDK-A-1293-#fJ3" }
{ "index": { "_id": 2 }}
{ "price" : 20,"avaliable":true,"date":"2019-01-01", "productID" : "KDKE-B-9947-#kL5" }
{ "index": { "_id": 3 }}
{ "price" : 30,"avaliable":true, "productID" : "JODL-X-1937-#pV7" }
{ "index": { "_id": 4 }}
{ "price" : 30,"avaliable":false, "productID" : "QQPX-R-3956-#aD8" }

//查詢 dynamic mapping 後的結果
GET products/_mapping

//進行 term query，並計算出分數
POST products/_search
{
  "profile": "true",
  "explain": true,
  "query": {
    "term": {
      "avaliable": true
    }
  }
}

//進行 term query，但使用 filter context，因此不算分
POST products/_search
{
  "profile": "true",
  "explain": true,
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "avaliable": true
        }
      }
    }
  }
}

//使用 range 查詢 & 搭配 filter context 跳過算分步驟
GET products/_search
{
  "query" : {
    "constant_score" : {
      "filter" : {
        "range" : {
          "price" : {
            "gte" : 20,
            "lte"  : 30
          }
        }
      }
    }
  }
}

//使用特殊字元(y->年)處理日期相關查詢
POST products/_search
{
  "query" : {
    "constant_score" : {
      "filter" : {
        "range" : {
          "date" : {
            "gte" : "now-2y"
          }
        }
      }
    }
  }
}

//使用 prefix 搜尋年份為 201 開頭的電影
//但這樣的搜尋必須是 year 欄位為 keyword/text/wildcard 才有辦法執行
GET /movies/_search
{
  "query": {
    "prefix": {
      "year": "201"
    }
  }
}


// =========== multi-value field 的處理 ===========

//若是 field 中包含多個 text
POST /movies/_bulk
{ "index": { "_id": 1 }}
{ "title" : "Father of the Bridge Part II","year":1995, "genre":"Comedy"}
{ "index": { "_id": 2 }}
{ "title" : "Dave","year":1993,"genre":["Comedy","Romance"] }

//由於 genere 為 multi-value field
//因此以下搜尋會將 genre 中有包涵 Comedy 的資料全部顯示出來
//若希望有完全精準的比對，則必須額外加上一個 count 欄位，搭配 boolean query 來完成
POST /movies/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "genre.keyword": "Comedy"
        }
      }
    }
  }
}
```



搜索的相關性算分
==============

## 相關性 & 相關性算分

- **搜尋的相關性算分**描述一個 document 與查詢語句相符的程度，ES 會對每個符合查詢條件的結果進行算分，並放到 _score field 中

- 分數的是要用在搜尋結果排序中，在 ES5 之前使用 `TF-IDF` 算分，後面的版本使用 `BM 25`


## Term Frequency (詞頻) & 逆文檔頻率(Inverse Document Frequency, IDF)

- 檢索詞在一篇 document 中出現的頻率 = `檢索詞出現的次數 / document 總字數`

- 衡量查詢 & 結果 document 相關性的簡單方法就是將每個 term 的 TF 相加 = `TF(word1) + TF(word2) + TF(word3)`

- stop word 出現太多，一般計算分數時不會列入考慮

- IDF = `log(全部 document 數量 / 檢索詞出現過的 document 總數)`

- `TF-IDF` 其實就是從 sum(TF) 變成了 sum(TF + IDF)

- 現代的 search engine 大多以 TF-IDF 為基礎再加上大量的優化

![Elasticsearch - TD-IDF](/blog/images/Elasticsearch/es_tf-idf-scoring.png)

- 從上圖可見，TF-IDF 的評分公式，除了 TD & IDF 之外，還包涵了 `boosting` & `field length(欄位長度)` 兩項


## BM 25

![Elasticsearch - BM 25](/blog/images/Elasticsearch/es_bm25.png)

- 跟 TF-IDF 相比，當 TF 無限增加時，BM 25 算分會趨近於某個值，不會無限變大


## Boosting Relevance

- boosting 是可以用來控制相關度的一種方式，可用在 index, field, 或是查詢子條件中

- 當 `boost > 1` 可提昇相關度，`0 < boost < 1` 計算分數的權重相對降低, `boost < 0`，貢獻為負分

以下是一個簡單的範例：

```json
//新增資料
PUT /testscore/_bulk
{ "index": { "_id": 1 }}
{ "content":"we use Elasticsearch to power the search" }
{ "index": { "_id": 2 }}
{ "content":"we like elasticsearch" }
{ "index": { "_id": 3 }}
{ "content":"The scoring of documents is caculated by the scoring formula" }
{ "index": { "_id": 4 }}
{ "content":"you know, for search" }

//使用 match query 並檢視實際算分過程
//會計算 tf, idf ... 等值
POST /testscore/_search
{
  "explain": true,
  "query": {
    "match": {
      //"content":"you"
      "content": "elasticsearch"
      //"content":"the"
      //"content": "the elasticsearch"
    }
  }
}

//加上 boosting 後的算分方式
//可自訂 boosting 來對搜尋結果進行優化
POST testscore/_search
{
  "explain": true,
  "query": {
    "boosting" : {
      "positive" : {
        "term" : {
          "content" : "elasticsearch"
        }
      },
      "negative" : {
        "term" : {
          "content" : "like"
        }
      },
      "negative_boost" : 0.2
    }
  }
}
```


Query & Filtering 與多字符串多字段查詢
===================================

## Query Context & Filter Context

- 在 Elasticsearch 中，有 Query & Filter 兩種不同的 Context ([官網文件](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html))

- Query Context 會對相關性進行算分

- Filter Context 不會進行算分，但可利用 cache 來取得更好的效能


## Bool Query

- 一個 bool query 是一個 or 多個查詢子句所組成 (可組合成複合查詢)

| 子句 | 效果 |
|-----|------|
| `must` | (**Query Context**) 必須符合，對算分有貢獻 |
| `should` | (**Query Context**) 選擇性符合，對算分有貢獻 |
| `must_not` | (**Filter Context**) 必須不能符合，對算分無貢獻 |
| `filter` | (**Filter Context**) 必須符合，對算分無貢獻 |

- bool query 中的每一個查詢子句得到的分數都會被合併成總和的相關性評分

關於查詢的語法，有以下幾點需要注意：

- 子查詢可以以任意的順序出現

- 可用 list 的方式在一個子查詢中加入多個查詢

- 若 bool query 中沒有 must 條件，那 should 


## 範例

### 包含多個子查詢的 Bool Query

```json
//新增多筆資料
POST /products/_bulk
{ "index": { "_id": 1 }}
{ "price" : 10,"avaliable":true,"date":"2018-01-01", "productID" : "XHDK-A-1293-#fJ3" }
{ "index": { "_id": 2 }}
{ "price" : 20,"avaliable":true,"date":"2019-01-01", "productID" : "KDKE-B-9947-#kL5" }
{ "index": { "_id": 3 }}
{ "price" : 30,"avaliable":true, "productID" : "JODL-X-1937-#pV7" }
{ "index": { "_id": 4 }}
{ "price" : 30,"avaliable":false, "productID" : "QQPX-R-3956-#aD8" }

//bool query，使用多個子查詢
POST /products/_search
{
  "query": {
    "bool" : {
      //Query Context
      "must" : {
        "term" : { "price" : "30" }
      },
      //Filter Context
      "filter": {
        "term" : { "avaliable" : "true" }
      },
      //Filter Context
      "must_not" : {
        "range" : {
          "price" : { "lte" : 10 }
        }
      },
      //Query Context
      "should" : [
        { "term" : { "productID.keyword" : "JODL-X-1937-#pV7" } },
        { "term" : { "productID.keyword" : "XHDK-A-1293-#fJ3" } }
      ],
      "minimum_should_match" :1
    }
  }
}
```

### 複雜的 Bool Query

在 bool query 中再放入一個 bool query：(多層 bool query 的概念)

```json
POST /products/_search
{
  "query": {
    "bool": {
      "must": {
        "term": {
          "price": "30"
        }
      },
      //在 bool query 的子查詢中再塞進一個 bool query
      "should": [
        {
          "bool": {
            "must_not": {
              "term": {
                "avaliable": "false"
              }
            }
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```

### 不同的算分標準

```json
POST /animals/_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "text": "brown" }},
        { "term": { "text": "red" }},
        { "term": { "text": "quick"   }},
        { "term": { "text": "dog"   }}
      ]
    }
  }
}

POST /animals/_search
{
  "query": {
    "bool": {
      "should": [
        { "term": { "text": "quick" }},
        { "term": { "text": "dog"   }},
        {
          //將同類型的條件分類在另外一個 bool query 中
          //會讓相關性分數的計算更加準確
          "bool":{
            "should":[
               { "term": { "text": "brown" }},
                 { "term": { "text": "brown" }},
            ]
          }

        }
      ]
    }
  }
}
```

### 搭配 boosting 進行查詢

```json
DELETE news

//新增多筆資料
POST /news/_bulk
{ "index": { "_id": 1 }}
{ "content":"Apple Mac" }
{ "index": { "_id": 2 }}
{ "content":"Apple iPad" }
{ "index": { "_id": 3 }}
{ "content":"Apple employee like Apple Pie and Apple Juice" }

//若是想要搜尋的是與 apple computer 相關的資訊
//最後一筆與食物相關的訊息會變成搜尋結果第一筆，因為 apple 出現次數最多
POST /news/_search
{
  "query": {
    "bool": {
      "must": {
        "match":{"content":"apple"}
      }
    }
  }
}

//限制食物相關訊息不能出現
POST /news/_search
{
  "query": {
    "bool": {
      "must": {
        "match":{"content":"apple"}
      },
      "must_not": {
        "match":{"content":"pie"}
      }
    }
  }
}

//希望 apple 關鍵字的訊息都出現
//但透過 boosting 的設定讓食物相關訊息分數較低
//只要有 match 'pie' 的搜尋結果，算分會變更低
POST /news/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": {
          "content": "apple"
        }
      },
      "negative": {
        "match": {
          "content": "pie"
        }
      },
      "negative_boost": 0.5
    }
  }
}
```


單字符串多字段查詢：Disjunction Max Query
======================================

一般在進行 single string multiple field 的查詢時，預設的分數計算方式過於簡單，往往會出現非使用者所預期的情況，例如以下範例：

```json
DELETE blogs

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

//改用 dis_max 後，會綜合判斷所有符合 document 的分數 & 相關 field
//找到最符合的評分後再返回
//因此 id=2 的 document 符合程度較高，就會被排在前面
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
```

因此從上面的範例來看，透過 **Disjunction Max Query** 可以找到最符合查詢條件的結果。

但 Disjunction Max Query 也並非萬靈丹，因為其算分方式也是有盲點的，例如以下範例：

```json
//使用 Disjunction Max Query，會讓以下搜尋結果的分數都相同
POST /blogs/_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "title": "Quick pets" }},
        { "match": { "body":  "Quick pets" }}
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

關於 `Disjunction Max Query` & `tie_breaker` 的搭配，可以參考[官方文件](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-dis-max-query.html)說明。



單字符串多字段查詢：Multi Match
============================

一般 single string multiple field 的查詢，會發生在三個場景：

- `Best Field` (預設 type)

> 每個 field 之間會互相競爭，搜尋時會把評分最高的 field 中的資料回傳(可以額外加上 `tie_breaker` or `minumum_should_match` 來調整搜尋精準度)

- `Most Field`

> 通常會使用在英文的內容時，通常會在 main field 中設定 English Analyzer，並加入同義詞來試圖符合更多的 document，並適時的 text 中加入 sub field 搭配 standard Analyzer 盡量保留原始訊息，以提供更精確的搜尋結果

- `Cross Field`

> 某些訊息需要同時在多個 field 查詢時，可以做一種類似多(single string -> multi words)對多(multi field)的搜尋

首先使用以下幾個範例來說明 **Best Field**：

```json
DELETE blogs

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
      //"tie_breaker": 0.2,
      //"minimum_should_match": "20%", 
      "type": "best_fields",
      "query": "Quick pets",
      "fields": ["title","body"]
    }
  }
}
```

```json
DELETE titles

//設定 analyzer 為 english
PUT /titles
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "english"
      }
    }
  }
}

//新增多筆資料
POST titles/_bulk
{ "index": { "_id": 1 }}
{ "title": "My dog barks" }
{ "index": { "_id": 2 }}
{ "title": "I see a lot of barking dogs on the road " }

//搜尋結果跟預期不同
//由於 english analyzer 會將 barking dogs 分詞成 bark & dog 兩個字
//因此上面兩個 document 計算出的分數都是相同的
GET titles/_search
{
  "query": {
    "match": {
      "title": "barking dogs"
    }
  }
}
```

從上面可以看出 best field 在某些時候的確會有一些盲點，因此我們可以試著使用以下方法來調整：

- 額外增加一個 `std` field，並使用 standard analyzer

- 將 **query.multi_match.type** 改成 `most_fields`

> 也可以根據需求適時加入 boosting 設定

```json
DELETE /titles

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

接著如果我們需要針對更複雜的資料(例如：地址)，並同時在多個 field 中做搜尋，則可以藉由 `cross_fields` 來取得更精確的搜尋結果：

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
> 若使用 `copy_to` 的方式解決，就需要額外的儲存空間



全文檢索案例
==========

## 思考 & 分析

- `Full text query` or `Term query`

- 針對搜尋的條件 & field，定義合適的 analyzer 是很重要的

- 開發時需要對不同的 analyzer 做測試

- 透過調整 mapping 設定，並改寫搜尋條件，來交叉評估不同的搜尋結果以取得期望的結果

## 相關性測試

- 相關性測試並非簡單工作，必須了解原理 & 多做分析 & 多做調整

- 可能會需要多 analyzer & boosting ... 等參數做各式各樣的調整

- 每個環境跟需求對應到的設定 & 搜尋條件也都不會相同，不會有 silver bullet 般的設定可以一次解決所有問題

## 監控 & 理解用戶行為

- 相關性測試其實是最後一步，不要進行過度的相關性測試是很重要的

- 必須嘗試監控用戶的搜尋結果，並試著理解使用者的使用行為
  
  - 例如：在後台實作功能，查詢使用者的哪些查詢是沒有回傳任何結果的

  - 衡量使用者對於實際搜尋結果的點擊率



使用 Search Template 和 Index Alias 查詢
=======================================

## [Search Template](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-template.html)

- 主要的功能在於 de-couple `程式`與`查詢用的 DSL` 兩者之間的關係，讓專業的工程師可以各司其職的完成工作(開發人員/查詢工程師/效能調校工程師)

- 程式設計師不需要了解查詢的優化細節，只要使用 ES 工程師提供的 Search Template 即可

- ES 工程師則可以專注在效能的調校 & 優化 search template 工作上

以下是一個簡單範例：

```json
//新增 search template，id = 'tmdb'
POST _scripts/tmdb
{
  "script": {
    "lang": "mustache",
    "source": {
      "_source": [
        "title","overview"
      ],
      "size": 20,
      "query": {
        "multi_match": {
          "query": "{{q}}",
          "fields": ["title","overview"]
        }
      }
    }
  }
}

//檢視 search index 的內容
GET _scripts/tmdb

//使用 search template 查詢，帶入預先定義的參數 q
POST tmdb/_search/template
{
    "id":"tmdb",
    "params": {
        "q": "basketball with cartoon aliens"
    }
}

//刪除 search template
DELETE _scripts/tmdb
```

## [Index Alias](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html)

- `index alias` 就跟 Linux 中的 `alias` 指令一樣，是作為設定別名的用途。

- 而通常為 index 設定別名是有其需要的，例如每天建立一個新的 index，但在程式中每天都根據日期去產生一個字串來作為 index 來查詢，其實挺麻煩的；此時透過設定一個名稱為 **latest_index** 並指到每天最新的 index 的方式，問題就迎刃而解了。

- 除了別名之外，甚至可以額外加入 filter 條件，先針對 index 資料內容進行 filter 後再查詢

以下是一個簡單範例：

```json
PUT /movies-2019/_doc/1
{
  "name":"the matrix",
  "rating":5
}

PUT /movies-2019/_doc/2
{
  "name":"Speed",
  "rating":3
}

//建立 index alias
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "movies-2019",
        "alias": "movies-latest"
      }
    }
  ]
}

//對 index alias 進行查詢
POST /movies-latest/_search
{
  "query": {
    "match_all": {}
  }
}

//另外建立一個 index alias
//並額外增加 filter 條件對資料先進一步過濾
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "movies-2019",
        "alias": "movies-lastest-highrate",
        //index alias 設定中還可以額外設定 filter 
        "filter": {
          "range": {
            "rating": {
              "gte": 4
            }
          }
        }
      }
    }
  ]
}

//對 index alias 進行查詢
POST /movies-lastest-highrate/_search
{
  "query": {
    "match_all": {}
  }
}
```


綜合排序：Function Score Query 優化算分
====================================

## Scoring & Sorting

- Elasticsearch 預設會以 document 的相關度算分進行排序

- 可以指定一個 or 多個 field 進行排序

- 但預設情況下，無法對相關度 or 排序進行更多細部的控制調整


## [Function Score Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html)

可以在查詢結束後，對每一個符合的 document 進行重新算分，並根據新的分數來進行排序。

Elasticsearch 中已經包含了以下幾種用來算分的函數：(非全部，比較常用的，詳細資訊可參考[官網](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html))

- `Weight`：為每一個 document 設定一個簡單不被規範化的權重

- `Field Value Factor`：可指定特定的 field 來影響算分過程，例如指定**點讚數**作為算分的條件之一

- `Random Score`：隨機算分

- `Decay Function`：以某個 field 的值作為基準，距離越遠得分越高

- `Script Score`：自己寫 script 來自定算分邏輯

以下用一個實際範例來說明使用方法：

```json
DELETE blogs

//加入三筆一模一樣的資料到 blogs index
//唯一的差別只有 votes 數量而已
//若使用預設的 BM25 進行算分，三個 document 會得到相同分數
PUT /blogs/_doc/1
{
  "title":   "About popularity",
  "content": "In this post we will talk about...",
  "votes":   0
}
PUT /blogs/_doc/2
{
  "title":   "About popularity",
  "content": "In this post we will talk about...",
  "votes":   100
}
PUT /blogs/_doc/3
{
  "title":   "About popularity",
  "content": "In this post we will talk about...",
  "votes":   1000000
}

//使用 function score query，搭配 field_value_factor 來指定重要的 field
//但會發現 votes 值超高的 document 會取得超高的分數
POST /blogs/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field": "votes"
      }
    }
  }
}
```

由於上面只根據 `votes` 欄位作為重新計算分數的依據，因此投票數超高的 document 得到了超高的分數。

但若是希望算分不要差距這麼大，希望做個平滑處理，那可以搭配 `modifier` & `factor` 兩個參數來完成，以下是示範範例：

```json
//因此除了 field_value_factor 之外
//可以額外設定 modifier = log1p 來進行平滑曲線設定
//new score = old score * log(1 + votes)
POST /blogs/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field": "votes",
        "modifier": "log1p"
      }
    }
  }
}

//除了 modifier 之外，還可以額外增加 factor 對平滑曲線做更細膩的控制
//new score = old score * log(1 + factor * votes)
POST /blogs/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      "field_value_factor": {
        "field": "votes",
        "modifier": "log1p" ,
        "factor": 0.1
      }
    }
  }
}
```

### Boost Mode & Max Boost

在 function score query 中同樣也可以加入 boost 的設定，並可以搭配多種不同的 boost mode 來調整計算分數時的基礎：

- `multiply`: query score 與 function score 相乘(預設值)

- `replace`: 忽略 query score，只使用 function score

- `sum`: query score 與 function score 相加

- `avg`: query score 與 function score 的平均值

- `max`: query score 與 function score 兩者中較大的值

- `min`: query score 與 function score 兩者中較小的值

以下是個簡單範例：

```json
POST /blogs/_search
{
  "query": {
    "function_score": {
      "query": {
        "multi_match": {
          "query":    "popularity",
          "fields": [ "title", "content" ]
        }
      },
      //使用 function score query，搭配 log1p & factor 的設定來做平滑曲線的處理
      "field_value_factor": {
        "field": "votes",
        "modifier": "log1p" ,
        "factor": 0.1
      },
      //還可以額外設定 boost mode & max_boost 來進行搜尋的調校
      "boost_mode": "sum",
      //最高就是 3 分了
      "max_boost": 3
    }
  }
}
```

### 一致性的隨機函數

- 當網站上的廣告需要提高曝光率，又不想要亂槍打鳥時 (希望每個使用者進來看到的廣告是相同的)

- 透過一致性的隨機函數可以讓每個用戶看到不同的隨機排名，但同一個用戶每一次的 request 都會得到一致的回應

以上的需求就可以透過 function score query 中的 `Random Score` 來達成：

```json
//以隨機但一致的方式進行算分(透過 seed 參數控制)
//因此 seed 可以給入像是 user id 這一類的資訊
POST /blogs/_search
{
  "query": {
    "function_score": {
      "random_score": {
        "seed": 911119
      }
    }
  }
}
```

### 其他參考資料

- [通過Function Score Query優化Elasticsearch搜尋結果 - IT閱讀](https://www.itread01.com/content/1549644662.html)

- [ElasticSearch - function_score 簡介](https://kucw.github.io/blog/2018/7/elasticsearch-function_score/)



Term & Phrase Suggester
=======================

[Elastic 官網文件 - Suggester](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html)

## 什麼是搜尋建議?

- 搜尋建議是現代化搜尋引擎常見的功能

- 用途是在幫助使用者在輸入搜尋內容的過程中，協助補完搜尋內容，或是進行錯誤的偵測，當然也可以推薦用戶符合更多搜尋條件的建議，因此這樣的作法同樣也可以提高搜尋時找到合適結果的機率


## [Elasticsearch Suggester API](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html)

- 搜尋建議的功能在 Elasticsearch 中是透過 Suggester API 來實現的

- 以 term suggester 為例，其運作原理在於將輸入的內容分解為 token，接著在索引的字典裡面尋找相似的 term 並回傳

- 目前 Elasticsearch 一共支援四種 suggester，分別是 [`Term`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#term-suggester), [`Phrase`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#phrase-suggester), [`Completion`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#completion-suggester), [`Context`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#context-suggester) 四種

- Term suggester 中還可以額外設定好幾個 suggestion mode，分別是：

  - `Missing`: 如果索引中已經存在，就不提供建議 (表示使用者已經輸入正確的內容)

  - `Popular`: 推薦出現頻率比較高的詞

  - `Always`: 無論是否存在，都提供建議

以下是幾個簡單範例：

```json
DELETE articles

//在 index 中新增多筆資料
POST articles/_bulk
{ "index" : { } }
{ "body": "lucene is very cool"}
{ "index" : { } }
{ "body": "Elasticsearch builds on top of lucene"}
{ "index" : { } }
{ "body": "Elasticsearch rocks"}
{ "index" : { } }
{ "body": "elastic is the company behind ELK stack"}
{ "index" : { } }
{ "body": "Elk stack rocks"}
{ "index" : {} }
{  "body": "elasticsearch is rock solid"}

//進行 term suggestion
//透過檢查 response 中的 "suggest" 區段內容
//可以發現 "lucen" 取得了 "lucene" 的建議
//而 rock 就沒有了，因為 suggest mode 設定為 missing 的關係
POST /articles/_search
{
  "size": 1,
  "query": {
    "match": {
      "body": "lucen rock"
    }
  },
  "suggest": {
    "term-suggestion": {
      "text": "lucen rock",
      "term": {
        "suggest_mode": "missing",
        "field": "body"
      }
    }
  }
}

//同樣的 suggest 搜尋，但搭配 suggestion mode 為 "popular"
//此時 rock 就會出現了
POST /articles/_search
{

  "suggest": {
    "term-suggestion": {
      "text": "lucen rock",
      "term": {
        "suggest_mode": "popular",
        "field": "body"
      }
    }
  }
}
```

每個回傳的 suggest 都包含了一個算分，這也代表著相似性(相似性越高，分數越高)，相似性是由 **Levenshtein Edit Distance** 的計算方法來實作出來的，核心概念就是 `一個詞變更了多少字元就可以和另外一個詞一致`。

而相似性的設定部份也可以透過一些參數可以來控制，例如 `max_edits`


```json
//若是打錯字(範例中的 hocks)，也是可以可以取得搜尋建議
//但是必須要把 prefix_length 那行設定打開
//prefix_length 的功能如下：
// The number of minimal prefix characters that must match in order be a candidate for suggestions. Defaults to 1. Increasing this number improves spellcheck performance. Usually misspellings don’t occur in the beginning of terms. (Old name "prefix_len" is deprecated)
POST /articles/_search
{

  "suggest": {
    "term-suggestion": {
      "text": "lucen hocks",
      "term": {
        "suggest_mode": "always",
        "field": "body",
        //"prefix_length":0,
        "sort": "frequency"
      }
    }
  }
}

//一個很簡單的 phrase suggester 的範例
POST /articles/_search
{
  "suggest": {
    "my-suggestion": {
      "text": "lucne and elasticsear rock hello world ",
      //要改成 phrase suggester，只要在這裡從 "term" 改為 "phrase" 即可
      "phrase": {
        "field": "body",
        //相較於 term suggester，還額外支援了不少的搜尋參數
        //稍微調整一下兩個參數的值就可以看出會有些不同
        "max_errors":2,
        "confidence":0,
        "direct_generator":[{
          "field":"body",
          "suggest_mode":"always"
        }],
        "highlight": {
          "pre_tag": "<em>",
          "post_tag": "</em>"
        }
      }
    }
  }
}
```

## Fuzzy Query

Fuzzy Query 也是以 **Levenshtein Edit Distance** 為基礎，使用 `fuzzy` 關鍵字，可以容忍使用者搜尋時輸入少量的錯誤，依然可以找到符合的結果，以下是簡單的範例：

```json
//一個字元輸入錯誤，還是可以找到資料
//因為 fuzziness 設定為 1
GET /movies/_search
{
  "query": {
    "fuzzy": {
      "title": {"value": "intersteller", "fuzziness": 1}
    }
  }
}

//但有兩個字元輸入錯誤就無法找到資料了
GET /movies/_search
{
  "query": {
    "fuzzy": {
      "title": {"value": "inersteller", "fuzziness": 1}
    }
  }
}

//調整 fuzziness = 2 之後就可以找到資料了
GET /movies/_search
{
  "query": {
    "fuzzy": {
      "title": {"value": "inersteller", "fuzziness": 2}
    }
  }
}
```




Completion & Context Suggester
==============================

這個是搜尋體驗優化的另一個重要功能；這功能可以協助使用者在輸入每一個字元時，系統可以快速到後端查詢到與目前輸入字元相關的結果並給出查詢字串補全的功能。

此外，在 7.0 版後的 Elasticsearch，還提供了一個 `[search_as_you_type](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-as-you-type.html)` data type，用來協助使用者建置查詢字串自動補全的功能。


## [Completion Suggester](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#completion-suggester)

- Completion Suggester 提供 auto completion 的功能，因此使用者每輸入一個字，就需要即時發送一個查詢到後端來搜尋符合項目

- 承上，這樣的行為對效能的要求就會相對嚴格；而 Elasticsearch 本身則採用了不同的數據結構來實現這個需求，將 analyzer 處理過後的數據編碼成 FST 和索引一起存放，而整個 FST 會被放到記憶體中，因此存取相對快很多

- 但 FST 只能進行 prefix 的搜尋

要使用 completion suggester，就必須要在 index mapping 的設定中進行相關設定：

```json
PUT articles
{
  "mappings": {
    "properties": {
      "title_completion":{
        //需要將 type 設定為 completion
        "type": "completion"
      }
    }
  }
}
```

以下是實際的操作範例：

```json
//新增資料
POST articles/_bulk
{ "index" : { } }
{ "title_completion": "lucene is very cool"}
{ "index" : { } }
{ "title_completion": "Elasticsearch builds on top of lucene"}
{ "index" : { } }
{ "title_completion": "Elasticsearch rocks"}
{ "index" : { } }
{ "title_completion": "elastic is the company behind ELK stack"}
{ "index" : { } }
{ "title_completion": "Elk stack rocks"}
{ "index" : {} }

//使用 completion suggester
POST articles/_search?pretty
{
  "size": 0,
  "suggest": {
    "article-suggester": {
      "prefix": "e",
      //也可以試試看
      //"prefix": "elk",
      "completion": {
        "field": "title_completion"
      }
    }
  }
}
```


## [Context Suggester](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#context-suggester)

- Context Suggester 是由 completion suggester 擴充而成的

- 可以在搜尋中加入更多上下文的訊息，例如輸入 `star`：

  - 若與咖啡相關：建議 `starbucks`
 
  - 與電影相關：建議 `star wars`

## 如何實作 Context Suggester ?

- 可以定義兩種類型的 context

  - `Category`：任意的 string

  - `Geo`：地理位置訊息

- 實作 Context Suggester 的步驟：

  - 設定 mapping

  - 將數據進行索引，並且為每個 document 加入 context 訊息(這樣就可以知道每個 document 跟什麼樣的 context 有關係)

  - 結合 context 進行 suggestion 查詢

以下是一個簡單的範例：

```json
DELETE comments
//新增 index & 設定 mapping
PUT comments
PUT comments/_mapping
{
  "properties": {
    "comment_autocomplete":{
      //基於 completion suggester 擴充而來
      "type": "completion",
      //設定 context，指定類型為 category，並設定 context field "comment_category"
      "contexts":[{
        "type":"category",
        "name":"comment_category"
      }]
    }
  }
}

//新增 movie category 相關的 document
POST comments/_doc
{
  "comment":"I love the star war /movies",
  "comment_autocomplete":{
    "input":["star wars"],
    "contexts":{
      "comment_category":"movies"
    }
  }
}

//新增 coffee category 相關的 document
POST comments/_doc
{
  "comment":"Where can I find a Starbucks",
  "comment_autocomplete":{
    "input":["starbucks"],
    "contexts":{
      "comment_category":"coffee"
    }
  }
}

//指定 category 來取得建議的關鍵字
POST comments/_search
{
  "suggest": {
    "MY_SUGGESTION": {
      "prefix": "sta",
      "completion":{
        "field":"comment_autocomplete",
        "contexts":{
          "comment_category":"coffee"
          //可以嘗試把 category 改成 /movies 再試試看
          //"comment_category":"movies"
        }
      }
    }
  }
}
```


## 關於 Precision(精準度) & Recall(招回率)

以 `Precision(精準度)` 來看： (**盡可能返回較少的無關 document**)

> Completion > Phrase > Term

以 `Recall(招回率)` 來看： (**盡量返回較多的相關 document**)

> Term > Phrase > Completion

以 performance 來看：

> Completion > Phrase > Term



Cross Cluster Search
====================

- [官方參考文件](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cross-cluster-search.html)

## 水平擴展的問題

- 若只有 single cluster，水平擴展是不能無限增加節點數的

> 當 cluster metadata(node, index, cluster status) 過大時，會導致更新壓力變大，單一個 active master 會成為效能的瓶頸，導致整個 cluster 無法正常工作

- 以前透過 **tribe node** 實現 cross node search，但因為問題不少，因此 Elasticsearch 5.3 已經沒有使用 tribe node，而是改用 `Cross Cluster Search` 功能來取代

## Cross Cluster Search

- 允許任何節點扮演 federated node，以輕量的方式處理搜尋請求

- 不需要以 client node 的形式加入其他的 cluster

