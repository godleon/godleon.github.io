---
layout: post
title:  "Istio 流量管理相關功能介紹"
description: "本篇文章是介紹 Istio 在 Traffic Management 的部份，有哪些特性 & 功能可供使用，藉此可以根據實際需求，在 Istio Mesh 中制定更多種不同的流量管理策略"
date: 2022-08-31 06:45:00
published: true
comments: true
categories:
  - Istio
tags:
  - Istio
  - ServiceMesh
---


Subsets & Destination Rule
==========================

## Subset & Virtual Service 的搭配

在[之前的 Istio 文章](/blog/ServiceMesh/istio-introdution/)中有提過在 `VirtualService` 的部份，可以透過 subset 的方式進行 traffic 分流，但沒提到 `subset` 是怎麼被定義出來的，以下進行說明。

假設目前有個 Kubernetes service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: customers
  namespace: default
spec:
  selector:
    app: customers
  ports:
    - name: http
      protocol: TCP
      port: 80
```

若是 customers 同時有多個版本在 k8s 中，分別帶有 `version: v2` & `version: v3`，若僅使用 k8s service，要進一步拆分的方式就是定義另外另外一個 service 來處理。

但若是透過 [Istio Destination Rule](https://istio.io/latest/docs/reference/config/networking/destination-rule/)，那就可以在同一個 k8s service 的前提下，透過 subset 來細化，例如：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: customers-ab
spec:
  host: customers.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
  subsets:
  # subnet "v1"，指定帶有 "version: v1" label 的 pod
  - name: v1
    labels:
      version: v1
    
  # subnet "v2"，指定帶有 "version: v2" label 的 pod
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

從上面的範例中，有以下兩個重點：

1. 透過 pod 中的 `version` label 來進一步拆分成多個 subset，而這件事情是在 Destination Rule 完成的，而 k8s service 一直都只有一個

2. Destination Rule 中可以定義 Global-Level 的 traffic policy，但每個 subset 還可以設定各自的 traffic policy 設計來 override global 的設定

那搭配 VirtualService 執行分流的範例如下：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customers-route
spec:
  hosts:
  - customers.default.svc.cluster.local
  http:
  - name: customers-v1-routes
    route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v1
      weight: 70
  - name: customers-v2-routes
    route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v2
      weight: 30
```

透過 Destination Rule 定義 subset + Virtual Service 的分流機制，再不改變原有 k8s service 的設定下，就可以輕鬆達到 A/B testing 的目標。

## Destination Rule 其他功能

除了定義 subset 之外，Destination Rule 還可以針對 Traffic Policy 的部份設定以下功能：

1. [Load Balancer Settings](https://istio.io/latest/docs/reference/config/networking/destination-rule/#LocalityLoadBalancerSetting)：可針對 traffic 目的地設定不同的 load balancing policy

2. [Connection Pool Settings](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings)：這是屬於 host-level 的設定，可以套用在 TCP-level or HTTP-level

3. [Outlier Detection](https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection)：這是用來設計熔斷(circuit breaker)機制時所使用

4. [Client TLS settings](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ClientTLSSettings)

5. [Port Traffic policy](https://istio.io/latest/docs/reference/config/networking/destination-rule/#TrafficPolicy-PortTrafficPolicy)：讓 traffic policy 可以套用在 port-level 上

這部份可以設定的細節很多，可以參考上面連結指向的 Istio 官網文件得到最新的資訊



Resiliency
==========

Istio 提供了些功能，可以在服務遇到某些偶發性的故障 or 某些特殊狀況(例如：接近負載上限)而短時間無法正常運行時，可以提供保持可接受服務水準的能力；這些功能不是為了避免服務故障，而是嘗試在沒有停機 or 資料遺失的前提下面對服務故障的狀況，目標是在服務發生故障時可以恢復到正常的狀態。

因此 Istio 中提供了 `timeout` & `retry` 的機制，可以用來將這兩個機制套用在 Virtual Service 上。

## timeout

以下是一個 [timeout](https://istio.io/latest/docs/concepts/traffic-management/#timeouts) 設定的例子，若是 request 超過 10 秒，那 Envoy proxy 會放棄這個 request 並回應 `HTTP 408(Request Timeout)`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    timeout: 10s
```

關於 timeout 還有以下重點需要注意：

1. 在 Istio 中可以為所有服務設定 default timeout，但若是 default timeout 不符合特定服務使用，就可以透過 Virtual Service 為其設定個別的 timeout 設定

2. 當 request 因為 timeout 產生 408 error，connection 還會持續開啟不會關閉，除非 [outlier detection](https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection) 設定被觸發(發生服務熔斷的狀況)


## retry

除了 timeout 之外，搭配 retry 還可以進一步作到更細膩的控制，以下是一個 retry 設定的範例：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
    retries:
      attempts: 10
      perTryTimeout: 2s
      # 可以指定在特定狀況下才進行 retry
      retryOn: connect-failure,reset
```

retry 設定上有以下幾個重點 & 特性需要了解：

1. 在 retry 設定中，可以設定 `attempts`(嘗試次數) & `perTryTimeout`(每次 retry 的 timeout)，還可以透過 `retryOn` 關鍵字，設定在特定的條件下才會進行 retry(例如：HTTP 5xx)

2. 如果同時設定 retry timeout & timeout，那就會以較長的 timeout 設定為主

3. `retryOn` 設定的選項可以參考 [Envoy 文件](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter#x-envoy-retry-on)取得更詳細的說明

4. 假設原本 service 有三個 endpoint，其中一個 endpoint 發生錯誤而產生 retry 行為時，request 就會往另外兩個正常服務的 endpoint 發送，而不會發送到造成錯誤導致需要 retry 的 endpoint



Failure Injection
=================

在 micro service 的架構中，服務之間互相發送請求是頻繁發生的行為，而我們無法保證上游服務 100% 都沒問題，也許有時候會有偶發性的故障，甚至一段時間無法提供服務。

而為了提昇服務應對上游服務(upstream service)故障時的能力，我們可以在上游服務中設定一些 fault injection policy 來模擬對上游服務進行請求時發生故障時的情境；而若是在上游服務故障的情況下，服務都可以正常運行，那表示服務不會因為上游服務的問題導致自身故障，進而提昇服務的穩定性。

在 Istio 中提供了兩種 [fault injection](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/) policy，分別是 `delay` & `abort` 以下分別進行說明。

## delay

delay 可以用來模擬以下兩種情況：

1. 緩慢的網路

2. 上游服務負載過重

以下是 delay 的範例：

```yaml
piVersion: networking.istio.io/v1beta1
kind: VirtualService
...
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 7s
        percentage:
          value: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
    fault:
      delay:
        percentage:
          value: 5
        fixedDelay: 3s
```

上面的設定 dely 設定可以分成兩個部份：

1. 若是在 request 中就有加上 `end-user: jason` header，然後每個 request 都會被延遲 7 秒鐘

2. 若是沒有加上 `end-user: jason` header，其中 5% 的 request 會延遲 3 秒鐘


## abort

abort 很直觀，就是直接模擬上游服務故障的狀況，以下給個範例：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
...
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
    fault:
      abort:
        # 若沒有指定 percentage，那就會 100% 都 abort
        percentage:
          value: 30
        httpStatus: 404
```

上面的 abort 設定，會讓送到 rating 服務的 request，有 30% 的 request 會回應 404。

此外，關於 abort 還有以下重點需要記得：

1. 以上面設定的範例，若是沒有指定 percentage，那 100% 都會 abort

2. 若是 virtual service 中有設定 retry policy，只有真實的故障才會觸發 retry 的行為；fault injection 所造成的故障，不會觸發 retry 的行為



Service Entry & Wordload Entry
===============================

若是服務安裝在 Kubernetes 中，只要加上了 Envoy sidecar proxy，那我們就可以搭配一些 mesh feature(例如：Fault Injection、Destination Rule ... 等等) 使用，但如果服務在外部 or 在其他 VM 上，那肯定就沒有 Envoy sidecar proxy 存在了。

此時，若依然對這類型的服務在 Istio mesh 中設定 traffic routing or fault injection 這樣的功能，就可以搭配 [Service Entry](https://istio.io/latest/docs/reference/config/networking/service-entry/) & [Workload Entry](https://istio.io/latest/docs/reference/config/networking/workload-entry/) 來達成這個目標

## Service Entry

一般來說，透過 Istio 掛上 Envoy sidecar proxy 的服務會被自動發現並註冊到 Istio service registry 來加入 mesh 中，而 `ServiceEntry` 的目的是透過手動指定的方式將其他的服務註冊到 Istio service registry，藉此達到呼叫外部服務時也可以使用 mesh feature 的目的。

以下是 `ServiceEntry` 的定義範例，定義了一個服務，並將存取 Dropbox, Google、Facebook ... 等外部服務 API 的 HTTPS 請求都歸類在這一類

```yaml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc-https
spec:
  hosts:
  - api.dropboxapi.com
  - www.googleapis.com
  - api.facebook.com
  location: MESH_EXTERNAL
  ports:
  - number: 443
    name: https
    protocol: TLS
  resolution: DNS
```

有了這個定義後，就可以透過像是 Fault Injection 的功能來模擬上述外部服務故障時的應對狀況。

此外，Service Entry 定義的方式其實很彈性，除了上面的方式外，也可以透過 IP & port number；也可以搭配 egress gateway 做出更多的變化，相關內容可以參考 [Istio Service Entry 官方文件](https://istio.io/latest/docs/reference/config/networking/service-entry/)。 


## Workload Entry

`WorkloadEntry` 是用來描述非 Kubernetes 的單一 workload(例如：VM or bare metal server)，目的也是同樣要將這些 workload 加入到 Istio mesh 中，但需要搭配 Service Entry 才可以將 Workload Entry 中所定義的內容註冊到 mesh 中。

這裡透過以下範例說明一下如何使用：

```yaml
# 指定 vm1 & IP
---
apiVersion: networking.istio.io/v1beta1
kind: WorkloadEntry
metadata:
  name: customers-vm-1
spec:
  serviceAccount: customers
  address: 1.0.0.0
  labels:
    app: customers
    instance-id: vm1

# 指定 vm2 & IP
---
apiVersion: networking.istio.io/v1beta1
kind: WorkloadEntry
metadata:
  name: customers-vm-2
spec:
  serviceAccount: customers
  address: 2.0.0.0
  labels:
    app: customers
    instance-id: vm2

# 透過 ServiceEntry，指定 vm1 & vm2 共同的 label
# 將其視為同一個 service(customers-svc) 並註冊到 service registry 中
---
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: customers-svc
spec:
  hosts:
  - customers.com
  location: MESH_INTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: STATIC
  workloadSelector:
    labels:
      app: customers
```



Sidecar
=======

預設情況下，Istio sidecar proxy 的行為是：

1. inject 到所有的 workload

2. 處理跟 workload 有關的所有流量，這包含 inbound & outbound 流量

但若是我們希望調整這樣的行為呢? 例如：只要處理特定 port 的 inbound 流量、只要處理對外連到特定 namespace 的流量才需要處理

若是有上述的需求，就需要透過 `Sidecar` CRD 來進行客製化，以下舉個範例：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Sidecar
metadata:
  name: no-ip-tables
  namespace: prod-us1
spec:
  # 選擇帶有指定 label 設定的 workload
  workloadSelector:
    labels:
      app: productpage
  ingress:
  # 將送到 port 9080 的 traffic 轉發到 workload 本身的 8080 port
  - port:
      # binds to proxy_instance_ip:9080 (0.0.0.0:9080, if no unicast IP is available for the instance)
      number: 9080 
      protocol: HTTP
      name: somename
    defaultEndpoint: 127.0.0.1:8080
    # not needed if metadata is set for entire proxy
    captureMode: NONE 
  egress:
  # 將送到 workload 的 3306 port 的 MySQL request，轉發到外部的 mysql.foo.com:3306
  - port:
      number: 3306
      protocol: MYSQL
      name: egressmysql
    # not needed if metadata is set for entire proxy
    captureMode: NONE 
    bind: 127.0.0.1
    hosts:
    - "*/mysql.foo.com"
```

上面的範例中，針對帶有 label `app: productpage` 的 workload，調整了 sidecar proxy 對於 ingress & egress 的運作行為。

另外還有些使用 Sidecar CRD 時的重點：

1. 若是設定沒有指定 `workloadSelector` 的 Sidecar CRD 只能有一個，這樣的定義方式是用來作為調整整個 namespace 中 sidecar proxy 的預設行為

2. 在 ingress 設定中，`defaultEndpoint` 可以是 **IP:Port** 的形式設定(但僅能是 `127.0.0.1:PORT` or `0.0.0.0:PORT` 兩種)，或是透過 `unix:///path/to/socket` 來表示 UNIX domain socket

3. 在 egress 設定中，`hosts` 可以是 `namespace/dnsName` 形式的設定，表示特定 namespace 的服務；也可以指定任何透過 [VirtualService](https://istio.io/latest/docs/reference/config/networking/virtual-service/) & [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/) 註冊在 service registry 的服務

3. `captureMode` 用來定義送到 sidecar proxy listener 的流量如何被捕捉，詳細的說明可以參考 [Istio 官網文件](https://istio.io/latest/docs/reference/config/networking/sidecar/#CaptureMode)



Envoy Filter
============

`EnvoyFilter` 提供了一個機制，讓使用者可以調整 Istio Polit 所產生的 Envoy configuration，達成像是調整特定欄位(例如：request/response header)的值、增加特定的 filter、增加新的 listener(or cluster) ... 等等，**這功能必須小心使用，否則可能會影響整個 Istio mesh 的穩定性**。

Envoy Filter 的使用設定有以下特定：

1. EnvoyFilter 設定是可以疊加的，這表示特定 namespace 的特定 workload(由 `workloadSelector` 設定)可以設定任意數量的 Envoy Filter

2. root namespace(一般是 `istio-system`，但可以從 [MeshConfig](https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig) 中調整) 的 Envoy Filter 會先被套用，再來才是 workload namepsace 中所有匹配的 Envoy Filter

以下是個 Envoy Filter 的簡單範例：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: api-header-filter
  namespace: default
spec:
  workloadSelector:
    labels:
      app: web-frontend
  configPatches:
  # 設定 filter
  # 1. 處理當 traffic 進入 default namespace 中，帶有 label "app: web-frontend" 的服務
  # 2. 進入 port 8080
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        portNumber: 8080
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
            subFilter:
              name: "envoy.filters.http.router"
    patch:
      # 透過 LUA，在 response 中增加一個 "api-version: v1" 的 header
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.lua
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
          inlineCode: |
            function envoy_on_response(response_handle)
              response_handle:headers():add("api-version", "v1")
            end
```

透過上面的 Envoy Filter，可以為特定的 workload，增加一個額外的 response header。

[Istio 官網文件](https://istio.io/latest/docs/reference/config/networking/envoy-filter/)中還有不少範例，可以參考看看，但若是要查詢究竟 Envoy 支援的 HTTP filter 有哪些，每一個支援了什麼功能，那到 [Envoy 官網文件](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/http_filters)中可以看到最詳細的資訊。

> 上述範例中所使用到的 [Lua](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter) 只是 Envoy 支援的眾多 HTTP filter 的其中一種而已




References
==========

- [Tetrate Academy | Learn Istio from experts](https://academy.tetrate.io/courses/istio-fundamentals)

- [Istio / Virtual Service](https://istio.io/latest/docs/reference/config/networking/virtual-service/)

- [Istio / Destination Rule](https://istio.io/latest/docs/reference/config/networking/destination-rule)

- [Istio / Traffic Management - Network resilience and testing](https://istio.io/latest/docs/concepts/traffic-management/#network-resilience-and-testing)

- [Istio / Fault Injection](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)

- [Istio / Service Entry](https://istio.io/latest/docs/reference/config/networking/service-entry/)

- [Istio / Workload Entry](https://istio.io/latest/docs/reference/config/networking/workload-entry/)

- [Istio / Sidecar](https://istio.io/latest/docs/reference/config/networking/sidecar)

- [Work Note-Unix domain socket. socket是常見的一種process之間的溝通方式（IPC），socket通訊… | by Chin-Hung Liu | Medium](https://medium.com/@chinhung_liu/work-note-unix-domain-socket-62b42f25ffc2)

- [Istio / Envoy Filter](https://istio.io/latest/docs/reference/config/networking/envoy-filter/)