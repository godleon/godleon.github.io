---
layout: post
title:  "Istio 簡介"
description: "本篇文章是學習 Udemy 課程 'Istio Hands-On for Kubernetes' 過程中所留下的學習筆記"
date: 2021-06-26 15:45:00
published: true
comments: true
categories:
  - Istio
tags:
  - Istio
  - ServiceMesh
---

Architecture
============

- Istio Gateway 可以有多個，而接收的流量是來自於 Istio ingressgateway service

- Istio ingressgateway service 是由 Istio profile 所產生，且同時會產生對應的 AWS classic ELB

- 所有的 k8s service 都會註冊到 Istio Service Registry

- 要調整 traffic routing 的行為，可以在 virtual service 上面指定，將特定的流量導向特定的 service 

- virtuial service 正是 Istio 發揮 routing 能力的地方，以下舉幾個例子：
  - canary deployment
  - routing rules based on http header attribute
  - 30% request 回應 HTTP 400 (fault injection)
  - 10% request 以 timeout 方式處理
> 以上是單純 ingress 做不到的，但透過 Istio Virtual Service 就可以

- service entries 是 egress 用的，主要用來定義在 istio mesh 外部的服務，而非 istio mesh 的服務若是希望在 k8s 內部被存取到，就需要被增加到 service entry 中

- TLS 屬於 L4，無法看到 HTTP header host

- SNI 是用來告訴 server 要回應哪個 certificate (Istio Gateway 才知道要回應哪個 certificate)


## Traffic Splitting

1. 需要為 pod 加上 `version` label，並非在 service level

2. 也需要為 pod 加上 app label

3. 建立 Istio Destination Rules (`kind: DestinationRule`)，根據不同的 version label 不同的值，設定不同的 subset (可以把 version 與 subnet 視為相同的概念)

4. 在 virtual service 中加上 subset & weight 的資訊 (但在 `gateway` 的設定中，必須同時包含 gateway 名稱 & `mesh` 關鍵字)
> 關鍵字 `mesh` 是用來讓 virtual service 中的規則都被設定在每一個 sidecar envoy proxy 中，而不會只有在 mesg edge(就是 gateway) 出現


## Identity based routing

- 可以在 client 端設定特定的 HTTP header，並在 virtual service 中根據 HTTP header 的內容進行 routing

- 應用範例：在 header 中放入 user_id，只有特定的 user_id 會 routing 到新版的的服務中

- 規則是

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - reviews
  gateways: # names of gateways and sidecars that should apply these routes
  - bookinfo-gateway # Don't ONLY USE this gateway as "reviews" k8s service is used internally by productpage service, so this VS rule should be applied to Envoy sidecar proxy inside reviews pod, not edge proxy in gateway pod. 
  - mesh # applies to all the sidecars in the mesh. The reserved word mesh is used to imply all the sidecars in the mesh. When gateway field is omitted, the default gateway (mesh) will be used, which would apply the rule to all sidecars in the mesh. If a list of gateway names is provided, the rules will apply only to the gateways. To apply the rules to both gateways and sidecars, specify mesh as one of the gateway names. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - match: # <---- this routing to apply to all requests from the specifieduser
    - headers: # Note: The keys uri, scheme, method, and authority will be ignored.
        end-user: # WARNING: merely passing this key-value pair in curl won't work as productpage service won't propagate custom header attributes to review service! You need to login as a user Ref: https://stackoverflow.com/a/50878208/1528958
          exact: log-in-as-this-user
    route:
    - destination:
        host: reviews
        subset: v1 # then route traffic destined to reviews (defined in hosts above) to backend reviews service of version 1
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 10 # <--- canary release. % of traffic to subset v1
    - destination:
        host: reviews
        subset: v2
      weight: 10
    - destination:
        host: reviews
        subset: v3
      weight: 80
```


## Query String based routing

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: # destinations that these routing rules apply to. VirtualService must be bound to the gateway and must have one or more hosts that match the hosts specified in a server
  - reviews
  gateways: # names of gateways and sidecars that should apply these routes
  - bookinfo-gateway # Don't ONLY USE this gateway as "reviews" k8s service is used internally by productpage service, so this VS rule should be applied to Envoy sidecar proxy inside reviews pod, not edge proxy in gateway pod. 
  - mesh # applies to all the sidecars in the mesh. The reserved word mesh is used to imply all the sidecars in the mesh. When gateway field is omitted, the default gateway (mesh) will be used, which would apply the rule to all sidecars in the mesh. If a list of gateway names is provided, the rules will apply only to the gateways. To apply the rules to both gateways and sidecars, specify mesh as one of the gateway names. Ref: https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService
  http: # L7 load balancing by http path and host, just like K8s ingress resource
  - match: # <---- this routing to apply to all requests from the specifieduser
    - headers: # Note: The keys uri, scheme, method, and authority will be ignored.
        end-user: # WARNING: merely passing this key-value pair in curl won't work as productpage service won't propagate custom header attributes to review service! You need to login as a user Ref: https://stackoverflow.com/a/50878208/1528958
          exact: log-in-as-this-user
    # - uri:
    #     regex: '^/tester' # could be exact|prefix|regex
    #   ignoreUriCase: true
    route:
    - destination:
        host: reviews
        subset: v1 # then route traffic destined to reviews (defined in hosts above) to backend reviews service of version 1
  - match:
    - queryParams:
        test-v2:
          exact: "true" # if "?test-v2=true" in query string. NOTE: prefix not supported for queryString
    route:
    - destination:
        host: reviews
        subset: v2 # then route traffic destined to reviews (defined in hosts above) to backend reviews service of version 1
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 10 # <--- canary release. % of traffic to subset v1
    - destination:
        host: reviews
        subset: v2
      weight: 10
    - destination:
        host: reviews
        subset: v3
      weight: 80
```

這樣透過 `http://example.com/?test-v2=true` 就可以連到 v2



Egress Traffic Security and Traffic Control
===========================================

- istio 預設對 egress 的流量是全部開放的(ALLOW_ANY)

- 但這樣就無法監控 service 對外的流量情況

- 沒有監控到的在 Kiali 中就會以 PassThroughCluster 來呈現，若是有間控到的就會以 service entry 的方式出現



Istio Hands-On for Kubernetes
=============================

## Finding Performance Problems

- 可透過 Kiali & Jeager 來檢視回應狀況 & 時間

- 要避免 cascade failure 的方式 => circuit break

## 為什麼不直接用 Envoy 而用 Istio ?

- Envoy 其實挺複雜的，看一下官網文件就知道了

- Istio 大幅簡化了 Envoy，讓其可以很方便的放入 Kubernetes 中運行，透過原本操作 Kubernetes 的方式就可以操作 Envoy

- 1.5 版之前，Istio control plane 是很複雜的，後來這些功能都收到 istiod 裡面了

## Telemetry

- 沒有辦法直接塞一個 sidecar container proxy 進到已經存在的 pod，只能砍掉那個 pod 讓他自己重新長出來

## distributed tracing

- Istio 不需要額外使用 OpenTracing client library 蒐集資料，但需要 propagate headers

- trace 是整個要追蹤的目標，這會根據目標的不同而不同；而 span 則是 trace 中不同的分段

![Trace & Span](/blog/images/service-mesh/tracing-trace-span.png)

- [requirements - x-request-id](https://istio.io/latest/docs/tasks/observability/distributed-tracing/overview/#trace-context-propagation)

- 若要收集 open tracing 的資料，application 需要做點調整，當收到以下 HTTP header 資訊時，若有進一步的呼叫，要帶上這些 HTTP header 的內容：
  - x-request-id
  - x-b3-traceid
  - x-b3-spanid
  - x-b3-parentspanid
  - x-b3-sampled
  - x-b3-flags
  - x-ot-span-context


Traffic Management
==================

- 當 workload 移到 Istio 後，建議額外加上一個 `app` label，因為在 Kiali 監控的顯示中會拉取 `app` label 的資料作為 App 名稱的顯示，提供一個方便辨識的值對於監控會很有幫助

- 加上 `vresion` label，當同一個 app 有不同的 version 存在時，可以在 Kialia Graph 中的 **Versioned app graph** 中看到不同 version 的分佈

- 對 virtualservice 的調整，會被 istiod 佈署到所有的 envoy proxy 中；並非用來取代 k8s 中的 service，其實兩者沒啥太多關係

- virtualservice 其實只需要一個就好，並不需要每個 micro service 都搭配一個

- 在 virtualservice 的 `spec.hosts` 中的設定，指定的是 service 名稱，用來指定 virtualservice 的 rule 要套用到哪些 service 的 envoy proxy 上；若是要指定其他 namespace 的 service，則需要使用 FQDN

- 若要實作 sticky session，virtual service 就不需要有 weighted 的相關設定； 但若使用 useSourceIp 可能還是會有問題

- 若要透過 HTTP header 做 ConsistentHashing 來達成 sticky session 的效果，那程式中就必須要實作 header propagation
 
- 若沒有設定 weighting，那就跟原生的 k8s 一樣是 round robin；用了 weighting 就可以調整權重；進一步要用 sticky session 就不能用 weighting 設定了

- 適用情境：當每個 user 進來的 request 需要花大量的資源運算處理，而這結果是可以復用的；那就可以透過 ConsistentHashing 的設定讓同一個 user 去使用相同的 pod，而 pod 在處理完後快取在自己的 memory 中，有同樣的 request 就可以直接回應
> 不過這也衍生出另外一個問題 => 要考量 pod lifecycle，不應該期待 pod 一直都會存活著


Gateway
=======

- canary 用在 frontend pod 會失效，因為 frontend pod 是第一個入口，而且 proxy 是運行在 container 本身發送 request 之後，因此分配比例這件事情其實就只是 browser 端就已經決定了，因此 canary deployment 設定的比例就會無效

![Istio without Edge Proxy](/blog/images/service-mesh/istio-without-edge-proxy.png)

- 承上，若要在 frontend service 中要達成 canary 佈署指定的分配，就會需要一個 edge proxy 來解決這個問題(也就是 Istio Gateway)，作為整個 service chain 的第一個 service

![Istio without Edge Proxy](/blog/images/service-mesh/istio-with-edge-proxy.png)
![Istio without Edge Proxy Gateway](/blog/images/service-mesh/istio-with-edge-proxy-gateway.png)

- Gateway 中的 `spec.servers.hosts` 設定可以用來限定特定 domain name 的 traffic


Dark Releases
=============

- 可以直接將新版的程式直接以 canary 的方式佈署，並透過 HTTP header match 的方式來篩選可用新版的使用者

- 透過上面的方式，新版程式可以直接佈署到 production 環境，透過 browser extension 的方式修改 HTTP header 直接在 production 環境真實測試

- 但要完成上面的目的，程式同時需要提供 HTTP header propagation 的相關開發



Fault Injection
===============

- 可以透過 fault injection 來讓特定比例的 request 得到錯誤的回應，或是讓回應速度變慢

- 通常是用來與 chaos engineering 的概念搭配，試圖製造某些混亂，目的在於檢視目前系統是否可以進一步提昇穩定性的評估方法 


Circuit Breaking
================

- 網路不可靠，總是會有問題發生，如何避免 cascading fauilure 是個需要考慮的議題

- cascading failure 隨時都有可能發生，不只可能是網路造成的，沒寫好的程式或是其他因素都有可能

- failure 一定會發生，如何 fail fast 且不會影響服務，就可以搭配 circuit breaking 的機制來做

- Istio 上的 circuit breaking 機制可以細到 pod level，因此同一個服務的多個 pod 其中一個有問題，就只會對其熔斷，不會影響同一個 service 的其他 pod

- 每一段時間，circuit breaker 就會去測試有問題的 pod 是否恢復，恢復的話就會重新把 traffic 送過去

- micro service 的架構上，必須設計可以容忍某個服務臨時有狀況中斷，相關的 error handler 要做好

- circuit breaking 並不是用來緩解有問題的程式的一個方式，通常是用來緊急處理用，有問題的程式終究還是要找到方法解決

- istio 中要開發 circuit breaking 相關的設定，需要透過 Destination Rule (關鍵字：`Outlier Detection`)

- baseEjectionTime 是用來指定將 pod 從 LB 移除的時間單位，隨著每次 pod 被偵測到尚未恢復，pod 從 LB 移除的時間就會呈現倍數加長

- 若是 service 有設定 circuit breaking，在 Kiali 上就會有一個小小的閃電圖示

- 其實 circuit breaking 可能會把系統弄的更複雜，因為目前沒有一個好的監控機制可以即時了解目前 circuit breaking 的運作狀況(但這問題應該之後會被其他軟體解決)


Mutual TLS
==========

- 可以用來避免在 cluster 內的 man-in-the-middle attack；同一個 worker node 之間的 pod 相互傳輸是沒啥問題的，但不同 worker node 之間的資料傳輸安全性可就無法被保證了
> 跨不同 AZ 的 worker node，pod 之間的網路傳輸是會離開資料中心的

- 做法：
  - 撰寫規則，阻擋所有 envoy proxy 之間非 TLS 的流量
  - 升級所有 envoy proxy，讓相互之間的通訊套用 mTLS

- mTLS 其實已經預設開啟，在 Kiali 圖形顯示中，把 Display 中的 `Security` 打勾，在服務之間的連線有出現一個鎖頭的圖示就表示是流量是經過 TLS 加密的

- 如果有個 pod 從 non-istio enabled namespace 送出 request 到 istio enabled namespace 中的 pod，中間的流量就無法自動被 istio mTLS 加密，而為了讓這流量可以正常，可以設定 `PERMISSIVE` mTLS(這是預設值)
![Istio permissive mTLS](/blog/images/service-mesh/istio-permissive-mtls.png)

- 也可以將 mTLS 設定為 restrict，這樣就會拒絕非 HTTPS 的請求，只要套用以下的 YAML 設定：

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "force-mtls"
  namespace: "default"
spec:
  mtls:
    mode: STRICT
```


Customization Istio
===================

- 建立使用 istioctl 來管理 istio 相關的安裝 & 設定

- `default` & `demo` profile 除了安裝的元件不太相同之外，對於元件的 resource request & limitation 設定也不同，而 default 會相對需要多一些的硬體資源

- 建議透過 Prometheus & Grafana 觀察一段時間後，再設定合適的 resource request & limitation

- istio 若要透過 YAML 安裝在 minikube 中，需要把 `values.global.jwtPolicy` 設定為 `first-party-jwt`


Upgrade Istio
=============

- in-place upgrade 很容易失敗，千萬不要在 production 環境上做這件事情

- 要使用 istio canary upgrade 必須先設定 `revision` label 才可以使用

- 帶有 revision label 的情況下，每個 envoy proxy 只會連到對應 revision label 的 istiod

- canary upgrade 依然會將 ingress gateway & egress gateway 進行 in-place upgrade，風險依然很大的

- 最保險的方式還是使用 Live Cluster Switchover 的方式，先建立一座全新的 cluster，安裝好所有新版的相關套件 & workload，測試 ok 後再從 DNS level 將流量切過去 (只是因為修改的是 DNS，這就需要花點時間等待完全生效)