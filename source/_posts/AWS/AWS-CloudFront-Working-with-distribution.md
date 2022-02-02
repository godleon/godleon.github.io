
Overview
========

每一組 CloudFront 的設定稱為 `distribution`，其中可以讓管理者進行設定的內容包含：

- content origin：這部份可以是 S3 bucket、MediaPackage channel、特定的 HTTP server，可隨意的組合，但最多一個 distribution 可以有 25 個 content origin
- access：是否讓所有 client 都可以存取內容，或是只有特定的 client
- security：是否要求 client 使用 HTTPS 連線
- cache key：使用哪些值作為 cache key
- origin request settings：是否要讓 CloudFront edge location 回源時，連同 HTTP header、cookie、query string .... 等資料一併帶過去
- geo-restriction：可限定不讓特定區域的 client 連線
- access log：是否產生 access log 供後續分析用



Cache Key
=========

要有效率的運用 CloudFront，必須要清楚哪些資料要被 edge server 儲存(快取, cache)起來，當資料被 cache 在 edge server 後，下一個有 cache hit 的使用者就可以從 edge server 直接拉資料回來而不需要再度回源，進而可以大幅提昇使用者體驗，同時也可以降低 origin server 的負擔；而 edge server 如何決定資料要不要 cache 這件事情，除了 `URL` 的值之外，就取決於 `cache key` 的設定。

因此在設定上可以這樣思考：若是決定了某個 path 底下的 URL 都需要快取，但如果又希望可以針對特定的條件來對快取進行額外的拆分，那就加上 cache key 來完成。

cache key 可以把這東西想像成是 Key/Value store 的 key，只是這個 key 可以是多個值的組合，而這些值的來源可以是以下幾種：

- query string

- HTTP header

- cookie

cache key 可以是上述三種資料的組合，管理者可以根據網站需要加速的需求，調整 cache key 的設定，來提高 cache hit rate；而 cache hit rate 越高，自然也代表越多使用者是由 edge server 來提供服務，使用體驗自然也相對較好。

為了拉高 cache hit rate，自然要讓 cache key match 的機率提高，而直覺來想，其實就是盡量把 cache key 設計簡單一點，只設定真正需要的值作為 cache key，讓不同使用者瀏覽網站時，拉高符合 cache key 設定的機率，自然就會提高 cache hit rate。


Cache Hit Ratio
===============

- 目前 CloudFront 支援以下選項作為 cache key：
  - query string
  - cookie
  - HTTP header

- query string 是大小寫不一樣的，因此 `?id=a1`, `?id=A1`, `?Id=a1`, `?ID=a1` ... 等都會算是不同的 query string；所以程式使用 query string 時盡可能使用一致的命名規則才可以提高 cache hit ratio

- 若是回傳的內容不支援壓縮(compression)，那就移除 `Accept-Encoding` HTTP header


建立 CloudFront Distribution
============================

- 管理者必須確認的是，指定的 origin server(不論是 S3 or EC2 or others)，必須要讓 CloudFront edge location 可以正確的回源