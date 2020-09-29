---
layout: post
title:  "[架構設計] 可擴展架構設計"
description: "此篇文章是極客時間\"從 0 開始學架構\"課程時所留下的學習筆記，主要內容為可擴展的架構設計、Microservice 架構設計 & 相關的 Best Practice"
date: 2020-09-26 14:20:00
published: true
comments: true
categories:
  - Architecture Design
tags:
  - Architecture Design
---


可擴展架構的基本思想和模式
======================

- 軟體系統與硬體和建築系統最大的差異在於軟體是可擴展的；一個硬體生產出來後就不會再進行改變、一個建築完工後也不會再改變其整體結構

- 軟體系統的魅力 & 困難之處：
  - 魅力：可以通過修改和擴展，不斷地讓軟體系統具備更多的功能和特性，滿足新的需求或者順應技術發展的趨勢
  - 困難：**如何以最小的代價去擴展系統**

- 如何避免擴展時改動範圍太大，是軟體架構可擴展性設計的主要思考點


## 可擴展的基本思想

- 所有的可擴展性架構設計，背後的基本思想都可以總結為一個字：**拆！**

- 拆，就是將原本大一統的系統拆分成多個規模小的部分，擴展時只修改其中一部分即可，無須整個系統到處都改，通過這種方式來減少改動範圍，降低改動風險

- **軟體系統中的"拆"是建設性的**，因此難度要高得多

- 常見的拆分思路有三種：`面向流程拆分`、`面向服務拆分`、`面向功能拆分`

- 從範圍來看，從大到小為：**流程 > 服務 > 功能**


## 可擴展方式

- 合理的拆分，能夠強制保證即使 developer 出錯，出錯的範圍也不會太廣，影響也不會太大

- 不同拆分方式應對擴展時的優勢：
  - 面向流程拆分：擴展時大部分情況只需要修改某一層，少部分情況可能修改關聯的兩層，不會出現所有層都同時要修改
  - 面向服務拆分：對某個服務擴展，或者要增加新的服務時，只需要擴展相關服務即可，無須修改所有的服務
  - 面向功能拆分：對某個功能擴展，或者要增加新的功能時，只需要擴展相關功能即可，無須修改所有的服務

- 典型的可擴展系統架構有：
  - 面向流程拆分：**分層架構**
  - 面向服務拆分：**SOA & Microservice**
  - 面向功能拆分：**微內核架構**



傳統的可擴展架構模式：分層架構和 SOA
===============================

## 分層架構

- 分層(Tier)架構是很常見的架構模式，它也叫 N 層架構，通常情況下，N 至少是 2 層

- 需要保證每個 Tier 之間的差異足夠清晰，邊界足夠明顯，讓人看到架構圖後就能看懂整個架構

- 分層架構之所以能夠較好地支撐系統擴展，本質在於**隔離關注點(separation of concerns)**，每個層中的組件只會處理本層的邏輯

- 分層時要保證每層之間的依賴是穩定的，才能真正支撐快速擴展

- 對於 OS 這類複雜的系統，interface 本身也可以成為獨立的一層

- 特點是層層傳遞，一旦分層確定，**整個業務流程是按照層進行依次傳遞的，不能在層之間進行跳躍**

- 分層結構的這種約束，好處在於強制將分層依賴限定為兩兩依賴，降低了整體系統複雜度


### C/S 架構、B/S 架構

![Client/Server architecture](/blog/images/architecture-design/scalable-arch-cs.png)

- 劃分的對象是**整個業務系統**，劃分的維度是**用戶交互**

- 將和用戶交互的部分獨立為一層，支撐用戶交互的後台作為另外一層

### MVC 架構、MVP 架構

![MVC/MVP architecture](/blog/images/architecture-design/scalable-arch-mvc.png)

- 劃分的對象是**單個業務子系統**，劃分的維度是**職責**

- 將不同的職責劃分到獨立層，但各層的依賴關係比較靈活

### 邏輯分層架構

以 Android OS 為例：
![Business Logic architecture](/blog/images/architecture-design/scalable-arch-logic.png)

- 劃分的對象可以是單個業務子系統，也可以是整個業務系統，劃分的維度也是**職責**

- 邏輯分層架構中的層是自最上層向下依賴的

- 典型的有 OS kernel 架構、TCP/IP 架構


## SOA

- SOA 解決了傳統 IT 系統重複建設和擴展效率低的問題，但其本身也引入了更多的複雜性

- ESB(Enterprise Service Bus) 要完成這麼多協議和資料格式的互相轉換，工作量和複雜度都很大，而且這種轉換是需要耗費大量計算效能的

- 當 ESB 承載的消息太多時，ESB 本身會成為整個系統的效能瓶頸


為了應對傳統 IT 系統存在的問題，SOA 提出了 3 個關鍵概念：

### 服務

- 所有業務功能都是一項**服務**

- 服務就意味著要對外提供開放的能力，當其他系統需要使用這項功能時，無須定製化開發

- 服務可大可小，可簡單也可複雜

### ESB (Enterprise Service Bus)

- ESB 用途是將企業中各個不同的服務連接在一起

- SOA 使用 ESB 來隱藏異質系統對外提供各種不同 interface 的方式，以此來達到服務間高效率的互連互通

### 鬆耦合(Loose Coupling)

- 目的是減少各個服務間的依賴和互相影響

- 要做到完全後向兼容，是一項複雜的任務



深入理解 Microservice 架構：銀彈 or 焦油坑？
=========================

## SOA v.s. Microservice

- SOA 和 Microservice 本質上是兩種不同的架構設計理念，只是在**"服務"**這個點上有交集而已

- SOA 和 Microservice 是兩種不同理念的架構模式，只是應用場景不同而已，並沒有使用哪種架構一定會比較好的問題

- **Small**、**Lightweight**、**Automated**，基本上濃縮了 Microservice 的精華，也是 Microservice 與 SOA 的本質區別所在

具體做法比較：

### 服務粒度

- SOA 的服務粒度要粗一些，而 Microservice 的服務粒度要細一些

### 服務通信

- SOA 採用了 ESB 作為服務間通信的關鍵元件，負責服務定義、Service Routing、消息轉換、消息傳遞，總體上是重量級的實現

- Microservice 推薦使用統一的協議和格式，例如，RESTful、RPC，無須 ESB 這樣的重量級實現

### 服務交付

- SOA 對服務的交付並沒有特殊要求，因為 SOA 更多考慮的是兼容已有的系統

- Microservice 的架構理念要求"快速交付"，相應地要求採取自動化測試、持續集成、自動化部署等敏捷開發相關的最佳實踐

### 應用場景

- SOA 更加適合於龐大、複雜、異質的企業級系統

- Microservice 更加適合於快速、輕量級、基於 Web 的網際網路系統，這類的系統業務變化快，需要快速嘗試、快速交付

- 整體系統的複雜度是隨著 Microservice 數量的增加，同時也呈現指數級的增加


## Microservice的陷阱

- 實施 Microservice 需要避免踩的陷阱，簡單來說：
  - Microservice 拆分過細，過分強調 **"Small"**
  - Microservice 基礎設施不健全，忽略了 **"Automated"**
  - Microservice 並不輕量級，規模大了後，**"Lightweight"** 不再適應

### 服務劃分過細，服務間關係複雜

- 服務劃分過細，單一服務的複雜度確實下降了，但整個系統的複雜度卻上升

- Microservice 將系統內的複雜度轉移為系統間的複雜度

### 服務數量太多，團隊效率急劇下降

- 很多團隊看到 "micro" 後，就覺得必須將服務拆分得很細

### 調用鏈路太長，效能下降

- 由於 Microservice 之間都是通過 HTTP 或者 RPC 調用的，每次調用必須經過網路

- 為了支撐業務請求，可能需要大幅增加硬體，這就導致了硬體成本的大幅上升

### 調用鏈太長，問題定位困難

- 由於 Microservice 數量較多，且故障存在擴散現象，要快速定位到底是哪個 Microservice 故障是一件複雜的事情

### 沒有自動化支撐，無法快速交付

- 如果沒有相應的自動化系統進行支撐，都是靠人工去操作，那麼 Microservice 不但達不到快速交付的目的，甚至還不如一個大而全的系統效率高

### 沒有服務治理，Microservice 數量多了後管理混亂

- 隨著 Microservice 種類和數量越來越多，如果沒有服務治理系統進行支撐，Microservice 提倡的 lightweight 就會變成問題

- 主要問題類型：
  - Service Routing
  - 服務故障隔離
  - 服務註冊和發現


## 討論整理精華

### 1

Microservice 架構其實相當複雜，可以分成好幾個階段理解：

1. 第一階段，Microservice 架構就是去掉了 ESB 的 SOA 架構，只不過是通信的方式和結構變了

2. 第二階段，沒有了 ESB，原本很多由 ESB 完成的事情，轉到服務的提供者和調用者這裡了；此時就需要考慮服務的拆分粒度

3. 第三階段，隨著服務的數量大幅增加，服務的管理越來越困難，此時 DevOps 出現了，為此階段的 Microservice 架構帶入大量的自動化，已經是跟 SOA 架構是完全不同的東西

### 2

大型企業從傳統架構，轉向 Microservice 架構的過程：

1. 建設好基礎設施，RPC、服務治理、日誌、監控、持續集成、持續部署、維運自動化是基本的，其它包括 Service Orchestration、Distributed Tracking ... 等等

2. 要逐步演進和迭代，不要過於激進，更不要拆分過細，拆分的粒度，要與團隊的架構相互匹配(這裡可參考**康威定律**)

3. Microservice 與 DB 方面，是個很大的困難點，可以深入瞭解下領域驅動設計，做好領域建模，特別是 DB 要隨著服務一起拆分

### 3

- 做服務化改造首先問以下問題：
  - 業務真的需要 Microservice 來解決嗎？
  - 真的所有 module 的問題都要 Microservice 來解決嗎？
  - 技術人員的配置和水平達到要求了嗎？

- 個人比較認同康威定律，Microservice 的拆分粒度一定要和組織結構匹配起來，組織結構和開發管理模式是 Microservice 粒度的最重要參考指標之一

- 採用 Microservice 架構確實解決了 monolithic 應用的一些問題，如資源問題(但消耗資源是比以前多的)、耦合問題、快速迭代問題，服務與服務的職責更加清晰，不同的服務粒度設計的系統是天差地別的

- istio 就是想隱藏 Microservice 的基礎設施，讓業務系統本身不需要感知基礎設施(服務之間的關係、自動化測試部署、服務可視化差錯、容錯、熔斷)



Microservice架構最佳實踐 - 方法篇
===============================

## 服務粒度

- 針對 Microservice 拆分過細導致的問題，建議基於團隊規模進行拆分(例如：**2 pizza team** or **三個火槍手**)

- 從系統規模來講，3 個人負責開發一個系統，系統的複雜度剛好達到每個人都能全面理解整個系統，又能夠進行分工的粒度

- 從團隊管理來說，3 個人可以形成一個穩定的備份

- 從技術提升的角度來講，3 個人的技術小組既能夠形成有效的討論，又能夠快速達成一致意見

- "三個火槍手"的原則主要應用於 Microservice 設計和開發階段，如果經過一段時間發展後已經比較穩定，處於維護期了，無須太多的開發，那麼平均 1 個人維護 1 個 Microservice 甚至幾個 Microservice 都是可以的


## 拆分方法

以下拆分方式不是多選一，而是可以根據實際情況自由排列組合：

### 基於業務邏輯拆分

- 雖然看起來很直觀，但在實踐過程中最常見的一個問題就是**團隊成員對於"職責範圍"的理解差異很大**，經常會出現爭論，難以達成一致意見

- 要判斷拆分粒度，不能從業務邏輯角度，而要計算一下大概的服務數量範圍，然後再確定合適的"職責範圍"

### 基於可擴展拆分

- 將系統中的業務模組按照穩定性排序，將已經成熟和改動不大的服務拆分為**穩定服務**，將經常變化和迭代的服務拆分為**變動服務**

- 不穩定的服務粒度可以細一些，但也不要太細，始終記住要控制服務的總數量

- 主要是為了提升項目快速迭代的效率，避免在開發的時候，不小心影響了已有的成熟功能導致線上問題

### 基於可靠性拆分

- 將系統中的業務模組按照優先級排序，將可靠性要求高的核心服務和可靠性要求低的非核心服務拆分開來

- 重點**保證核心服務的高可用**

- 優點：
  - 避免非核心服務故障影響核心服務
  - 核心服務高可用方案可以更簡單
  - 能夠降低高可用成本：只針對核心服務做高可用方案，機器、頻寬等成本比不拆分要節省較多

### 基於效能拆分

- 將效能要求高或者效能壓力大的模組拆分出來，避免效能壓力大的服務影響其他服務

- 常見的拆分方式和具體的效能瓶頸有關，可以拆分 Web 服務、資料庫、緩存等


## 基礎設施

- 大部分人主要關注的是 Microservice 的 `Small` 和 `Lightweight` 特性，但實際上真正決定 Microservice 成敗的，都是那個被大部分人都忽略的 `Automated`

- Microservice 並沒有減少複雜度，而只是將複雜度從 ESB 轉移到了基礎設施

- 建議按照下面優先級來搭建基礎設施：
  1. Service Discovery、Service Routing、Service Failt Tolerance：這是最基本的 Microservice 基礎設施。
  2. Interface Framework、API Gateway：主要是為了提升開發效率，Interface Framework 是提升內部服務的開發效率，API Gateway 是為了提升與外部服務對接的效率。
  3. 自動化部署、自動化測試、Configuration Center：主要是為了提升測試和維運效率
  4. Service Monitoring/Tracking/Security：主要是為了進一步提昇維運效率


## 討論整理精華

- 現在很火的 Service Mesh 本質上就是借鑑了通信領域的控制面資料面分離思想做出的去中心化的 ESB



Microservice架構最佳實踐 - 基礎設施篇
===================================

為了讓 Microservice 可以順利的運行，完善的 infra 是不能少的，而為了有效管理數量龐大的 Microservice，會需要有以下幾項基礎服務：

## 自動化測試

- Microservice 提倡快速交付，版本週期短，版本更新頻繁，必須通過自動化測試系統來完成絕大部分測試回歸的工作

- 自動化測試涵蓋的範圍包括程式級別的 unit test、單個系統級別的 integration test、系統間的 interface test(優先)，理想情況是每類測試都自動化


## 自動化部署

- Microservice 部署的次數是大一統系統部署次數的幾十倍，採用人工手工處理，需要投入大量的人力，且容易出錯

- 自動化部署系統包括 version control、資源管理（例如，機器管理、虛擬機管理）、部署操作、回退操作等功能

- Configuration Center 包括配置 version control、增刪改查配置、節點管理、配置同步、配置推送等功能


## Configuration Center 

- Microservice 的節點數量非常多，通過人工登錄每台機器手工修改，效率低，容易出錯

- 有的運行期配置需要動態修改並且所有節點即時生效，人工操作是無法做到的

- Configuration Center 包括 configuration version control、增刪改查配置、節點管理、配置同步、配置推送等功能


## Interface Framework

- Microservice 提倡輕量級的通信方式，一般採用 HTTP/REST or RPC 方式統一 interface 協議

- 還需要統一 interface 傳遞的資料格式

- 可以由不同的程式語言實現(但基本上還是建議使用統一的 technique stack)


## API Gateway

- 在外部系統看來，它不需要也沒辦法理解這麼多 Microservice 的職責分工和邊界，它只會關注它需要的能力，而不會關注這個能力應該由哪個Microservice 提供

- 如果外部系統直接訪問某個 Microservice，則意味著每個 Microservice 都要自己實現安全和權限的功能，這樣做不但工作量大，而且都是重複工作

- 需要一個統一的 API Gateway，**負責外部系統的訪問操作**

- API Gateway 是外部系統訪問的 interface ，所有的外部系統接⼊系統都需要通過 API Gateway，主要包括接入認證(是否允許接入)、權限控制(可以訪問哪些功能)、傳輸加密、請求路由、流量控制 ... 等功能

- API Gateway 不要有業務邏輯


## Service Discovery

- Microservice 節點經常變化

- 希望節點的變化能夠及時同步到所有其他依賴的 Microservice；如果採用手工配置，是不可能做到即時更改生效的

- 實現方式：
  - 自理式：每個 Microservice 自己完成 Service Discovery
  - 代理式：Microservice 之間有一個負載均衡系統(圖中的 Load Balancer 節點)，由負載均衡系統來完成 Microservice 之間的 Service Discovery

- 代理式的方式看起來更加清晰，Microservice 本身的實現也簡單了很多

- 使用代理式的缺點：
  - 可用性風險：Load Balancwe 系統故障，就會影響所有 Microservice 之間的調用
  - 效能風險：所有的 Microservice 之間的調用流量都要經過 Load Balancer 系統，效能壓力會隨著 Microservice 數量和流量增加而不斷增加，最後成為效能瓶頸

- 自理式示意圖：
![Service Discovery - self](/blog/images/architecture-design/service-discovery-self.png)

- 代理式示意圖：
![Service Discovery - proxy](/blog/images/architecture-design/service-discovery-proxy.png)


## Service Routing

- Service Routing 核心的功能就是`路由算法`

- 常見的路由算法有：隨機路由、輪詢路由、最小壓力路由、最小連接數路由等


## Service Failt Tolerance

- 系統拆分為 Microservice 後，單個 Microservice 故障的機率變小，故障影響範圍也減少，但是 Microservice 的節點數量大大增加

- 從整體上來看，系統中某個 Microservice 出故障的機率會大大增加

- 需要 Microservice 能夠自動應對這種出錯場景，及時進行處理

- 常見的 Service Failt Tolerance 包括請求重試、流控和服務隔離


## Service Monitoring

- 系統拆分為 Microservice 後，節點數量大大增加，導致需要監控的機器、網路、進程、interface 調用數等監控對象的數量大大增加

- 一旦發生故障，我們需要快速根據各類信息來定位故障

- 主要用途：
  - 實時蒐集信息並進行分析，避免故障後再來分析，減少了處理時間
  - Service Monitoring 可以在實時分析的基礎上進行預警，在問題萌芽的階段發覺並預警，降低了問題影響的範圍和時間

- 需要蒐集並分析大量的資料，因此建議做成獨立的系統


## Service Tracking

- 如果需要跟蹤某一個請求在 Microservice 中的完整路徑，Service Monitoring 是難以實現的

- Service Monitoring 會記錄請求次數、響應時間平均值、響應時間最高值、錯誤碼分佈這些信息

- Service Tracking 會記錄其中某次請求的發起時間、響應時間、響應錯誤碼、請求參數、返回的 JSON 對象等信息


## Service Security

- 從業務的角度來說，部分敏感資料或者操作，只能部分 Microservice 可以訪問，而不是所有的 Microservice 都可以訪問

- 需要設計Service Security機制來保證業務和資料的安全性

- Service Security 主要分為三部分：接入安全、資料安全、傳輸安全

- Service Security 可以集成到 Configuration Center 系統中進行實現：
  - Configuration Center 配置 Microservice 的接入安全策略和資料安全策略
  - Microservice 節點從 Configuration Center 獲取這些配置信息
  - 在處理具體的 Microservice 調用請求時根據安全策略進行處理



References
===========

- [從0開始學架構_架構基礎_架構入門-極客時間](https://time.geekbang.org/column/intro/81)

- [康威定律 - 維基百科，自由的百科全書](https://zh.wikipedia.org/wiki/%E5%BA%B7%E5%A8%81%E5%AE%9A%E5%BE%8B)

- [Microservice架構的理論基礎-康威定律 - 每日頭條](https://kknews.cc/zh-tw/news/6q2l2rm.html)