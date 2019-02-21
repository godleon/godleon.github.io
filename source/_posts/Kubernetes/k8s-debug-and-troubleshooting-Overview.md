---
layout: post
title:  "[Kubernetes] Debug & Troubleshooting 簡介"
description: "本篇文章介紹在使用 Kubernetes 時遇到的各種錯誤狀況時，可用的一些思考 & 處理方式"
date: 2019-02-22 05:30:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - Kubernetes Troubleshooting
  - CKA
  - CKA Troubleshooting
---





Application Introspection and Debugging
=======================================

`kubectl get event` 是屬於 namespace level，若要檢視其他 namespace 的 event，就要額外加上 `--namespace` 參數



Auditing
========

等到之後了解 Elasticsearch + Fluentd + Kibana 之後再來研究這個部份



Core metrics pipeline
=====================

- k8s 從 v1.8 以後提供了 metrics API，可以提供系統使用資源的資訊，但由於 k8s 本身不儲存這些資料，因此若要查詢一段時間內的資源使用率，就必須使用外部服務來達成

- 要使用 metrics API，就必須先在 k8s cluster 中安裝 [metrics server](https://github.com/kubernetes-incubator/metrics-server) 才行

- 每個 node 上佈署的 kubelet 會提供 summary API，而 metrics server 就是透過 summary API 蒐集相關資訊



Debug Init Containers
=====================

在 Pod 中可以定義 init container，甚至可以定義多個 init container 協助提供正式服務的 container 做一些前置的設定工作，而 init container 工作的方式如下：

- init container 總是會執行到結束為止

- 若定義多個 init container，會依序一個一個執行，且必須等到前一個 init container 執行成功後，才會輪到下一個

- 如果任何一個 init container 工作執行失敗，k8s 就會重新啟動 pod 並再度啟動所有的 init container 繼續執行工作，除非 restartPolicy 設定為 `Never`

若是 pod 啟動時因為 init container 發生錯誤，就會出現類似以下訊息：

```txt
NAME         READY     STATUS     RESTARTS   AGE
<pod-name>   0/1       Init:1/2   0          7s
```

可以透過 `kubectl describe pod <pod-name>` 檢視更細節的 log 資訊：

```txt
Init Containers:
  <init-container-1>:
    Container ID:    ...
    ...
    State:           Terminated
      Reason:        Completed
      Exit Code:     0
      Started:       ...
      Finished:      ...
    Ready:           True
    Restart Count:   0
    ...
  <init-container-2>:
    Container ID:    ...
    ...
    State:           Waiting
      Reason:        CrashLoopBackOff
    Last State:      Terminated
      Reason:        Error
      Exit Code:     1
      Started:       ...
      Finished:      ...
    Ready:           False
    Restart Count:   3
    ...
```

若是 pod status 的開頭為 `Init:`，就表示目前的狀態(or 問題)與 init container 有關，以下是與 init container 的 pod status 範例列表：

- `Init:N/M`：共有 M 的 init container，目前已經完成 N 個

- `Init:Error`：有一個 init container 無法正確完成指定工作

- `Init:CrashLoopBackOff`：有一個 init container 一直無法正確完成指定工作

- `Pending`：尚未啟動任何 init container

- `PodInitializing` or `Running`：init container 相關工作都已經正確執行完璧



Debug Pods and ReplicationControllers
=====================================

## Pod 無法正常啟動

pod 無法正常啟動，原因大概可以歸為幾類，若是持續處於 `pending` 狀態，可能的原因是：

- 資源不足 (可增加更多 node or 移除不必要的 pod)

- 使用了 `hostPort`，但沒注意到 hostPort 的使用數量是有限的，儘量改用 `service` 提供外部存取

若是 pod 持續處於 `waiting` 狀態，可能的原因是：

- 無法正確的 pull container image

若是 pod 會 crash or 一直不是處於很健康的狀態，就可以透過 `kubectl logs` 來檢視 log 資訊，或是使用 `kubectl exec` 在 container 中執行特定指令來進行測試。

因此這個部份會用到的指令大概如下：

```bash
# 檢視 pod 相關資訊
$ kubectl describe pods ${POD_NAME}

# 檢視 node 的系統資源是否足夠
$ kubectl get nodes -o yaml | grep '\sname\|cpu\|memory'
$ kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, cap: .status.capacity}'

# 檢視 pod 中的特定 container log
$ kubectl logs ${POD_NAME} ${CONTAINER_NAME}

# 檢視上一次的 failure log
$ kubectl logs --previous ${POD_NAME} ${CONTAINER_NAME}

# 在 pod 中的特定 container 中執行命令
# 若 pod 中只有一個 container，則 "-c ${CONTAINER_NAME}" 可以省略
# 但 container 必須已經啟動才可以
$ kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ${CMD} ${ARG1} ${ARG2} ... ${ARGN}
```

## Pod 啟動了，但行為不如預期

在 k8s 中建立 application 時，有時候會忽略使用者的輸入錯誤，例如將 `command` 打成 `commnd`，這樣會造成 pod 被建立了，但指定的 command 並沒有正確的被執行，導致於 pod 行為不如預期，為了避免這樣的情況，在建立 pod 時可加上 `--validate` 參數，確保 YAML 語法的 100% 正確性，以下是個簡單範例：

> kubectl create --validate -f mypod.yaml

再來可以確認在 API server 中儲存的 pod definition 是否與當初建立的時候一致，可以透過類似 `kubectl get pods/mypod -o yaml > mypod-on-apiserver.yaml` 的指令輸出 API server 的 pod definition，然後仔細比對原始的 pod definition，有時候兩者的些許不同就是造成 pod 行為不如預期的主要原因。



Debug Services
==============

由於從 k8s 外面並無法直接存取 pod，因此當 service 出問題時，要利用一些特別的手段來檢查，以下會說明一些常見的方式。

## 確認 service 存在

首先最基本的，就是透過 `kubectl get svc svc_name` 來判斷 service 是否存在，是否有對應到 pod 對外開放的 port。

## 在 pod 中執行命令

接著要進入到 k8s 內部進行檢查，但首先必須要有一個已經成功啟動的 pod，才可以透過這個 pod 進入到 k8s 內部進行網路相關的檢查，透過以下指令可以在 pod 中執行命令：

> kubectl exec <POD-NAME> -c <CONTAINER-NAME> -- <COMMAND>

但若是沒有現成的 pod 可用怎辦? 那可以自行啟動一個 nettools pod 來使用，裏面已經帶有一些常用網路工具在內：

> kubectl run -it --rm --restart=Never nettools --image=godleon/nettools sh

## 檢測 service 是否正常運作

再來是檢測從 k8s 內部可否存取到 service & DNS 是否運作正常：

```bash
# 從 nettools pod 中檢測是否可以存取 service DNS
u@nettools_pod$ wget -O- hostnames
wget: bad address 'hostnames'

# 從下面的輸出可以看出 service DNS 目前無法正確解析
u@nettools_pod$ nslookup hostnames
Server:		10.233.0.3
Address:	10.233.0.3#53

** server can't find hostnames: NXDOMAIN

# 若是有對應的 service 就會出現以下訊息
# 然後一般也都會可以 ping 的通
u@nettools_pod$ nslookup kubernetes
Server:		10.233.0.3
Address:	10.233.0.3#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.233.0.1

u@nettools_pod$ nslookup kubernetes.default
Server:		10.233.0.3
Address:	10.233.0.3#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.233.0.1

u@nettools_pod$ nslookup coredns
Server:		10.233.0.3
Address:	10.233.0.3#53

** server can't find coredns: NXDOMAIN

u@nettools_pod$ nslookup coredns.kube-system
Server:		10.233.0.3
Address:	10.233.0.3#53

Name:	coredns.kube-system.svc.cluster.local
Address: 10.233.0.3

u@nettools_pod$ ping coredns.kube-system
PING coredns.kube-system.svc.cluster.local (10.233.0.3) 56(84) bytes of data.
64 bytes from coredns.kube-system.svc.cluster.local (10.233.0.3): icmp_seq=1 ttl=64 time=0.041 ms
... (略)
```

比較需要注意的是 service name 的部份需要額外加上 namespace name 才可以，例如上方的範例 `coredns`，屬於 `kube-system` namespace，要查找的時候就要用 `coredns.kube-system` 來查找；但若是上面的 busybox pod 是位於 kube-system namespace 中，就可以不需要加上 namespace name，但一切還是以 `/etc/resolv.conf` 的設定為準。

## 以 IP-based 的方式檢測 service

假設 DNS 解析並非問題所在，可以考慮往 IP based 的方式檢測 service 是否正常，例如要測試 HTTP 服務，可以透過 `curl SERVICE_IP:port` 的方式測試是否有回應。

## Service 背後有 Endpoints 嗎?

service 要可以正常運作，必須仰賴後方的 endpoint，可透過 `kubectl -n NAMESPACE_NAME get endpoints SERVICE_NAME` 來找到 service endpoint，再以上一個步驟 IP-based 的方式檢測 endpoint 服務是否正常。


## kube-proxy 有正常運作嗎?

當 service 的部份檢查都沒有問題了，從 cluster 內部測試都是正常，但從外面連進來就是不正常，此時的問題可能就是發生在 kube-proxy 上。

由於每個 node 都會有安裝 `kube-proxy` 的服務，因此可以透過 `ps aux | grep 'kube-proxy'` 指令檢查 kube-proxy 服務是否有存在。

當 kube-proxy 服務有存在，可以進一步透過 [journalctl](https://www.itread01.com/content/1519304536.html) 來查詢是否有 kube-proxy 的相關錯誤，若是有出現類似無法連結到 master 的錯誤，那就可能需要再檢查一下 node configuration。

### iptables 

> 以下說明是當 kube-proxy 以 iptables 模式安裝時

如何確認 kube-proxy 有正確寫入 iptables rules?

由於每個 namespace 都會帶有一個名稱為 `kubernetes` 的 service，我們可以透過以下指令檢查是否有對應的 iptables rules 存在：

```bash
$ iptables-save | grep 'default/kubernetes'
-A KUBE-SERVICES ! -s 10.233.64.0/18 -d 10.233.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.233.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
```

若有出現相關資訊，表示 kube-proxy 有正確寫入 iptables rules。

### IPVS

> 等下回有安裝 IPVS 模式時再來補這一塊!



Troubleshoot Clusters
=====================

這部份要說明的是 cluster level 的 debug，當 k8s cluster 發生問題，運行不如預期或故障時，該怎麼 debug 呢? 以下分成幾個部份來說明。

## 檢視 log

透過 `journalctl` 指令搭配以下關鍵字，就可以查到各個 component 所對應的 log：

- kube-apiserver (@master-node)

- kube-scheduler (@master-node)

- kube-controller (@master-node)

- kube-controller-manager (@master-node)

- kubelet (@worker-node)

- kube-proxy (@worker-node)

## 一般故障狀況

當某些故障狀況發生時，會對 k8s 中的特定功能造成影響，例如：

- API server 關機 or 掛掉時
  - 對 pod, service, replication controller 等等都無法 start, stop & upgrade
  - 已經存在的 pod 會繼續工作，除非它們依賴 k8s API

- 當 API server 的 storage 掛掉時
  - API server 會無法正常啟動
  - 其他 node 上的 kubelet 再也無法與 API server 通訊，但會持續保持現存的 pod 的運作 & service proxy 工作
  - 重新啟動 API server 之前，手動恢復 API server 的狀態是必要的

- 提供 supporting server(node controller, replication controller manager, scheduler, etc) 的 VM 關機 or 掛掉時
  - 暫時無法與 API server 通訊
  - 無所謂，因為 persistent state 不在這些 service 上，只要後續有恢復即可

- worker node 關機
  - 在該 node 上的 pod 會停止運作，一段時間後會被 reschedule 到其他 node 上

- kubelet 軟體錯誤
  - 無法在 node 中啟動新的 pod
  - 可能會誤刪 pod
  - 該 kubelet 所在的 node 會被標記為 unhealthy
  - replication controller 可能會在其他 node 啟動新的 pod

當然我們也可以做一些預防措施，避免上述情況的發生，例如：

- 利用 IaaS 平台上，VM 故障自動重新啟動的功能

- 利用 IaaS 平台上所提供的相對可靠的 storage for API server & etcd service

- 將 k8s cluster 佈署成具有 HA 的特性

- 定期對 API server 的 storage 作 snapshot

- 不要直接使用 pod，改用 replication controller + service

- application 要設計成可以容忍非預期的 restart

- 佈署成多個獨立的 cluster


References
==========

- [Application Introspection and Debugging - Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application-introspection/)

- [Core metrics pipeline - Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/)

- [Debug Init Containers - Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-init-containers/)

- [Debug Pods and ReplicationControllers - Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-pod-replication-controller/)

- [Logging Architecture - Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/logging/)