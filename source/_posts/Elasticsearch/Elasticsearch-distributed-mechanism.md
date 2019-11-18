---
layout: post
title:  "[Elasticsearch] 分散式特性 & 分散式搜尋的機制"
description: "此篇文章是在極客時間學習 Elasticsearch 課程時留下的一些學習筆記，主要內容包含 Elasticsearch 分散式特性 & 分散式搜尋的機制"
date: 2019-11-18 21:10:00
published: true
comments: true
categories:
  - Elasticsearch
tags:
  - Elasticsearch
---


集群分佈式模型及選主與腦裂問題
=========================

## 分散式特性

Elasticsearch 的分散式架構帶來以下優點：

- 可以水平擴展儲存空間，支援 PB 等級的數據儲存

- 提高系統的可用性，當某些節點停止服務時，整個 cluster 的服務不會受影響

關於設定 Elasticsearch cluster：

- 不同的 cluster 透過不同的名字區分，預設為 `elasticsearch`

- cluster name 可以透過設定檔修改，也可以在啟動指令中指定 `-E cluster.name=[CLUSTER_NAME]` 進行設定


## Node

- Node 就是一個 Elasticsearch 的 Java process；基本上一台機器上可以同時運行多個 Elasticsearch process，但 production 使用建議還是只要一個就好

- 每個 node 都有名稱，可透過設定檔配置，也可以在啟動時透過 `-E node.name=[NODE_NAME]` 進行設定

- 每個 node 啟動之後都會分配一個 UID，並儲存在 `/data` 目錄下


## Coordinating Node

- 處理 request 的 node 稱為 **Coordinating Node**，其功能是將 request 轉發到正確的 node 上(例如：建立 index 的 request 要轉發給 master node)

- 所有的 node 都預設是 Coordinating Node

- 可以透過將其他 node type 都設定為 `false`，這樣就可以讓特定的 node 變成 dedicated coordinating node


## Data Node

- 可以保存數據資料的 node，每個 node 啟動後都會預設是 data node，可以透過設定 `node.data: false` 停用 data node 功能

- 用來保存分片數據資料，實現數據資料的 scalibility (由 master node 決定如何把分片分發到不同的 data node 上)

- 透過增加 data node 可以解決數據水平擴展 & 解決單點故障導致數據資料遺失的問題


## Master Node

- master node 用來處理以下工作：

  - 處理建立/刪除 index 的 request，並實際執行

  - 決定每個 shard 要被分配到哪個 data node 上

  - 維護 & 更新 cluster state

- master node 配置的 best practice：

  - master node 很重要，佈署上要考慮避免單點故障的狀況發生

  - 為 cluster 設置多個 master node，且必須是 dedicated master node


- Master Eligible Node & 選舉流程

  - cluster 中可以設定多個 master eligible node，當 master node 發生問題時，這些 master eligible node 就會開始選舉流程，選出下一個 msater node

  - 每個 node 啟動時就預設是一個 master eligible node，可以透過設定 `node.master: false` 取消此預設設定

  - 當 cluster 中的第一個 master eligible node 啟動時，就會把自己選舉為 master node


## Cluster State

- cluster state 維護了一個 cluster 中必要的訊息，包含以下內容：

  - 所有的 node 資訊

  - 所有的 index & 相對應的 mapping/setting 配置

  - shard 的路由資訊

- 每個 node 上都保存了 cluster state

- 只有 master 才可以修改 cluster state 並負責同步給其他 node (若是每個 node 都可以修改 cluster state，很有可能會導致 cluster state 不一致的狀況發生)


## 配置不同的 node type

以下是要設定不同的 node type 時所會用到的設定參數：

| Node Type | 配置參數 | 預設值 |
|-----------|---------|-------|
| Master Eligible | `node.master` | true |
| Data | `node.data` | true |
| Ingest | `node.ingest` | true |
| Dedicated Coordinating | `無` | 設置上面三個參數皆為 false |
| Machine Learning | `node.ml` | true (需要 enable x-pack) |



Shard & Cluster 的故障轉移
========================

## Primary Shard (提昇系統儲存容量)

- shard 是 Elasticsearch 分散式儲存的基礎，包含 primary shard & replica shard

- primary shard 功能是將一份被索引的數據，分散到多個 data node 上存放，實現儲存方面的水平擴展

- primary shard 在建立索引時就會指定，後續是無法修改的，若要修改就必須要進行 reindex


## Replica Shard (提高數據資料可用性)

- replica shard 用來提供數據資料可用性，當 primary shard 遺失時，replica shard 就可以被 promote 成 primary shard 來保持資料完整性

- replica shard 數量可以動態調整，讓每個 data node 上都有完整的數據資料

- replica shard 可以一定程度的提高讀取的效能

> 若不設定 replica shard，一旦有 data node 故障導致 primary shard 遺失，數據資料可能就無法恢復了

> ES 7.0 開始，primary shard 預設為 `1`，replica shard 預設為 `0`


## Shard 的規劃 & 設定

- 若是 primary shard 數量設定太小，且遇到 index 資料增加很快時，cluster 無法通過增加 ndoe 數量對 index 進行數據擴展

- 若 primary shard 數量設定太大，可能導致每個 shard 容量很小，讓一個 data node 上有過多 shard 而影響效能

- replica shard 若設定過多，會降低 cluster 整體的寫入效能

> replica shard 必須和 primary shard 被分配在不同的 data node 上；但所有的 primary shard 可以在同一個 data node 上


## cluster & shard 設定情境 & 範例

### single node cluster

![Elasticsearch - single node cluster](/blog/images/Elasticsearch/es_cluster-statue-1.png)

- 由於 cluster 中只有單一個 node，因此全部都會被 primary shard 佔據

- replica shard 無法被分配，因此 cluster 健康狀態為黃色

### 增加第二個 node

![Elasticsearch - add second node to cluster](/blog/images/Elasticsearch/es_cluster-statue-2.png)

- 增加第二個 node，因此 replica shard 就可以被正常分配

- cluster 健康狀態就會轉為綠色

- 目前 cluster 已經具備 failover 的能力了

### 增加第三個 node

![Elasticsearch - add third node to cluster](/blog/images/Elasticsearch/es_cluster-statue-3.png)

- 此時 primary shard & replica shard 都會再重新分配，力求資料可以平均分散

- 由 master node 決定每個 shard 會被分配到那一個 data node

- 透過持續增加 node，可以提高 cluster 的計算能力

### Failover

![Elasticsearch - cluster failover](/blog/images/Elasticsearch/es_cluster-failover-1.png)

> 1 index, 3 primary shard, 1 replica shard

- 上面的部份是正常的時候，下面是表示 node 1(master node) 發生故障

- node 2 透過選舉的過程變成了 master node

- node 1 上的 P0 跟 R1 已經不可用

- node 3 上的 R0 會提昇 P0 成用來取代 Node 1 上的 P0

- 為了要滿足 replica shard 的設定，node 2(目前的 master) 重新將 R0 & R1 分配到剩餘的 data node 上

- cluster 狀態重新變成綠色

> 若是擔心 reboot 機器造成 failover 動作開始執行，可以設定將 replication 延遲一段時間後再執行，避免無謂的 data copy 動作


## 值得一看的 Q&A 資料

1：選主的過程中可能存在問題的場景？
> 選主的過程應該很短，這個期間，如果有創建index或者分片reallocation有可能會出錯。

2：故障轉移期間可能會出現問題的場景？
> 故障轉移期間，如果只是黃色變綠，應該不影響讀寫，因為副本會提升為主分片。集群變紅，代表有主分片丟失，這個時候會影響讀寫。

3： 故障轉移，數據重新分配，消耗性能的避免方式？
> 例如一個主分片不可用了。只要設置了副本分片，其中一個副本分片立即會將自己提升為主分片。同時會將自己的數據分配到一個新的replica上，有時候，我們只是重啟一台機器，可以讓這個reallocation的動作延遲一段時間再做，從而避免無謂的數據拷貝。

4：故障轉移可能存在數據丟失的場景嘛？
> node如果丟失，如果沒有落盤。就有丟失的可能。如果節點重新回來，會從translog中恢覆沒有寫入的數據



Document 分散式儲存
==================

## document 儲存在 shard 中

- document 會儲存在特定的 primary shard & replica shard 中，例如 document(id=1) 儲存在 P0 & R0 shard 上

- 將 document 進行分散式儲存的可能解決方案：

  - random / round robin 的方式，但此方法在 shard 數量大時，需要多次查詢才能找到 document

  - 維護 document & shard 之間的 mapping 關係，但當 document 數量大時，維護成本就會變得很高

  - 透過即時運算，以 document id 為基礎，計算出要去那一個 shard 儲存 & 取得 document


## document 到 shard 的路由計算方式

> ` shard = hash(_routing) % number_of_primary_shard`

- hash algorithm 確保 document 均勻的分散到 primary shard 中

- 預設的 `_routing` value 是 document id

- 可以自行指定 routing value，藉此讓 document 都分配到特定的 shard 上

- 也是因為上述路由設計機制的原因，導致於設定 index settings 之後，primary shard 數量無法隨意變更


## 更新 document 流程 

![Elasticsearch - flow of updating a document](/blog/images/Elasticsearch/es_distribute-shard-update.png)

1. 使用者送出一個 update document request 到 master node(同時也是 coordinating node)

2. coordinating node 透過 hash 計算出 document 存放的 shard 在哪裡

3. 將 update request 轉發到正確的 data node 上

4. data node 取得 update request，會先將 document 刪除

5. 再重新索引新的 document，完成標準的 update 操作

6. 將更新成功訊息回傳給 coordinating node

7. coordinating node 回傳操作成功訊息給使用者


## 刪除 document 流程

![Elasticsearch - flow of deleting a document](/blog/images/Elasticsearch/es_distribute-shard-delete.png)

1. 使用者送出一個 delete document request 到 master node(同時也是 coordinating node)

2. coordinating node 透過 hash 計算出 document 存放的 primary shard 在哪裡，並將 request 轉發到正確的 data node 上

3. data node 刪除 shard 中指定的 document

4. data node 送出 delete replica request 到儲存 replica shard 的 data node 上 (透過每個 node 上的 cluster state 才可以找到 routing 資訊)

5. replica shard 中的資料被刪除，data node 會回傳成功訊息

6. 儲存 primary shard 的 data node 將刪除成功的訊息回傳給 coordinating node

7. coordinating node 回傳操作成功訊息給使用者


## 值得一看的 Q&A 資料

- 請問，視頻中更新和刪除文檔的請求，首先會發送到master節點嗎，還是通過前置的負載均衡工具分發到某一個節點？

> 視頻中發送到9200，我也沒在開發環境中指定dedicated的節點。所以這個節點既是master也是data，當然肯定也是coordinating節點。

> 在生產環境，你可以設置dedicate的 coordinate節點，發查詢到這些節點。不建議直接發送請求到master節點，雖然也會工作，但是大量請求發送到master，會有潛在的性能問題




Shard & Life cycle
==================

## shard 內部原理

- shard 是 ES 中最小的工作單元

- shard 是一個 Lucene 的 index

關於一些 Elasticsearch 的相關問題：

- Elasticsearch 的搜尋是如何作到接近即時的 ?

- Elasticsearch 如何確保臨時的停電不會造成數據資料遺失 ?

- 為什麼刪除 document 後，不會馬上釋放空間 ?


## Inverted Index 的不可變動性

- Inverted Index 使用 immutable design，一旦產生出來就無法變更

- 然而不可變動的特性會帶來以下好處：

  - 不需要考慮同時多個 document 寫入的問題，因此避免了 lock 機制所帶來的效能問題

  - 一旦資料進入到 cache，就會留在裡面；只要 cache 夠大，大部分 request 就不會有 dick access，藉此大幅提昇讀取性能
  
  - cache 容易產生 & 維護

  - 數據資料可以被壓縮

- 不可變動的特性帶來的缺點 => 若需要讓一個新的 document 可以被搜尋，需要重建 index


## Index Refresh

![Elasticsearch - Index Refresh](/blog/images/Elasticsearch/es_index-refresh.png)

- Elasticsearch 寫入 document 時，會先寫入稱為 `Index Buffer` 的儲存空間

- 到特定時間點(or 滿足特定條件)時，就會將 Index Buffer 中的內容寫入 `Segment`，而這寫入的動作就稱為 `Refresh`，但預設不會執行 **fsync** 操作
> 預設一秒一次，可以透過設定 `index.refresh_interval` 進行調整

- 當 document 被 refresh 進入到 segment 之後，就可以被搜尋到了

- 若系統有大量的數據資料寫入，就會產生很多 segment

- 當 Index Buffer 被佔滿時，也會觸發 refresh 動作，預設值是 JVM 的 10%


## Transaction Log

![Elasticsearch - Transaction Log](/blog/images/Elasticsearch/es_transaction-log.png)

- segment 寫入磁碟的過程相對耗時，因此藉由 cache，在進行 refresh 時先將 segment 寫入 cache 以開放查詢

- 但使用 cache 可能就會有數據資料遺失的問題，因此為了保證資料不會遺失，就有了 transaction log 的設計

- 將 document 進行索引時，同時也會寫入 transaction log，且預設都會寫入磁碟中
> 每個 shard 都會有對應的 transaction log

- Elasticsearch 進行 refresh 時，index buffer 會被清空，但 transaction log 則不會


## Flush

![Elasticsearch - Flush](/blog/images/Elasticsearch/es_flush.png)

- 執行 refresh，將 index buffer 清空

- 執行 fsync，將在 cache 中的 segment 全部寫入磁碟中

- 清空 transaction log

- 由於操作對資源消耗相對大，因此預設 30 分鐘執行一次

- 當 transaction log 滿了(預設為 512 MB)，也會觸發 flush 的操作


## Merge

- 當 flush 工作陸陸續續完成後，segment 上的資料會寫入磁碟中，因此就會造成很多空的 segment，而 merge 就可以協助定期將 segment 合併
> 可以減少 segment 數量 & 將被刪除的 document 從磁碟中真正的移除掉

- Elasticsearch 會定期自動執行 merge 工作，但若是要強制執行，可以呼叫 `POST [INDEX_NAME]/_forcemerge` 執行


## 值得一看的 Q&A 資料

1. 客戶端發起數據寫入請求，對你寫的這條數據根據_routing規則選擇發給哪個Shard。
  - 確認Index Request中是否設置了使用哪個Filed的值作為路由參數，
  - 如果沒有設置，則使用Mapping中的配置，
  - 如果mapping中也沒有配置，則使用_id作為路由參數，然後通過_routing的Hash值選擇出Shard，最後從集群的Meta中找出出該Shard的Primary節點。

2. 寫入請求到達Shard後，先把數據寫入到內存（buffer）中，同時會寫入一條日誌到translog日誌文件中去。
  - 當寫入請求到shard後，首先是寫Lucene，其實就是創建索引。
  - 索引創建好後並不是馬上生成segment，這個時候索引數據還在緩存中，這裡的緩存是lucene的緩存，並非Elasticsearch緩存，lucene緩存中的數據是不可被查詢的。

3. 執行refresh操作：從內存buffer中將數據寫入os cache(操作系統的內存)，產生一個segment file文件，buffer清空。
  - 寫入os cache的同時，建立倒排索引，這時數據就可以供客戶端進行訪問了。
  - 默認是每隔1秒refresh一次的，所以es是准實時的，因為寫入的數據1秒之後才能被看到。
  - buffer內存佔滿的時候也會執行refresh操作，buffer默認值是JVM內存的10%。
  - 通過es的restful api或者java api，手動執行一次refresh操作，就是手動將buffer中的數據刷入os cache中，讓數據立馬就可以被搜索到。
  - 若要優化索引速度, 而不注重實時性, 可以降低刷新頻率。

4. translog會每隔5秒或者在一個變更請求完成之後，將translog從緩存刷入磁盤。
  - translog是存儲在os cache中，每個分片有一個，如果節點宕機會有5秒數據丟失，但是性能比較好，最多丟5秒的數據。。
  - 可以將translog設置成每次寫操作必須是直接fsync到磁盤，但是性能會差很多。
  - 可以通過配置增加transLog刷磁盤的頻率來增加數據可靠性，最小可配置100ms，但不建議這麼做，因為這會對性能有非常大的影響。

5. 每30分鐘或者當tanslog的大小達到512M時候，就會執行commit操作（flush操作），將os cache中所有的數據全以segment file的形式，持久到磁盤上去。
  - 第一步，就是將buffer中現有數據refresh到os cache中去。
  - 清空buffer 然後強行將os cache中所有的數據全都一個一個的通過segmentfile的形式，持久到磁盤上去。
  - 將commit point這個文件更新到磁盤中，每個Shard都有一個提交點(commit point), 其中保存了當前Shard成功寫入磁盤的所有segment。
  - 把translog文件刪掉清空，再開一個空的translog文件。
  - flush參數設置：
  - index.translog.flush_threshold_period:
  - index.translog.flush_threshold_size:
  - #控制每收到多少條數據後flush一次
  - index.translog.flush_threshold_ops:

6. Segment的merge操作：
  - 隨著時間，磁盤上的segment越來越多，需要定期進行合併。
  - Es和Lucene 會自動進行merge操作，合併segment和刪除已經刪除的文檔。
  - 我們可以手動進行merge：POST index/_forcemerge。一般不需要，這是一個比較消耗資源的操作

> 當數據從hot移動到warm，官方建議手工執行一下_forcemerge

![Elasticsearch - Q&A Chap40 - 1](/blog/images/Elasticsearch/es_qa_chap40-1.png)

![Elasticsearch - Q&A Chap40 - 2](/blog/images/Elasticsearch/es_qa_chap40-1.png)



剖析 Distributed Search(分佈式查詢)及相關性算分
==========================================

## Distributed Search 的運作機制

- Elasticsearch 的搜尋會分為兩個階段，分別是 `Query` & `Fetch`，也就是 `Query-then-Fetch`


## Query-then-Fetch 範例

以下面的搜尋為例：

![Elasticsearch - Distributed Search](/blog/images/Elasticsearch/es_distrbuted-search.png)

### Query 階段

- 使用者送出 search 到 Elasticsearch，Coordinating Node 會在六個 primary & replica shard 中隨機挑選三個 shard 並送出 request 給 data node

- 被選中的 shard 執行查詢 & 排序，返回 `From + Size` 個**排序後**的 document ID & 排序值給 Coordinating Node
> 上圖中是 P0, P1, P2 被選中 (**目前只有取得 document ID，沒有內容**)

### Fetch 階段

- Coordinating Node 會將 Query 階段中從每個 shard 取得的 document ID 重新排序，並根據 `From` & `Size` 重新選出 document ID list

- 以 multiple GET 的方式，從對應的 shard 取得詳細的 document 資訊


## Query-then-Fetch 的潛在問題

效能問題：

- 每個 shard 需要查詢的 document 數量 = `from` + `size`

- 最後 Coordinating Node 需要處理 `number_of_shard * (from + size)` 數量的 document

- 若是遇到深度分頁的情況，效能會變很差

相關性算分：

- 每個 shard 都基於自己 shard 上的數據進行相關度計算；若是在數據量少，shard 數量越大會導致算分越不準確


## 如何解決算分不準的問題 ?

- 數據資料量不大時，可以將 primary shard 數量設定為 1
> 但若資料足夠大時，只要 document 可以平均分散在多個 shard 上，結果就不會有太大偏差

- 使用 CFS Query Then Fetch

- 在搜尋的 URL 中指定參數 `_search?search_type=dfs_query_then_fetch`

- 這會到各 shard 中蒐集完整的 TF & IDF 的資料，再重新匯總並算分，但這會消耗太多 CPU & memory，不建議使用


## 值得一看的 Q&A 資料

![Elasticsearch - Q&A Chap41 - 1](/blog/images/Elasticsearch/es_qa_chap40-1.png)



排序及 Doc Values & Fielddata
============================

## Sorting

- Elasticsearch 預設會使用相關性算分對結果進行排序

- 可以透過設定 `sorting` 參數決定排序的條件

- 若自己給入 sort 條件時但卻不是指定 `_score`，則算分為 NULL

以下是幾個範例：

```json
//single field 查詢，並指定 "sort"
//回傳的資料在 _score 部份都會以 null 呈現
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"order_date": {"order": "desc"}}
  ]
}

//multiple field 查詢的範例
//因為有指定將 _score 拿來排序，因此 _score 就不會是 null
//查詢中使用 "match_all"，因此所有的算分都會是 1
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }
  },
  "sort": [
    {"order_date": {"order": "desc"}},
    {"_doc":{"order": "asc"}},
    {"_score":{ "order": "desc"}}
  ]
}
```

若想要對 text 類型的 field 做排序：

```json
//這樣的查詢會報錯，預設是無法針對 text 類型的 field 做排序
POST /kibana_sample_data_ecommerce/_search
{
  "size": 5,
  "query": {
    "match_all": {

    }!
  },
  "sort": [
    {"customer_full_name": {"order": "desc"}}
  ]
}

//透過將 field 中的 "fielddata": true 設定打開
//這樣就可以對 text 的 field 進行排序
PUT kibana_sample_data_ecommerce/_mapping
{
  "properties": {
    "customer_full_name" : {
          "type" : "text",
          "fielddata": true,
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        }
  }
}
```


## sorting 的過程

- sorting 是針對原始的 field 內容進行，因此 inverted index 無法發揮作用

- 需要使用到 forward index(正排索引)，透過 document ID & field 快速得到 field 原始內容

- 在 Elasticsearch 中有兩種實現 sorting 的方式：

  - Fielddata

  - Doc Values (列式儲存，對 text 類型無效))


## Doc Values v.s. Field data

|  | Doc Values | Field data |
|--|------------|------------|
| 何時建立? | 資料進行索引時，和 inverted index 一起建立 | 搜尋時動態建立 |
| 建立位置 | 磁碟檔案 | JVM Heap |
| 優點 | 避免佔據大量記憶體 | 索引速度快，不佔用額外的磁碟空間 |
| 缺點 | 降低索引速度，佔用額外的磁碟空間 | document 過多時，動態建立的消耗大，佔用過多的 JVM Heap |
| 預設? | ES 2.x 之後 | ES 1.x 及之前 |

> 一般不會開啟 fielddata，因為對 text field 排序通常沒什麼意義，通常僅有在 aggregation 需求時才會開啟 fielddata 的設定


## 關閉 Doc Values

透過以下的語法，可以關閉 Doc Values：

```json
PUT test_keyword/_mapping
{
  "properties": {
    "user_name":{
      "type": "keyword",
      "doc_values":false
    }
  }
}
```

- Doc Values 預設啟用，可以透過 mapping 設定關閉，優點可以增加索引速度 & 減少硬碟空間的消耗

- 如果要重新打開，需要重建 index

- 若是對於不需要進行 sorting or aggregation 的欄位，可以關閉該欄位的 Doc Values



分頁與遍歷：From, Size, Search After & Scroll API
===============================================

## From & Size

- Elasticsearch 預設對搜尋回傳 10 筆紀錄，`From` 是 document 開始位置，`Size` 是期望取得 document 的總數


## 分散式系統中 Deep Pagination(深度分頁)的問題

![Elasticsearch - Deep Pagination](/blog/images/Elasticsearch/es_deep-pagination-1.png)

- Elasticsearch 是個分散式系統，資料會分佈在多個 shard 中

- 但假設查詢的 `From=990` & `Size=10` 時，Elasticsearch 會進行以下操作：

  - 在每個 shard 中取得 1000 個 document，然後 coordinating node 會整合所有結果，最後透過排序選取前 1000 個 document

  - 頁數越深，佔用的 memory 越多，而 Elasticsearch 預設限制到 10000 個 document(可透過修改 `index.max_result_window` 來調整)


## 使用 search_after 避免 Deep Pagination(深度分頁)問題

為了避免 deep pagination 的問題，Elasticsearch 提供了 `search_after` 的功能，以下是一個簡單範例：

```json
//透過 "search_after" 可以明確指定從哪裡開始取得資料
//因此也就避免了 deep pagination 的問題
POST users/_search
{
    "size": 1,
    "query": {
        "match_all": {}
    },
    "search_after": [10, "ZQ0vYGsBrR8X3IP75QqX"],
    "sort": [
        {"age": "desc"} ,
        {"_id": "asc"}    
    ]
}
```

- 然而 search_after 會有以下兩項限制：

  - 不支援指定頁數 (也就是表示不能指定 `From`)

  - 只能往下翻頁，無法往回

- 搜尋時需要指定上一次的 `sort` value，如此一來就可以一次一次的搜尋結果進行翻頁

- `search_after` 是透過唯一排序值的定位方式，將每次要處理的 document 數量都控制在 10(可自訂 `size`)


## Scroll API

Scroll API 也是為了解決深度搜尋的另外一種方式，實踐的方法類似 `search_after`，作法如下：

- 搜尋時建立一個快照，作為後續繼續快速搜尋之用，但如果有新的數據資料寫入後，就沒辦法被查詢到了

- 每次查詢時，要將上次查詢結果中的 scroll id 拿來使用，才可以正確的繼續往下搜尋


## 不同的搜尋類型 & 使用場景

- **一般搜尋**：需要接近即時的取得最新的部份資料，例如：查詢最新訂單

- **Scroll**：需要全部 document，但過程中的分頁速度要快，例如：導出全部數據資料

- **Pagination**：使用 `from` & `size`，如果要處理 deep pagination 的問題，則使用 `search_after`



處理 Concurrent 讀寫操作
======================

## Concurrent control 的重要性

以下是一個常見的範例，假設在沒有做 concurrent control 的狀況下，下圖的情況就有可能發生，導致實際庫存錯誤：

![Elasticsearch - Concurrent Read & Write operation](/blog/images/Elasticsearch/es_concurrent-1.png)

- 兩個 web process 同時更新某個 document，如果沒有對 concurrent control，可能會導致更新的資料遺失

- 悲觀鎖定(Pessimistic Locking)：假設有變更衝突的可能性，因此對資源更新的時候會進行 lock，防止衝突，例如：DB row lock

- 樂觀鎖定(Optimistic Locking)
  
  - 假設更新衝突不會發生，因此不會阻止正在嘗試更新的操作

  - 但如果數據資料在讀寫中被修改，更新會失敗

  - 通常由應用程式端來解決這樣的衝突，例如：重新嘗試更新、使用新資料、或是回應錯誤訊息給使用者

  - Elasticsearch 採用的即是 Optimistic Locking


## Elasticsearch 如何作到 Optimistic Locking 下的 Concurrent Control ?

![Elasticsearch - Optimistic Locking](/blog/images/Elasticsearch/es_optimistic-locking-1.png)

- Elasticsearch 中的 document 是不可變更的；更新 document 會將原本的 document 標記為刪除，並增加一個全新的 document，同時 document 的 `version` field 加 1

- 內部版本控制，可透過 `if_seq_no` + `if_primary_term` 來處理

- 外部版本(Elasticsearch 只是用來作為類似 DB backup)時，可使用 `version` + `version_type=external` 來處理

以下是個簡單範例：

```json
DELETE products

//新增 document
PUT products/_doc/1
{
  "title":"iphone",
  "count":100
}

//取得新增的資料內容
//可以看到 "_seq_no" : 0 & "_primary_term" : 1
GET products/_doc/1

//進行更新操作，並指定 if_seq_no=0 & if_primary_term=1
//可以成功更新
PUT products/_doc/1?if_seq_no=1&if_primary_term=1
{
  "title":"iphone",
  "count":100
}

//再一次進行更新操作，同樣指定 if_seq_no=0 & if_primary_term=1
//這次更新就會失敗了，會出現 'version_conflict_engine_exception'
PUT products/_doc/1?if_seq_no=1&if_primary_term=1
{
  "title":"iphone",
  "count":102
}

//也可透過搭配 version & version_type=external 的方式更新
//這一次可以更新成功
PUT products/_doc/1?version=30000&version_type=external
{
  "title":"iphone",
  "count":100
}

//同樣的 version 再進行更新，就會失敗了
//出現 version_conflict_engine_exception
PUT products/_doc/1?version=30000&version_type=external
{
  "title":"iphone",
  "count":102
}
```



References
==========

- [持久化变更 | Elasticsearch: 权威指南 | Elastic](https://www.elastic.co/guide/cn/elasticsearch/guide/current/translog.html)

- [Guide to Refresh and Flush Operations in Elasticsearch](https://qbox.io/blog/refresh-flush-operations-elasticsearch-guide)