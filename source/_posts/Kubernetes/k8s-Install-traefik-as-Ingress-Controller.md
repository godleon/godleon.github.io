---
layout: post
title:  "[Kubernetes] 設定 Ingress Controller (以 Traefik 為例)"
description: "此篇文章介紹 Kubernetes Ingress 的基本概念 & 使用方式 (以 Traefik 為例)"
date: 2018-07-04 17:50:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - CKA
  - CKA Networking
---


前言
===

之前提到的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) & [Service](https://kubernetes.io/docs/concepts/services-networking/service/) 都會各自擁有一個 IP address 供存取，但這些 IP 僅有在 K8s cluster 內部才有辦法存取的到，但若要在 K8s cluster 上提供對外服務呢?

而目前可讓外部存取 K8s 上服務的方式主要有三種：

1. **Service NodePort**: 這方法會在每個 master & worker node 上都開啟指定的 port number (這樣其實造成不少資源浪費)

2. **Service LoadBalancer**: 只有在 GCP or AWS 這類的 public cloud 平台才有用的功能

3. **[Ingress][k8s Ingress] 就是被設計來處理這類的問題**：即為此篇文章要談的主角


首先，在沒有 Ingress 之前，從外面到 K8s cluster 的流量會如下圖：

```
    internet
        |
  ------------
  [ Services ]
```

> 但在原有的模式下，如果是在 GCP or AWS 使用 K8s 的話，還可以搭配 LoadBalancer 的 service type 動態取得 LB 對外提供服務；但如果是自己架設 K8s，那就只能透過 Service NodePort 的方式讓使用者從外部存取運行在 K8s 上的服務。

所以 Ingress 在 K8s v1.1 之後就應運而生了，而 Ingress 其實就是一堆 rule 的集合，讓外面進來的網路流量可以正確的被導到後方的 Service，架構變成如下圖：

```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

在官網文件中提到 Ingress 可以負責以下工作：

- give services externally-reachable urls

- load balance traffic

- SSL Termination

- offer name based virtual hosting

看的出來可以做到的功能實在很多，非常值得好好研究一下。


在使用 Ingress 之前
=================

看起來 Ingress 就是處理 inbound traffic 的銀彈，但直接在 YAML 中宣告 Ingress resource 並建立並不會有任何效果，必須搭配 [Ingress controller][k8s Ingress Controller] 才有辦法讓設定真正的生效。

> 目前 [K8s 官方 GitHub repository](https://git.k8s.io/ingress-nginx/README.md) 中提供的則是以 nginx 來作為 Ingress controller。

但我找到另外一個也開發的相當不錯的 HTTP reverse proxy 作為 Ingress Controller，名稱為 [Træfik][traefik]，它可以與相當多的平台或軟體進行整合，例如：Docker, Docker Swarm, Marathon, Consul, etcd, Rancher, Amazon .... 等等，當然 Kubernetes 也是其中之一。

而為什麼要選擇 [Træfik][traefik] 呢? 來看看[官網][[traefik]]提供的 feature 列表:

- Continuously updates its configuration (No restarts!)

- Supports multiple load balancing algorithms

- Provides HTTPS to your microservices by leveraging Let's Encrypt (wildcard certificates support)

- Circuit breakers, retry

- High Availability with cluster mode (beta)

- See the magic through its clean web UI

- Websocket, HTTP/2, GRPC ready

- Provides metrics (Rest, Prometheus, Datadog, Statsd, InfluxDB)

- Keeps access logs (JSON, CLF)

- Fast

- Exposes a Rest API

- Packaged as a single binary file (made with :heart: with go) and available as a tiny official docker image

[Træfik][traefik] 功能真的是相當強大阿! 當然要給它試試看....

因此接下來將會以 [Træfik][traefik] 作為 Kubernetes Ingress Controller 作示範。




```bash
$ kubectl get nodes
NAME            STATUS    ROLES          AGE       VERSION
kube-ingress0   Ready     ingress,node   6d        v1.10.4
kube-ingress1   Ready     ingress,node   6d        v1.10.4
kube-master0    Ready     master         6d        v1.10.4
kube-master1    Ready     master         6d        v1.10.4
kube-master2    Ready     master         6d        v1.10.4
kube-worker0    Ready     node           6d        v1.10.4
kube-worker1    Ready     node           6d        v1.10.4
kube-worker2    Ready     node           6d        v1.10.4

# 將 Ingress Node 加上 lable
$ kubectl label node kube-ingress0 k8s-app=traefik-ingress-lb
$ kubectl label node kube-ingress1 k8s-app=traefik-ingress-lb
```

若要檢視目前每個 node 所帶的 label 可使用以下指令：

> kubectl get nodes --show-labels


佈署 Træfik Ingress Controller
=============================

## 設定 RBAC

首先要讓 Træfik 有相對應的權限，將會進行以下設定：

1. 為 Træfik 建立一個 cluster role(`traefik-ingress-controller`)，並給定足夠的權限

2. 為 Træfik 建立一個 service account(`traefik-ingress-controller`)，並繫結(透過 cluster role binding `traefik-ingress-controller`)到上述的 cluster role 以取得權限

準備檔案 `traefik-rbac.yaml`：

```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - endpoints
      - secrets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: traefik-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: traefik-ingress-controller
subjects:
- kind: ServiceAccount
  name: traefik-ingress-controller
  namespace: kube-system
```

> kubectl apply -f traefik-rbac.yaml

或是也可以直接套用 traefik 最新的程式碼：

> kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-rbac.yaml


## 設定以 DaemonSet 的方式執行

traefik 可以用 Deployment or DaemonSet 的方式進行佈署，兩者的差異比較可以參考[官網說明](https://docs.traefik.io/user-guide/kubernetes/#deploy-trfik-using-a-deployment-or-daemonset)，以下是使用 DaemonSet 安裝所需要的 YAML 設定(`traefik-ds.yaml`)：

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      containers:
      - image: traefik
        name: traefik-ingress-lb
        ports:
        - name: http
          containerPort: 80
          hostPort: 80
        - name: admin
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --api
        - --kubernetes
        - --logLevel=INFO
---
kind: Service
apiVersion: v1
metadata:
  name: traefik-ingress-service
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
    - protocol: TCP
      port: 80
      name: web
    - protocol: TCP
      port: 8080
      name: admin
```

套用上述設定：

> kubectl apply -f traefik-ds.yaml

或是也可以直接套用 traefik 最新的程式碼：

> kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-ds.yaml

到此階段，traefik ingress controller 已經佈署完成，接著就是定義 ingress 並與其繫結。


## 佈署 Traefik Dashboard

traefik 有個不錯的優點就是提供了一個還蠻不錯看的 Dashboard，當然也是需要透過設定 ingress 才有辦法連到，因此準備以下檔案(`traefik-ui.yaml`)：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - port: 80
    targetPort: 8080

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: traefik-ui.k8s.example.com
    http:
      paths:
      - backend:
          serviceName: traefik-web-ui
          servicePort: 80
```

> kubectl apply -f traefik-ui.yaml

除此之外，還有 DNS 的部份需要設定；在上面的 ingress host 設定是 `traefik-ui.k8s.example.com`，由於這個 domain name 是虛擬的，因此我們必須修改 client 端的 DNS 設定(`/etc/hosts`)，將 worker node 的 IP 指到這個 domain name。

在我的實驗環境中，worker node 一共有三台，分別是 **10.103.10.[61-63]**，因此在 client 的 `/etc/hosts` 中加入以下設定

```txt
10.103.10.61  traefik-ui.k8s.example.com
10.103.10.62  traefik-ui.k8s.example.com
10.103.10.63  traefik-ui.k8s.example.com
```

完成之後就可以透過 [http://traefik-ui.k8s.example.com](http://traefik-ui.k8s.example.com) 連到 Traefik Dashboard 了!



佈署網站 & 設定 Ingress
====================

在這邊會另外建立一個 `ingress-test` 作為測試用的 namespace，並設定為 kubectl 的 default namespace：

```bash
# 建立新的 namespace 並改為預設值
$ kubectl create ns ingress-test
$ kubectl config set-context $(kubectl config current-context) --namespace=ingress-test
```

## 設定 DNS

接下來的範例會需要有相對應的 DNS 設定，首先必須確認兩件事情：

1. traefik DaemonSet 所存在的機器 IP (一般就是 worker node)

2. 可控制的 domain name

在本環境中，worker node 一共有三台，分別是 **10.103.10.[61-63]**，將會用到的 domain name 為：

- stilton.k8s.example.com

- cheddar.k8s.example.com

- wensleydale.k8s.example.com

由於上述的並非真的由我控制的 domain，因此我們必須修改 client 端的 DNS 設定(`/etc/hosts`) 如下：

```txt
10.103.10.61  stilton.k8s.example.com cheddar.k8s.example.com wensleydale.k8s.example.com
10.103.10.62  stilton.k8s.example.com cheddar.k8s.example.com wensleydale.k8s.example.com
10.103.10.63  stilton.k8s.example.com cheddar.k8s.example.com wensleydale.k8s.example.com
```


接著以下會分為 **Name-based Routing** & **Path-bsaed Routing** 進行示範：

## Name-based Routing

### Deployment

準備 Deployment 檔案(`cheese-deployments.yaml`)：

```yaml
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: stilton
  labels:
    app: cheese
    cheese: stilton
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: stilton
  template:
    metadata:
      labels:
        app: cheese
        task: stilton
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:stilton
        ports:
        - containerPort: 80
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: cheddar
  labels:
    app: cheese
    cheese: cheddar
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: cheddar
  template:
    metadata:
      labels:
        app: cheese
        task: cheddar
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:cheddar
        ports:
        - containerPort: 80
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: wensleydale
  labels:
    app: cheese
    cheese: wensleydale
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cheese
      task: wensleydale
  template:
    metadata:
      labels:
        app: cheese
        task: wensleydale
        version: v0.0.1
    spec:
      containers:
      - name: cheese
        image: errm/cheese:wensleydale
        ports:
        - containerPort: 80
```

> kubectl apply -f cheese-deployments.yaml

或是可以直接套用官網提供的 Deployment 範例：

> kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/cheese-deployments.yaml

### Service

接著準備 Service 檔案(`cheese-services.yaml`)：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: stilton
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: stilton
---
apiVersion: v1
kind: Service
metadata:
  name: cheddar
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: cheddar
---
apiVersion: v1
kind: Service
metadata:
  name: wensleydale
  annotations:
    traefik.backend.circuitbreaker: "NetworkErrorRatio() > 0.5"
spec:
  ports:
  - name: http
    targetPort: 80
    port: 80
  selector:
    app: cheese
    task: wensleydale
```

> kubectl apply -f cheese-services.yaml

或是可以直接套用官網提供的 Deployment 範例：

> kubectl apply -f https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/cheese-services.yaml

### Ingress

最後則是 Ingress 的設定(`name-based-routing_cheese-ingress.yaml`)：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cheese
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: stilton.k8s.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: stilton
          servicePort: http
  - host: cheddar.k8s.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: cheddar
          servicePort: http
  - host: wensleydale.k8s.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: wensleydale
          servicePort: http
```

> kubectl apply -f name-based-routing_cheese-ingress.yaml


最後就可以透過以下3個連結連到不同的 Ingress -> Service -> Deployment：

- [http://stilton.k8s.example.com](http://stilton.k8s.example.com)

- [http://cheddar.k8s.example.com](http://cheddar.k8s.example.com)

- [http://wensleydale.k8s.example.com](http://wensleydale.k8s.example.com)


## Path-bsaed Routing

這個在 DNS 設定部份就簡單多了，在以下範例中，只要設定一個 DNS record(`cheese.k8s.example.com`) 即可，因此 DNS 設定(`/etc/hosts`)必須加入以下資訊：

```txt
10.103.10.61  cheese.k8s.example.com
10.103.10.62  cheese.k8s.example.com
10.103.10.63  cheese.k8s.example.com
```

而在 **Deployment** & **Service** 的部份跟 Name-based Routing 是相同的，而這裡的 ingress 設定(`path-based-routing_cheese-ingress.yaml`)則是跟上面不同：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cheeses
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: cheeses.k8s.example.com
    http:
      paths:
      - path: /stilton
        backend:
          serviceName: stilton
          servicePort: http
      - path: /cheddar
        backend:
          serviceName: cheddar
          servicePort: http
      - path: /wensleydale
        backend:
          serviceName: wensleydale
          servicePort: http
```

> kubectl apply -f path-based-routing_cheese-ingress.yaml

最後就可以透過以下3個連結連到不同的 Ingress -> Service -> Deployment：

- [http://cheeses.k8s.example.com/stilton](http://cheeses.k8s.example.com/stilton)

- [http://cheeses.k8s.example.com/cheddar](http://cheeses.k8s.example.com/cheddar)

- [http://cheeses.k8s.example.com/wensleydale](http://cheeses.k8s.example.com/wensleydale)



References
==========

- [Ingress - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/ingress)

- [Træfik](https://docs.traefik.io/)

- [[Day 19] 在 Kubernetes 中實現負載平衡 - Ingress Controller - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](https://ithelp.ithome.com.tw/articles/10196261)




[k8s Ingress]: https://kubernetes.io/docs/concepts/services-networking/ingress/ "Kubernetes Ingress"

[k8s Ingress Controller]: https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers "Kubernetes Ingress Controller"

[traefik]: https://docs.traefik.io/ "Træfik"