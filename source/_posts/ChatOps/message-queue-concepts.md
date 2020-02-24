---
layout: post
title:  "Message Queue 簡介(以 RabbitMQ 為例)"
description: "此篇文章簡單介紹什麼是 Message Queue? 到底這類型的元件有什麼特性? 可以為系統帶來什麼優點? 以及其相關的特性 & 運作方式 (以 RabbitMQ 為例)"
date: 2020-02-12 17:20:00
published: true
comments: true
categories:
  - Message Queue
tags:
  - Message Queue
  - RabbitMQ
---


Preface
=======

以下內容是學習線上課程 [RabbitMQ and Messaging Concepts](https://www.udemy.com/course/rabbitmq-and-messaging-concepts/) 時所留下的學習筆記。

由於目前 Message Queue 在系統架設設計時很常出現，到底這類型的元件有什麼特性? 可以為系統帶來什麼優點? 以及其相關的特性 & 運作方式 (以 RabbitMQ 為例)。以下的部份會進行說明解釋。


Application 整合模式
===================

在一個系統中，一般不會只有一隻程式在運作，而是會有多隻程式同時負責各種不同的任務，而程式之間難免會有互相傳遞資料進行處理的需求，而這類的需求，以下都統稱為 applcation 的整合。

而一般常見的 application 整合方式大概可以分為以下幾種：

## Filed Based Integration

![File Based Integration](/blog/images/middleware/message-queue_file-based-integration.png)

- source application 會根據需要處理的工作，產生新的檔案到特定的路徑

- 其他的 process application 則會是一直監控該路徑有沒有新檔案，有新的檔案則取出進行處理

## Shared Database Integration

![Shared Database Integration](/blog/images/middleware/message-queue_shared-db-integration.png)

- source application 收到新任務時，或將資訊寫入 DB table 中

- processor application 則是持續監控 DB 中的特定 table，若有新紀錄則取出進行處理

- processor application 處理完工作後可能會將狀態回寫到 DB 中

## Direct Connection Integration

![Direct Connection Integration](/blog/images/middleware/message-queue_direct-connection-integration.png)

- source application 直接傳遞訊息給 processor application

- 可能透過 TCP/IP 或是 named pipe connection 的方式傳遞資料

- 不限傳遞的資料格式，連線兩端的 application 傳遞的資料格式可自訂，可能是純文字、XML or JSON

## 透過 Message Broker 以非同步的方式傳遞訊息Asynchronous Messaging

![Message Broker Based Integration](/blog/images/middleware/message-queue_message-broker-integration.png)

- 不限傳遞的資料格式

- 需要額外的 Message Queue middleware 的協助，也有可能被稱為 Message Broker 或是 Message Bus

- Message Broker 搜到訊息後(來自 producer application )會轉發給 consumer application

> 從上面看得出來，其實透過 Message Broker 的方式是以上很多不同方式改良後所產生的結果



使用 Message Queue 有什麼優點?
============================

了解 Message Queue(Broker) 的簡單架構後，那接下來的問題可能是：到底在系統中加入 Message Queue，具體可以帶來哪些好處 or 優點呢?

以下列出幾項說明：(`publisher` 為訊息傳送者，`consumer` 為訊息接收者)

- 將 publisher & consumer 進行 decouple 了，因此程式開發人員可以各自專心負責規模較小 & 單純的程式開發工作

- publisher & consumer 不需要知道雙方的實際的位置(例如：IP address)，只要將資料往 message queue 送就好

- 即使 consumer 短暫的無法提供服務也沒關係，message queue 可以將資料暫存起來，等待 consumer 重新上線時再送過去

- 比起持續 polling 的方式相對有效率的多

- 提供了一個可靠的方式，讓`訊息傳遞` & `工作處理`兩件事情可以用非同步的方式進行

- 當單一 consumer 不足以完成所有工作時，可以很容易的增加 consumer 數量進行水平擴展

從上面的優點可以看出，加入了 Message Queue，對於整體系統中各個不同元件的解耦很有幫助，同時了也帶來了水平擴展的可能性。(當然這部份也會牽涉到系統流程面上的設計，並非只透過系統架構就可以完成)


使用場景範例
==========

以下就透過兩個範例說明，介紹 Message Queue 在整體系統中可以作為什麼樣的角色。

## Case 1

![Message Queue Case 1](/blog/images/middleware/message-queue_case-1.png)

從上圖可以看出，當 product 有任何更動時，需要後端資料庫的 search index 時，相關的資訊會先傳進 message queue，然後會有其他 worker(consumer) 接收進行處理。

## Case 2

![Message Queue Case 2](/blog/images/middleware/message-queue_case-2.png)

在一個電子商務網站，可能會因為以下不同的理由，以`非同步`的方式寄送 email 給會員：

- 驗證 E-Mail address

- 重設密碼

- 訂單確認

- 促銷活動



Messaging System 中的標準組成元素
==============================

了解 Message Queue 的功能之後，接著說明在一個 Messaging System 會包含的標準元素：

- **Message**：這是最主要的部份，簡單來說 **message 就是要從一個 application 傳遞到另一個 application 的資料**，可以是很多形式，例如：command、query 或是任何事件資訊；而每個 message 會包含兩個部份，分別是 `routing information` & `payload(實際資料)`。

- **Producer/Publisher**：producer 是產生 message 並將其傳到 message broker

- **Consumer/Receiver**：收取來自 message broker 的 message 並進行處理

- **Message Queue**：是個存放來自 producer 的 message list，通常一個 messaging system 中會有多個 queue，而每個 queue 都會有一個識別名稱

- **Message Broker**：將 message 從傳送者轉發到 receiver 的一個中間者

- **Router/Exchange**：根據軟體設定，決定 message 傳遞的路徑，將不同的訊息轉發進不同的 queue 中
> Routing 的概念在 RabbitMQ 中以 `Exchange` 來表示

- **Connection**：真實的 TCP 連接，像是 producer & message broker 之間的連接，或是 message broker 與 consumer 之間的連接

- **Channel**：在真實的 TCP 連接中定義出來的 virtual connection，這才是實際上 producer/consumer 與 message broker 之間的連接改念

- **Binding**：Binding 定義了 `Exchange` & `Queue` 之間的關係以及訊息 routing 的設定，可能還包含了一些 filter 的設定
![Message Queue Case 2](/blog/images/middleware/message-queue_concept-binding.png)

 
細部探究 Message, Queue & Exchange 的屬性資訊
==========================================

了解了上面關於 Messaging System 的標準組成元素之後，可以大致了解一個 Message Queue 大概是如何與外部 application 進行溝通運作的；但如果要更探究 Message Queue 內部的運作細節，可以從標準組成元素中的屬性(attribute)來進行了解。

> 這部份的學習會以 [RabbitMQ](https://www.rabbitmq.com/) 為主，因此以下屬性介紹會以 RabiitMQ 為主，但跟其他主流的 Message Queue 應該不會差異太大

## Message 的屬性

- **Routing Key**：用來決定訊息進入到 Messaging System 後如何被轉發的資訊

- **Headers**：key/value 的資訊集合，可用來作為訊息 routing 之後或是傳遞 publisher 想要傳遞的額外訊息

- **Payload**：實際所要傳遞的資料

- **Publishing TimeStamp**(optional)：publisher 所提供的 timestamp 資訊

- **Expiration**：Message 可停留在 Queue 中的存活時間，超過 expiration 的設定則 message 會視為 dead 而不會傳送

- **Delivery Mode**：會有 `persistent` or `trasient` 兩個選項，而 persistent 會將 message 寫入 disk 中，即使 RabiitMQ 服務重啟，訊息也都不會遺失，而 trasient 則不會

- **Priority**：message 的優先權(0~255, 這個需要 Queue 支援才可以)

- **Message ID**(optional)：由 publisher 給入，用來識別 message 的 ID 資訊

- **Correlation ID**(optional)：在 RPC 場景中用來匹配 request & response 用的資訊

- **Replay To**(optional)：在 request-response 場景中會使用到的 exchange 或是 queue 的名稱


## Queue 的屬性

- **Name**：唯一的名稱，最多 255 個字元的 UTF-8 字串

- **Durable**：指定在 RabbitMQ 重啟後保留 or 移除 Queue 的依據

- **Auto Delete**：沒有任何訂閱者的情況下是否自動移除

- **Exclusive**：只服務特定一個 connection，一旦該 connection 斷掉就會移除 Queue

- **Max Length**：最多可以停留在 Queue 中的 message 數量，可以指定若超過會從最舊的訊息開始移除或是拒絕新的 message 進入

- **Max Priority**：priority 可設定的最大值

- **Message TTL**：存入到 Queue 中的 Message 的 TTL(若是 Message & Queue 都有 TTL 的設定，會以較低的為主)

- **Dead-letter Exchange**：可用來指定過期 or 被丟棄的 message 被自動傳送到某一個 exchange

- **Binding Configuration**：Queue & Exchange 之間的關聯資訊 (每個 Queue 都必須與某個 Exchange 綁定，確保可以從 Exchange 取得 message)


## Exchange 的屬性

- **Name**：Exchange 的名稱(必須唯一)

- **Type**：Exchange 的運作類型，會是 `Fanout`, `Direct`, `Topic`, `Headers` 四種之一 

- **Durability**：與 Queue Durability 相同，用來決定在 RabbitMQ 服務重啟後會不會依然存在

- **Auto Delete**：設定是否在沒有任何 Queue 綁定的情況下，就自動移除

- **Internal**：若設定為 Internal，表示僅能接收來自其他 exchange 的 message，無法接收來自 publisher 的 message

- **Alternate Exchange**：指定無法 route 的 message 的去向

- **Other Arguments(x-arguments)**：Exchange 其他的 metadata，可能會用於其他的 plugin 中


Exchange
========

要切入 RabbitMQ 之前，`Exchange` 的概念是必須先了解的，以下是 RabbitMQ 的內部架構圖： 

![RabbitMQ Exchange](/blog/images/middleware/message-queue_concept-binding.png)

- Exchange 是 RabbitMQ 系統中負責轉發訊息的元件

- Message Producer 無法將訊息直接傳到 Queue 中，在 RabbitMQ 中訊息的第一個進入點是 Exchange

- 實際儲存訊息的 Queue 會根據使用者的設定，與不同的 Exchange 進行綁定

- 當 Exchange 收到訊息後，就會轉發到與其綁定的 Queue (可能 0 到多個不等)

- Exchange 僅能將訊息轉發到與其綁定的 Queue 上

- Exchange 有四種轉發模式，分別是 `Fanout`, `Direct`, `Topic`, `Headers`

- 至少會有一個預設 Exchange 存在於 RabbitMQ 系統中，稱為 **default exchange**，轉發的模式為 `direct`；每個新建立的 Queue，若是沒指定 exchane 資訊，就會與預設的綁定


總結
===

透過上面關於 Messaging System 的標準組成元素說明，可以了解到一個 Message Queue 在整體系統上所扮演的角色；加上內部組件的屬性(attribute)說明，也大致可以推敲出一個 Messaging System(以 RabbitMQ 為主) 內部運作的狀況。

因此在實際使用上，我們可以利用 Message Queue 有效的將 application 進行解耦，讓大問題拆分為小問題，並且讓 developer 可以專注在特定的系統功能上，讓 application 面對的，就是 Message Queue 而已。


References
==========

- [Free RabbitMQ Tutorial - RabbitMQ and Messaging Concepts | Udemy](https://www.udemy.com/course/rabbitmq-and-messaging-concepts)

- [Documentation: Table of Contents — RabbitMQ](https://www.rabbitmq.com/documentation.html)