---
layout: post
title:  "[Kubernetes] DNS for Service & Pod"
description: "本篇文章的主題在介紹在 Kubernetes cluster 內部的 Service & Pod，其 DNS 解析功能是如何運作的 & 相關規則"
date: 2018-11-14 15:05:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Networking
  - CKA
  - CKA Networking
---


前言
===

DNS service 在 k8s cluster 內部負責 service discovery 的重責大任←讓使用者可以跳脫 IP based 的思維，以 domain name 的方式來進行 application 的架構設計。

而 k8s DNS service 其實也是由一個 DNS pod + service 所組成，並且告訴 kubelet 將每個 container 的 DNS resolution 都指向這個 DNS service，藉由此方式，每個 pod 在存取其他 service 時，都會向一開始就安裝好的 DNS pod 進行 domain name 的詢問。



Domain Name 解析規則
===================

要了解 k8s Domain Name 的解析規則，必須清楚知道以下兩件事情：

1. k8s 中有 namespace 的概念，由於不同的 namespace 中可以有同樣名稱的 service or pod，因此 DNS 解析的部份就需要考慮 namespace

2. k8s cluster domain name，若是未設定，預設就會是 `cluster.local`

有了以上兩個概念之後，接著可以繼續往下。

假設目前 k8s 有以下幾個 resource object：

- 兩個 namespace，分別是 `ns1` & `ns2`

- 在 ns1 中，有個 service 名稱為 `svc1`，與其相關連的 pod 為 `pod1`

- 在 ns2 中，有個 service 名稱為 `svc2`，與其相關連的 pod 為 `pod2`

假設目前有個 pod 位於 ns1 中：

- 可透過 domain name `svc1` or `svc1.ns1` or `svc1.ns1.svc.cluster.local` 存取 svc1

- 可透過 domain name `svc2.ns2` or `svc2.ns2.svc.cluster.local` 存取 svc2 (但無法使用 `svc1`，因為在不同的 namespace 中)

反之亦然。


DNS for Service
===============

## 準備環境

為了進行後面關於 DNS 的實驗，我們透過以下的 YAML 建立 service：

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-nginx
spec:
  selector:
    matchLabels:
      run: k8s-nginx
  replicas: 3
  template:
    metadata:
      labels:
        run: k8s-nginx
    spec:
      containers:
      - name: k8s-nginx
        image: nginx
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: svc-cluster
spec:
  selector:
    run: k8s-nginx
  ports:
  - name: http
    port: 80
    protocol: TCP
  
---
kind: Service
apiVersion: v1
metadata:
  name: svc-headless
spec:
  selector:
    run: k8s-nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
  clusterIP: None
```

套用以上設定後，可以看到類似以下資訊：

```bash
# 檢視套用上面設定後的結果
$ kubectl get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/k8s-nginx-6fb585ddf-4kf46   1/1     Running   0          95s
pod/k8s-nginx-6fb585ddf-6fbkq   1/1     Running   0          95s
pod/k8s-nginx-6fb585ddf-lhdzp   1/1     Running   0          95s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes     ClusterIP   10.233.0.1     <none>        443/TCP   32d
service/svc-cluster    ClusterIP   10.233.49.77   <none>        80/TCP    95s
service/svc-headless   ClusterIP   None           <none>        80/TCP    95s

NAME                        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-nginx   3         3         3            3           95s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-nginx-6fb585ddf   3         3         3       95s


# 取得 endpoint 相關資訊
$ kubectl get endpoints
NAME           ENDPOINTS                                               AGE
.... (略)
svc-cluster    10.233.103.201:80,10.233.76.10:80,10.233.76.11:80       108s
svc-headless   10.233.103.201:80,10.233.76.10:80,10.233.76.11:80       108s


# 取得 k8s DNS service IP
$ kubectl -n kube-system get svc
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
coredns                ClusterIP   10.233.0.3     <none>        53/UDP,53/TCP,9153/TCP   32d
```


## A records

在以上的範例中(namespace = `default`)，k8s 會為 service 自動建立 A record 如下：

- svc-cluster.default.svc.cluster.local

> 一般的 type=ClusterIP 的 service，因此會有一個匹配的 cluster IP

- svc-headless.default.svc.cluster.local

> `clusterIP: None`，屬於 headless service，因此不會有 cluster IP，解析出來的結果會是其 endpoints 資訊

```bash
# 從 host level 使用 nslookup
$ nslookup
> server 10.233.0.3   # 切換 DNS server 到 k8s DNS svc
Default server: 10.233.0.3
Address: 10.233.0.3#53

# 查詢 "default" namespace 中的 "svc-cluster" service domain name
# (type = ClusterIP)
Name:	svc-cluster.default.svc.cluster.local
Server:		10.233.0.3
Address:	10.233.0.3#53

Name:	svc-cluster.default.svc.cluster.local
Address: 10.233.49.77

# 查詢 "default" namespace 中的 "svc-headless" service domain name
# (clusterIP: None)
> svc-headless.default.svc.cluster.local
Server:		10.233.0.3
Address:	10.233.0.3#53

Name:	svc-headless.default.svc.cluster.local
Address: 10.233.103.201
Name:	svc-headless.default.svc.cluster.local
Address: 10.233.76.11
Name:	svc-headless.default.svc.cluster.local
Address: 10.233.76.10
```


## SRV records

除了 A record 之外，k8s DNS 還會額外建立相對應的 SRV record，並且用以下的命名規則來產生：

> [_my-port-name].[_my-port-protocol].[svc_name].[namespace_name].svc.cluster.local

以下進行實際的查詢來檢視一下 DNS SRV record 產生的結果：

```bash
#### 繼續上面的 nslookup 測試 ###

# 修改查詢 type 為 SRV
> set type=srv

# 測試 [_my-port-name].[_my-port-protocol].[svc_name].[namespace_name].svc.cluster.local
> _http._tcp.svc-cluster.default.svc.cluster.local
Server:		10.233.0.3
Address:	10.233.0.3#53

_http._tcp.svc-cluster.default.svc.cluster.local	service = 0 100 80 svc-cluster.default.svc.cluster.local.


# 測試 [_my-port-name].[_my-port-protocol].[headless_svc_name].[namespace_name].svc.cluster.local
> _http._tcp.svc-headless.default.svc.cluster.local                                           
Server:		10.233.0.3
Address:	10.233.0.3#53

_http._tcp.svc-headless.default.svc.cluster.local	service = 0 33 80 10-233-103-201.svc-headless.default.svc.cluster.local.
_http._tcp.svc-headless.default.svc.cluster.local	service = 0 33 80 10-233-76-10.svc-headless.default.svc.cluster.local.
_http._tcp.svc-headless.default.svc.cluster.local	service = 0 33 80 10-233-76-11.svc-headless.default.svc.cluster.local.
```



DNS for Pod
===========

## A record

跟 service 相同，每個 pod 在產生的時候也會以下述的格式分配一個 DNS A record 在 k8s DNS service 中：

> [pod-ip-address].[namespace-name].pod.cluster.local

因此假設 pod 的 ip 為 `1.2.3.4`，namespace 為 `default`，且 cluster domain 為 `cluster.local`，那就會有 `1-2-3-4.default.pod.cluster.local` 這筆 A record 產生。

> 以上的部份是[官方文件](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#a-records-1)上提到會有的設定，但實際上實驗結果(k8s version = `1.12.1`)並不是這樣! 並沒有在 k8s DNS service 中看到上述的 A record，因此這個部份還有帶後續釐清

## Pod 的 hostname & subdomain

這裡要分成兩個部份來說：

- `hostname`: 在預設情況下，每個 pod 的 **hostname** 都會使用在 pod 定義中 `metadata.name` 的值

- `subdomain`：預設情況下不會有這個部份的設定出現

但以上兩個其實都是可以在 pod spec 中定義的，分別是 `.spec.hostname` & `.spec.subdomain`。

因此假設有個 pod spec 有以下設定：

- `.spec.hostname`: **foo**

- `.spec.subdomain`: **bar**

並新增到 namespace `default` 中，就會在該 pod 中的 **/etc/hosts** 中多出 `foo.bar.default.svc.cluster.local` & pod IP 的對應記錄，但是並不會出現在 k8s DNS service 中。

> 不會出現在 k8s DNS service 中的部份，跟[官網文件](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-hostname-and-subdomain-fields)說明有出入，同樣也是待後續釐清

## DNS Policy & Config

此外 pod spec 中還可以根據自身的需求，額外設定 DNS policy & config，但在進行這樣的設定之前，建議先把 Linux `/etc/resolv.conf` 中的相關設定搞清楚，會比較容易理解這一段。

關於 Linux `/etc/resolv.conf` 的設定說明，可以參考下列文章：

- [DNS 客戶端設定](http://www.tsnien.idv.tw/Internet_WebBook/chap13/13-8%20DNS%20%E5%AE%A2%E6%88%B6%E7%AB%AF%E8%A8%AD%E5%AE%9A.html)

- [Linux下域名解析的优化 - 简书](https://www.jianshu.com/p/2c1c081cc521)

- [Linux 本地dns配置文件详解 - 好脑袋和烂笔头 - 开源中国](https://my.oschina.net/guol/blog/114297)

- [DNS域名解析时的顺序问题-linux on the way-51CTO博客](http://blog.51cto.com/linuxtro/282776)

有了以上的 DNS 觀念後，進行 pod DNS policy & config 設定時就會清楚每個設定所產生的效果會是如何了。

而這個部份(pod DNS policy & config)已經有網友有相當詳細的說明，可以參考下列文章：

- [[Kubernetes] DNS setting in your Pod | Hwchiu Learning Note](https://www.hwchiu.com/kubernetes-dns.html)



References
==========

- [DNS for Services and Pods - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)

- [DNS (bind) 設定 - SRV 紀錄 - Office 365 - Lync Server | 黃昏的甘蔗 - 點部落](https://dotblogs.com.tw/jerrymow/2014/01/06/139132)

- [[Kubernetes] DNS setting in your Pod | Hwchiu Learning Note](https://www.hwchiu.com/kubernetes-dns.html)

- [DNS 客戶端設定](http://www.tsnien.idv.tw/Internet_WebBook/chap13/13-8%20DNS%20%E5%AE%A2%E6%88%B6%E7%AB%AF%E8%A8%AD%E5%AE%9A.html)

- [Linux下域名解析的优化 - 简书](https://www.jianshu.com/p/2c1c081cc521)

- [Linux 本地dns配置文件详解 - 好脑袋和烂笔头 - 开源中国](https://my.oschina.net/guol/blog/114297)

- [DNS域名解析时的顺序问题-linux on the way-51CTO博客](http://blog.51cto.com/linuxtro/282776)