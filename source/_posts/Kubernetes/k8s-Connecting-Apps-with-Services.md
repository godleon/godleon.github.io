---
layout: post
title:  "[Kubernetes] Connecting Applications with Services"
description: "本篇文章的主題在介紹在 Kubernetes cluster 中，如何連線到 service，以及如何從頭建立一個帶有 TLS 憑證的 HTTPS 服務"
date: 2018-11-20 04:05:00
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



Kubernetes 如何將 container 相互連接?
==================================

假設在多台機器的環境中，每一台機器都安裝了 Docker，要讓這些起在這些機器上的 container 相互連結，若僅依賴 Docker 本身所提供的網路功能的話，就只能透過 `HOST_IP:PORT` 的方式來提供 container 相互溝通的能力；這樣的方式在同一個 Host 上所使用的 port 是不能重複的，這種方法在大型 or 多人的的環境上是不可行的，要與每個使用者溝通協調可以使用的 port 幾乎是不可能的事情。

因此在 k8s 中提出了 pod 的概念，並且透過額外建立 overlay network 的方式讓每個 pod 擁有自己的 IP(cluster private IP)，且可以相互溝通，當然也沒有 port conflict 的問題。

k8s 透過了 pod & overlay network 解決了 container 在不同 Host 之間的溝通問題，但還有另外一個問題，就是 pod IP 並不會固定，可能因為 container 重啟的關係而導致 IP 變更，此時其他 Application 就有可能無法透過原本的 IP 連線到該 pod，此時怎辦?

在 k8s 就額外了提供 `Service` resource object，透過 `DNS + Pod Load Balancer` 的概念，來解決這個問題。


開放 Pod 在 Cluster 內部存取
=========================

首先透過[以下的 YAML 定義](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/service/networking/run-my-nginx.yaml)建立一個搭配兩個(replica=2) nginx Pod 的 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
```

套用了以上設定後，可以檢視一下目前的 Pod 清單：

```bash
$ kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP               NODE              NOMINATED NODE
my-nginx-756f645cd7-dwzrp   1/1     Running   0          15s   10.233.76.12     leon-k8s-node05   <none>
my-nginx-756f645cd7-zp8gq   1/1     Running   0          15s   10.233.103.203   leon-k8s-node04   <none>
```

可以看到每個 pod 都已經取得 IP，然後在任何一個 node 執行 curl 應該都可以得到類似以下結果：

```bash
$ curl http://10.233.76.12
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
....(略)
</body>
</html>
```


建立 Service
===========

透過上一個部份，有了兩個在 cluster 內部都可以存取到的 pod，正常情況下，透過 pod IP 存取都是沒什麼問題的，但是若是 pod 因為某些原因重啟了(例如：node 掛掉)而造成 IP 變更時，那原本的 IP 就存取不到了，那怎麼辦?

於是 k8s 就提供了 Service 來解決這個問題，透過將 pod IP 這一層抽象化，讓直接對外提供服務的不是 pod 本身而已 service，且搭配內部的 DNS service，讓 cluster 內部的其他 application 可以簡單的透過 domain name 來存取 service，而 service 會將 network traffic 平均的分流到 pod member 中。

此外，service 有些特性也需要注意一下：

- 每個 service 會被分配一個獨一無二的 IP (cluster IP)

- 這個 IP 的生命周期是跟著 service 的

- 只要 service 活著，service IP 就不會變更


那如何建立一個 service 呢? 搭配上面的 pod，可以套用[以下設定](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/service/networking/nginx-svc.yaml)：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
```

也可以直接透過 expose 指令來處理：(expose 指令會協助新增 service)

> kubectl expose deployment/my-nginx

接著來檢視一下建立 service 後的狀況：

```bash
$ kubectl get svc/my-nginx
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-nginx   ClusterIP   10.233.12.73   <none>        80/TCP    19s

$ kubectl get svc/my-nginx
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
my-nginx   ClusterIP   10.233.12.73   <none>        80/TCP    19s
root@leon-k8s-node00:/srv/k8s/connect_app_with_svc# kubectl describe svc/my-nginx
Name:              my-nginx
Namespace:         default
Labels:            run=my-nginx
Annotations:       ... (略)
Selector:          run=my-nginx
Type:              ClusterIP
IP:                10.233.12.73
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.233.103.203:80,10.233.76.12:80
Session Affinity:  None
Events:            <none>
```

此時若是 **curl http://[CLUSTER_IP]:[PORT]** 也會得到跟上一個部份中一樣的結果



存取 Service
===========

在 k8s cluster 中要存取 service 有兩種主要方式，分別是：

- **Environment Variable**

- **DNS**

其中 `Environment Variable` 已經是內建的，而 DNS 的部份則是必須要安裝 addon 才會有，以下來看看兩種不同的方法是如何存取 service。

## Environment Variable(環境變數)

當一個 pod 被建立時，kubelet 就會自動的增加一些與 service 相關的環境變數進到 pod 中，但這樣的作法其實造成了一些問題，為什麼這樣說呢? 以下做個簡單的實驗：

首先檢視一下在目前 pod 中的環境變數資訊：

```bash
$ kubectl exec my-nginx-756f645cd7-dwzrp -- printenv | grep SERVICE
KUBERNETES_SERVICE_HOST=10.233.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
```

大家應該會發現，上面這個 IP(`10.233.0.1`) 並不是屬於這個 pod 的 service 的 IP，但如果真的去 curl 這個 IP:Port 會得到以下結果：

```bash
$ curl https://10.233.0.1
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403
}
```

會造成這樣的原因是因為這些 pod 建立的時間比 service 還要早，因此 kubelet 在填入環境變數的資訊時，並沒有 service 的訊息可以填入，因此才改填入 `--service-cluster-ip-range` 設定中的第一個 IP 來替代。

那要怎麼樣讓環境變數可以變成正確的值呢? 很簡單，只要把 pod 砍了再重新產生即可。

由於這些 pod 是由 Deployment 管理，因此可以透過將 Deployment 中的 replica 設定變為 0，再改回 2，這樣就可以讓 pod 重新產生，取得正確的 service 環境變數設定：

```bash
# 將 replica 改為 0，藉此砍掉所有的 pod
$ kubectl scale deployment/my-nginx --replicas=0
deployment.extensions/my-nginx scaled

# 確認 pod 都已經被移除
$ kubectl get deployment/my-nginx
NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-nginx   0         0         0            0           24h

# 重新將 replica 改為 2
$ kubectl scale deployment/my-nginx --replicas=2
deployment.extensions/my-nginx scaled

# 檢視 pod status & 新生成的 pod name
$ kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP               NODE              NOMINATED NODE
my-nginx-756f645cd7-cpvw6   1/1     Running   0          5m37s   10.233.76.13     leon-k8s-node05   <none>
my-nginx-756f645cd7-vvmll   1/1     Running   0          5m37s   10.233.103.204   leon-k8s-node04   <none>

# 檢視 pod 環境變數
# 此時的 service 相關的環境變數就正確了
$ kubectl exec my-nginx-756f645cd7-cpvw6 -- printenv | grep SERVICE
MY_NGINX_SERVICE_HOST=10.233.12.73
MY_NGINX_SERVICE_PORT=80
KUBERNETES_SERVICE_HOST=10.233.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
```

從上面的實驗可以看出，在 service 建立之後才產生的 pod，就會取得正確的環境變數了。


## DNS

DNS 則是 addon，不會預設安裝進 k8s 中，但目前普遍 k8s 安裝相關的專案都會協助安裝 DNS service(kube-dns or CoreDNS)，我們可以透過以下指令檢視：

```bash
$ kubectl -n kube-system get svc
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
coredns                ClusterIP   10.233.0.3     <none>        53/UDP,53/TCP,9153/TCP   35d
kubernetes-dashboard   ClusterIP   10.233.22.60   <none>        443/TCP                  35d
```

從上面可以看出來，我的 k8s cluster 中的 DNS service 是 CoreDNS，並非 kube-dns。

接著以下要同時檢視之前建立的 my-nginx service 是否可以正確的在 cluster 內部解析：

```bash
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.233.0.1     <none>        443/TCP   35d
my-nginx     ClusterIP   10.233.12.73   <none>        80/TCP    24h

# 新增一個 busybox pod 並直接進入其 terminal
$ kubectl run curl --image=radial/busyboxplus:curl -i --tty

# 檢視 pod DNS server 設定
# 可以看出被加入了 
[ root@curl-5cc7b478b6-29bjq:/ ]$ cat /etc/resolv.conf 
nameserver 10.233.0.3
search default.svc.cluster.local svc.cluster.local cluster.local qct.io
options ndots:5

# 透過 nslookup 查詢之前新增的 my-nginx service
# 解析出來的結果跟上面使用 kubectl 檢視的結果是相同的
[ root@curl-5cc7b478b6-29bjq:/ ]$ nslookup my-nginx
Server:    10.233.0.3
Address 1: 10.233.0.3 coredns.kube-system.svc.cluster.local

Name:      my-nginx
Address 1: 10.233.12.73 my-nginx.default.svc.cluster.local
```


如何更安全的存取 Service
=====================

以下示範如何設定一個名稱為 `my-secure-nginx` 的 HTTPS service。

## 產生 TLS certificate &  建立 Secret

首先產生 TLS certificate，並取得 Base64 編碼結果：(記得 TLS certificate 的 domain 要設定為 `my-secure-nginx`)

```bash
# 產生 TLS certificate
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /tmp/nginx.key -out /tmp/nginx.crt -subj "/CN=my-secure-nginx/O=my-secure-nginx"

# 取得 Base64 編碼結果
$ cat /tmp/nginx.crt | base64
LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS ... (略)
$ cat /tmp/nginx.key | base64 
LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS ... (略)
```

套用以下 YAML 設定，建立一個 Secret resource object：

```yaml
apiVersion: "v1"
kind: "Secret"
metadata:
  name: "my-secure-nginx-secret"
data:
  nginx.crt: "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS ... (略)"
  nginx.key: "LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS ... (略)"
```

```bash
# 確認 secret 是否有正確的被建立
$ kubectl get secret/my-secure-nginx-secret
NAME                     TYPE     DATA   AGE
my-secure-nginx-secret   Opaque   2      30s

# 檢視 secret 詳細內容
$ kubectl get secret/my-secure-nginx-secret -o yaml
apiVersion: v1
data:
  nginx.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS ... (略)
  nginx.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS ... (略)
kind: Secret
metadata:
  ... (略)
  creationTimestamp: 2018-11-18T19:54:17Z
  name: my-secure-nginx-secret
  namespace: default
  resourceVersion: "6164328"
  selfLink: /api/v1/namespaces/default/secrets/my-secure-nginx-secret
  uid: bb87fa8e-eb6b-11e8-890b-66712a0dd587
type: Opaque
```

## 新增 nginx SSL 相關設定

在 k8s 中，一般我們會以 ConfigMap 的方式來作為處理設定檔的方式，因此套用以下的 YAML 設定來新增 nginx 設定檔：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-secure-nginx-config
data:
  default.conf: |
    server {
            listen 80 default_server;
            listen [::]:80 default_server ipv6only=on;

            listen 443 ssl;

            root /usr/share/nginx/html;
            index index.html;

            server_name localhost;
            ssl_certificate /etc/nginx/ssl/nginx.crt;
            ssl_certificate_key /etc/nginx/ssl/nginx.key;

            location / {
                    try_files $uri $uri/ =404;
            }
    }
```

```bash
# 確認 ConfigMap 已經設定完成
kubectl get configmap/my-secure-nginx-config
NAME                     DATA   AGE
my-secure-nginx-config   1      11s
```

## 新增 Deployment & Service

最後要新增 Deployment & Service 之前，要先確認以下幾件事情：

1. secret 已經新增，名稱為 `my-secure-nginx-secret`，裡面的檔案為 `nginx.crt` & `nginx.key`

2. ConfigMap 已經新增，名稱為 `my-secure-nginx-config`，作為 nginx 的設定，檔名為 `default.conf`

3. nginx default 設定檔位置為 `/etc/nginx/conf.d/default.conf` (檔名與 ConfigMap 中的 data 設定相同)

4. nginx 中預設存放 certificate 的路徑為 `/etc/nginx/ssl`，檔名為 `nginx.crt` & `nginx.key` (與 secret 中的 data 設定相同)

確認了以上幾個要項之後，就可以套用以下設定來建立 Deployment & Service：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: my-secure-nginx
  labels:
    run: my-secure-nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    run: my-secure-nginx
    
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-secure-nginx
spec:
  selector:
    matchLabels:
      run: my-secure-nginx
  replicas: 1
  template:
    metadata:
      labels:
        run: my-secure-nginx
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: my-secure-nginx-secret
      - name: configmap-volume
        configMap:
          name: my-secure-nginx-config
      containers:
      - name: nginxhttps
        image: ymqytw/nginxhttps:1.5
        command: ["/home/auto-reload-nginx.sh"]
        ports:
        - containerPort: 443
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /index.html
            port: 80
          initialDelaySeconds: 30
          timeoutSeconds: 1
        volumeMounts:
        - mountPath: /etc/nginx/ssl
          name: secret-volume
        - mountPath: /etc/nginx/conf.d
          name: configmap-volume
```

## 從外部驗證 HTTPS 服務

套用以上設定後，檢視一下目前系統狀態：

```bash
# 僅列出與 my-secure-nginx 相關的物件
$ kubectl get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/my-secure-nginx-66646484d-hwrkr   1/1     Running   0          10s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/my-secure-nginx   NodePort    10.233.43.113   <none>        80:30779/TCP,443:31770/TCP   10s

NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-secure-nginx   1         1         1            1           10s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/my-secure-nginx-66646484d   1         1         1       10s
```

由於 service 設定為 NodePort 的關係，以下測試可以從外面來進行：

```bash
# 測試 HTTP (without TLS certificate)
$ curl http://10.107.13.10:30779
... (略)
<h1>Welcome to nginx!</h1>

# 測試 HTTPS (with TLS certificate)
# 因為這是 self-signed certificate，因此必須加上 -k(--insecure) 參數
$ curl -k https://10.107.13.10:31770
... (略)
<h1>Welcome to nginx!</h1>
```

## 從內部驗證 HTTPS 服務

若從內部驗證，我們就可以使用已經存在的 TLS certificate 來進行驗證，首先套用以下設定：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: curl-deployment
spec:
  selector:
    matchLabels:
      app: curlpod
  replicas: 1
  template:
    metadata:
      labels:
        app: curlpod
    spec:
      volumes:
      - name: secret-volume
        secret:
          secretName: my-secure-nginx-secret
      containers:
      - name: curlpod
        command:
        - sh
        - -c
        - while true; do sleep 1; done
        image: radial/busyboxplus:curl
        volumeMounts:
        - mountPath: /etc/nginx/ssl
          name: secret-volume
```

接著取得 pod 的名稱並進行驗證：

```bash
# 取得 pod 名稱
$ kubectl get pods -l app=curlpod
NAME                               READY   STATUS    RESTARTS   AGE
curl-deployment-78959f7dcc-4csbk   1/1     Running   0          13s

# 透過 pod 中的 curl 命令搭配 CA 憑證來驗證 (此時就不需要 -k 參數了)
$ kubectl exec curl-deployment-78959f7dcc-4csbk -- curl https://my-secure-nginx --cacert /etc/nginx/ssl/nginx.crt
... (略)
<h1>Welcome to nginx!</h1>
... (略)
```

看到 **Welcome to nginx!** 且沒有憑證相關問題的警告就沒錯啦!



References
==========

- [Connecting Applications with Services - Kubernetes](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/#exposing-the-service)