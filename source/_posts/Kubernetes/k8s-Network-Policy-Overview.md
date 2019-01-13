---
layout: post
title:  "[Kubernetes] Network Policy Overview"
description: "本篇文章的主題在介紹如何利用 Kubernetes 中的 Network Policy 機制，來管理 pod 之間的流量，藉此提高安全性"
date: 2018-12-27 05:15:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Networking
  - Kubernetes Security
  - CKA
  - CKA Networking
  - CKA Security
---


什麼是 Network Policy?
=====================

network policy 是 k8s 在 `v1.7` 版後正式支援的功能，在 k8s 中是一組用來定義 pod 之間(或是與 endpoint) 之間是否能夠互相通訊的規範，並以 `NetworkPolicy` resource object 的型式存在，同樣是透過 Label 的方式來選擇 pod & 定義規則。

但由於 network policy 並非 k8s 核心功能，而是由 network plugin 所實現，因此若要使用 network policy 就必須要使用支援此功能的 CNI plugin 才行。(例如：[Calico][calico])

> 若是使用不支援 network policy 的 CNI plugin 卻設定 network policy，也不是不行，只是不會有任何效果而已

另外，在一般的情況下，k8s 中所有的 pod 都是可以互相溝通的，跟帶有什麼 label or 位於那個 namespace 並沒有什麼關係，即使加上了 network policy，增加了透過 Label Selector 來篩選流量的效果，但只要不是設定被阻擋的來源，即使是不同 namespace 的 pod 也還是可以來存取。



如何設定 Network Policy?
======================

以下是一個標準的設定範例：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  # 指定套用 network policy 的 pod
  # 若沒指定 podSelector，就表示將 network policy 套用到 namespace 中的所有 pod
  podSelector:
    matchLabels:
      role: db
  # 設定 network policy 包含的 policy type 有那些
  policyTypes:
  - Ingress
  - Egress
  # ingress 用來設定從外面進來的流量的白名單
  ingress:
  - from:
    # 指定 IP range
    - ipBlock:
        cidr: 172.17.0.0/16
        # 例外設定
        except:
        - 172.17.1.0/24
    # 帶有特定 label 的 namespace 中所有的 pod
    - namespaceSelector:
        matchLabels:
          project: myproject
    # 同一個 namespace 中帶有特定 label 的 pod
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  # egress 用來設定 pod 對外連線的白名單
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

透過上面的設定，會達成以下的效果：

- 在 namespace `default` 中帶有 `role=db` label 的 pod 的進(**ingress**)出(**egress**)流量會被管制

- 允許下列的白名單存取帶有 `role=db` label 的 pod 的 `TCP:6379`

  - 帶有 `role=frontend` label 的 pod (無論任何 namespace 皆可)

  - namespace `default` 中帶有 `project=myproject` label 的 pod (跟著 network policy 的 namespace)

  - IP 落在 172.17.0.0/16(但不包含 172.17.1.0/24) 的來源端

- 允許帶有 `role=db` label 的 pod 對外連線到 IP 範圍為 `10.0.0.0/24` 的 `TCP:5978`

> 比較需要注意的是，如果沒有設定 `namespaceSelector`，那 network policy 的效果就只會對所在 namespace 中的 pod 有效



細部解析 from(ingress) & to(exgress) 的行為
========================================

從上面的範例可以看出，network policy 進出流量的規則可以分成兩類：

- **ingress** 搭配 `from`，負責管理從外部進來的流量

- **exgress** 搭配 `to`，負責管理從內部出去的流量

而上述進出兩類的規則中，共有 3 種方式可以來定義所要進行管理的目標，分別是 `podSelector`, `namespaceSelector`, & `ipBlock`

## podSelector

這是用來管理與 network policy 同一個 namespace 中的 pod 時會用到的設定方式，以上述的例子為例，該 NetworkPolicy 是位於 `default` namespace 中，因此單用 podSelector 的話就只會將規則套用在位於 `default` namespace 中的 pod 上。 

## namespaceSelector

顧名思義，這是是透過 Label 來篩選 namespace 的，而 namespace 的確也是可以設定 label，只是很少看到此類範例，以下是個簡單示範：

```bash
# 新增一個 namespace 名稱為 new-ns
$ kubectl create namespace new-ns
namespace/new-ns created

# 檢視目前 namespace 詳細資訊(此時無 label 資訊)
$ kubectl describe ns/new-ns
Name:         new-ns
Labels:       <none>
Annotations:  <none>
Status:       Active
...(略)

# 在新增的 namespace 上增加 label
$ kubectl label ns/new-ns project=myproject
namespace/new-ns labeled

# 重新檢視 namespace 資訊，可以看見 label 資訊已經上去了
$
$ kubectl describe ns/new-ns
Name:         new-ns
Labels:       project=myproject
Annotations:  <none>
Status:       Active
...(略)
```

而設定 `namespaceSelector` 的方式會讓被篩選中的 namespace 中所有的 pod 都套用 network policy。

## podSelector + namespaceSelector

這個就很清楚了，效果就是兩個加起來，以下是個設定範例：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
...
spec:
  ...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
      podSelector:
        matchLabels:
          role: client
  ...
```

首先會找到帶有 `user=alice` label 的 namespace，並把 network policy 套用在該 namespace 中帶有 `role=client` 的 pod 上。

## ipBlock

這就是很直覺的單純以 IP range 來進行流量的管理，但一般這個設定是用來過濾從外面進來的網路流量，而不是內部的，因為畢竟內部的 pod 所取得的 IP 並不是固定的，也無法預測。



設定 Default Policy
===================

有時候，網路規則的套用是針對整個 namespace 時，希望可以設定一個對所有 pod 都有效的 default network policy，此時該怎麼做呢?

而在設定 default policy 之前，只要了解幾項大原則，其實就不難推敲出如何設定：

- 若要選擇所有的 pod，則將 **podSelector** 設定為 `{}`

- Network Policy 的設定，若是設定 `policyType`，就表示要全部拒絕該種連線(ingress/egress)的流量

- 若要打開上面的限制，則要在 `spec.ingress` or `spec.egress` 中設定


有了上面 3 個大原則點的設定說明之後，可以衍生出 4 種預設規則，如下：

- 若要拒絕從外面進來的網路流量，要在 **spec.policyTypes** 中包含 `Ingress`，並且不要設定 `ingress`

- 若要拒絕從內部出去的網路流量，要在 **spec.policyTypes** 中包含 `Egress`，並且不要設定 `egress`

- 若要接受從外面進來的網路流量，要在 **spec.policyTypes** 中包含 `Ingress`，並將 `spec.ingress` 設定為 `{}`

- 若要接受從內部出去的網路流向，要在 **spec.policyTypes** 中包含 `Egress`，並將 `spec.egress` 設定為 `{}`


以下有幾個實際範例可以參考：

## 拒絕所有從外面進來的網路流量

要為 namespace 中的每個 pod 設定預設的 network policy，拒絕所有從外面進來的網路流量，基本上要完成兩件事情：

- 選擇所有的 pod

- 設定拒絕所有從外面進來的網路流量的 network policy

因此按照上面的設定規則，可透過以下設定來完成：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  # 選擇所有的 pod
  podSelector: {}
  policyTypes:
  - Ingress
  # 拒絕所有從外面進來的網路流量(不設定 ingress)
```

## 允許所有從外面進來的流量

其實 k8s 預設的行為就是這樣了! 不過還是說明一下，要達成**允許所有從外面進來的流量**的目標，要完成兩件事情：

1. 選擇所有的 pod

2. 設定允許所有從外面進來的網路流量的 network policy

> 雖然這原本就是 k8s 的預設行為，但一旦 allow-all 的 policy 被設定後，還是會有專屬的 iptable rules 來管理這些 pod 的流量

因此按照上面的設定規則，可透過以下設定來完成：

```yaml
  name: allow-all
spec:
  # 選擇所有的 pod
  podSelector: {}
  policyTypes:
  - Ingress
  # 允許所有從外面進來的網路流量
  ingress:
  - {}
```

## 拒絕所有從內部出去的流量

要為 namespace 中的每個 pod 設定預設的 network policy，拒絕所有從內部出去的網路流量，基本上要完成兩件事情：

- 選擇所有的 pod

- 設定拒絕所有從內部出去的網路流量的 network policy

因此按照上面的設定規則，可透過以下設定來完成：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

## 允許所有從內部出去的流量

這同樣也是 k8s 的預設行為! 要達成**允許所有從內部出去的流量**的目標，要完成兩件事情：

1. 選擇所有的 pod

2. 設定允許所有從內部出去的網路流量的 network policy

> 雖然這原本就是 k8s 的預設行為，但一旦 allow-all 的 policy 被設定後，還是會有專屬的 iptable rules 來管理這些 pod 的流量

因此按照上面的設定規則，可透過以下設定來完成：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
spec:
  podSelector: {}
  egress:
  - {}
  policyTypes:
  - Egress
```



實作測試
=======

首先說明，以下的實驗環境是使用 [Kubespray][https://github.com/kubernetes-sigs/kubespray] 安裝，並使用 [calico][calico] 3.1.3 作為 k8s CNI plugin。

> 透過 kubespray 安裝 k8s + calico，會自動在每一個 node 上安裝 [calicoctl][https://docs.projectcalico.org/v3.3/reference/calicoctl/commands/] CLI tool 來協助管理者可以用來檢查 calici 目前的服務相關資訊

接著，有了以上對 network policy 的了解之後，要來做個實驗來驗證一下 network policy 是否真的可以有效的限制 pod 的進出流量，以下有兩個實驗

## 限制 pod 對外的網路流量

在這個實驗中，準備要限制帶有 `network-policy-test1` label 的 pod 的對外流量，禁止其對外存取。

首先，利用以下語法，先在 namespace `new-ns` 中產生一個新的 pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: deny-egress-test
  labels:
    app: "deny-egress-test"
  namespace: "new-ns"
spec:
  containers:
  - name: nettools
    image: travelping/nettools
    command: ['sh', '-c', 'sleep 3600']
```

在預設情況下，對外是正常的：

```bash
$ kubectl -n new-ns exec deny-egress-test -- ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=57 time=2.29 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=57 time=2.37 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=57 time=2.28 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 2.280/2.318/2.379/0.043 ms
```

接著套用以下的 network policy：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-test
  namespace: "new-ns"
spec:
  podSelector:
    matchLabels:
      app: "deny-egress-test"
  policyTypes:
  - Egress
```

套用完規則後，再度測試對外連線：

```bash
# 此時對外連線已經被擋住了
$ c
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2014ms

command terminated with exit code 1
```

那在 Calico 中的 policy 又是如何呈現的呢? 這個部份必須透過 `calicoctl` 來檢查：

```bash
# 檢查指定的 namespace 中有哪些現存的 policy
$ calicoctl get policy --namespace=new-ns
NAMESPACE   NAME                           
new-ns      knp.default.deny-egress-test

# 檢視 policy 的內容內容
$ calicoctl get policy knp.default.deny-egress-test -o wide --namespace=new-ns
NAMESPACE   NAME                           ORDER   SELECTOR                                                               
new-ns      knp.default.deny-egress-test   1000    projectcalico.org/orchestrator == 'k8s' && app == 'deny-egress-test'

# 檢視更詳細的 policy 內容
$ calicoctl get policy knp.default.deny-egress-test -o yaml --namespace=new-ns
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  creationTimestamp: 2018-12-25T20:57:06Z
  name: knp.default.deny-egress-test
  namespace: new-ns
  resourceVersion: "12038597"
  uid: a38b8b9a-0887-11e9-a683-52fddc671d88
spec:
  order: 1000
  selector: projectcalico.org/orchestrator == 'k8s' && app == 'deny-egress-test'
  types:
  - Egress
```

## 限制對於 pod 的存取

在這個實驗中，要完成以下的測試內容：

1. 建立一個 nginx pod，作為被存取的 pod

2. 建立一個 pod(name=`nginx-can-access`, label=`priv=can-access`) 作為存取測試之用

3. 建立另一個 pod(name=`nginx-cannot-access`) 作為存取測試之用

首先使用下面的 YAML 設定建立用來測試的 pod：

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-network-policy-test
  labels:
    app: "nginx-network-policy-test"
  namespace: "new-ns"
spec:
  containers:
  - name: nginx
    image: nginx:1.14-alpine
    ports:
    - containerPort: 80

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-can-access
  labels:
    priv: "can-access"
  namespace: "new-ns"
spec:
  containers:
  - name: nettools
    image: travelping/nettools
    command: ['sh', '-c', 'sleep 3600']

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-cannot-access
  namespace: "new-ns"
spec:
  containers:
  - name: nettools
    image: travelping/nettools
    command: ['sh', '-c', 'sleep 3600']
```

完成 pod 的設定後，首先確認一下 nginx pod 是否可以正常被存取：

```bash
# 取得 nginx pod IP 為 10.233.103.218
$ kubectl -n new-ns get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE    IP               NODE              NOMINATED NODE
deny-egress-test            1/1     Running   24         24h    10.233.76.30     leon-k8s-node05   <none>
nginx-can-access            1/1     Running   0          96s    10.233.103.219   leon-k8s-node04   <none>
nginx-cannot-access         1/1     Running   0          96s    10.233.76.31     leon-k8s-node05   <none>
nginx-network-policy-test   1/1     Running   0          6m7s   10.233.103.218   leon-k8s-node04   <none>

# 尚未套用 network policy 前，可以正常存取
$ kubectl -n new-ns exec nginx-can-access -- /bin/sh -c "ping -c 2 10.233.103.218"
PING 10.233.103.218 (10.233.103.218) 56(84) bytes of data.
64 bytes from 10.233.103.218: icmp_seq=1 ttl=63 time=0.133 ms
64 bytes from 10.233.103.218: icmp_seq=2 ttl=63 time=0.035 ms

--- 10.233.103.218 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.035/0.084/0.133/0.049 ms

# 尚未套用 network policy 前，可以正常存取
root@leon-k8s-node00:/srv/k8s/network_policy# kubectl -n new-ns exec nginx-cannot-access -- /bin/sh -c "ping -c 2 10.233.103.218"
PING 10.233.103.218 (10.233.103.218) 56(84) bytes of data.
64 bytes from 10.233.103.218: icmp_seq=1 ttl=62 time=0.645 ms
64 bytes from 10.233.103.218: icmp_seq=2 ttl=62 time=0.294 ms

--- 10.233.103.218 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.294/0.469/0.645/0.176 ms
```

接著我們要限制 `nginx-cannot-access` 不能存取 nginx pod，所以套用以下設定：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-nginx-access
  namespace: new-ns
spec:
  podSelector:
    matchLabels:
      app: "nginx-network-policy-test"
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          priv: "can-access"
```

套用完上述規則後，再度測試存取 nginx pod：

```bash
# 白名單中的 nginx-can-access 依然可以存取
$ kubectl -n new-ns exec nginx-can-access -- /bin/sh -c "ping -c 2 10.233.103.218"
PING 10.233.103.218 (10.233.103.218) 56(84) bytes of data.
64 bytes from 10.233.103.218: icmp_seq=1 ttl=63 time=0.139 ms
64 bytes from 10.233.103.218: icmp_seq=2 ttl=63 time=0.099 ms

--- 10.233.103.218 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.099/0.119/0.139/0.020 ms

# 但不在白名單中的 nginx-cannot-access 已經無法存取
$ kubectl -n new-ns exec nginx-cannot-access -- /bin/sh -c "ping -c 2 10.233.103.218"
PING 10.233.103.218 (10.233.103.218) 56(84) bytes of data.

--- 10.233.103.218 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms

command terminated with exit code 1
```

最後檢視 calico policy：

```bash
$ calicoctl get policy -o wide --namespace=new-ns
WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.
NAMESPACE   NAME                            ORDER   SELECTOR                                                                        
new-ns      knp.default.deny-egress-test    1000    projectcalico.org/orchestrator == 'k8s' && app == 'deny-egress-test'            
new-ns      knp.default.deny-nginx-access   1000    projectcalico.org/orchestrator == 'k8s' && app == 'nginx-network-policy-test'   


$ calicoctl get policy knp.default.deny-nginx-access -o yaml --namespace=new-ns
WARNING: Your kernel does not support swap limit capabilities or the cgroup is not mounted. Memory limited without swap.
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  creationTimestamp: 2018-12-26T21:11:59Z
  name: knp.default.deny-nginx-access
  namespace: new-ns
  resourceVersion: "12203982"
  uid: e1ffb3df-0952-11e9-a683-52fddc671d88
spec:
  ingress:
  - action: Allow
    destination: {}
    source:
      selector: projectcalico.org/orchestrator == 'k8s' && priv == 'can-access'
  order: 1000
  selector: projectcalico.org/orchestrator == 'k8s' && app == 'nginx-network-policy-test'
  types:
  - Ingress
```

此外若想要了解更多 network policy + Calico 的示範，可以參考 Calico 官網的資訊：

- [Calico Tutorial - Simple Policy Demo](https://docs.projectcalico.org/v3.4/getting-started/kubernetes/tutorials/simple-policy)

- [Calico Tutorial - Stars Policy Demo](https://docs.projectcalico.org/v3.4/getting-started/kubernetes/tutorials/stars-policy/)

- [Calico Tutorial - Controlling ingress and egress traffic with network policy](https://docs.projectcalico.org/v3.4/getting-started/kubernetes/tutorials/advanced-policy)

- [Calico Tutorial - Application layer policy tutorial](https://docs.projectcalico.org/v3.4/getting-started/kubernetes/tutorials/app-layer-policy/)



SCTP 的支援
==========

在 v1.12 開始，network policy 已經以 beta 的型式開始支援 SCTP，因此若希望可以在 network policy 中來管理 SCTP 的流量，只要完成以下的設定即可：

1. 開啟 `SCTPSupport` feature gate

2. 使用支援 SCTP 的 CNI plugin (例如：[Calico][calico])



References
==========

- [Network Policies - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

- [Declare Network Policy - Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)

- [Calico Tutorial - Simple Policy Demo](https://docs.projectcalico.org/v3.4/getting-started/kubernetes/tutorials/simple-policy)

- [Kumu's Blog - Calico 笔记备忘](https://blog.opskumu.com/calico.html)


[calico]: https://www.projectcalico.org/ "Kubernetes CNI plugin - Calico"