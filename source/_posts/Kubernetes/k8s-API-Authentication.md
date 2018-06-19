---
layout: post
title:  "了解 Kubernetes 中的認證機制"
description: "本篇文章介紹在 Kubernetes 中存取 API 時的認證機制"
date: 2018-06-19 10:00:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
---


由於 Kubernetes 整體就是以 API 為基礎來設計的，因此要了解認證機制也就等同於要了解 k8s API server 的認證機制。


了解 Kubernetes API 認證機制
==========================

k8s 提供了多種的認證方式，管理者可以設定多種認證機制共存,只要任何一種通過就算通過。

以 k8s 中各服務之間的認證為例，是以 x509 的認證，同時也是最嚴格的認證方式。透過 Kubespray 安裝好 k8s 後，我們可以在 master node 上檢視一下 api server 的啟動設定：(`/etc/kubernetes/manifests/kube-apiserver.manifest`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
  ........(略)
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirst
  containers:
  - name: kube-apiserver
    image: gcr.io/google-containers/hyperkube:v1.10.4
    imagePullPolicy: IfNotPresent
    resources:
      limits:
        cpu: 800m
        memory: 2000M
      requests:
        cpu: 100m
        memory: 256M
    command:
    - /hyperkube
    - apiserver
    - --advertise-address=10.103.10.51
    - --etcd-servers=https://10.103.10.51:2379,https://10.103.10.52:2379,https://10.103.10.53:2379
    - --etcd-cafile=/etc/ssl/etcd/ssl/ca.pem
    - --etcd-certfile=/etc/ssl/etcd/ssl/node-kube-master0.pem
    - --etcd-keyfile=/etc/ssl/etcd/ssl/node-kube-master0-key.pem
    - --insecure-bind-address=127.0.0.1
    - --bind-address=0.0.0.0
    ........(略)
    - --client-ca-file=/etc/kubernetes/ssl/ca.pem
    - --kubelet-client-certificate=/etc/kubernetes/ssl/node-kube-master0.pem
    - --kubelet-client-key=/etc/kubernetes/ssl/node-kube-master0-key.pem
    - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
    - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
    - --service-account-key-file=/etc/kubernetes/ssl/service-account-key.pem
    - --secure-port=6443
    - --insecure-port=8080
    - --storage-backend=etcd3
    - --authorization-mode=Node,RBAC
    ........(略)
    - --requestheader-client-ca-file=/etc/kubernetes/ssl/front-proxy-ca.pem
    - --proxy-client-cert-file=/etc/kubernetes/ssl/front-proxy-client.pem
    - --proxy-client-key-file=/etc/kubernetes/ssl/front-proxy-client-key.pem
    livenessProbe:
    ........(略)
    volumeMounts:
    ........(略)
  volumes:
  - hostPath:
      path: /etc/kubernetes
    name: kubernetes-config
  - name: ssl-certs-host
    hostPath:
      path: /etc/ssl
  - name: usr-share-ca-certificates
    hostPath:
      path: /usr/share/ca-certificates
  - hostPath:
      path: /etc/ssl/etcd/ssl
    name: etcd-certs
```

從上面可以看出：

1. 與各個服務相互認證所使用的憑證 (`*.pem`)

2. 與本地端的其他服務則是透過 `--insecure-bind-address=127.0.0.1` & `--insecure-port=8080` 來溝通

3. 憑證存放的位置其實就是在各個 node 的本機上 (`hostPath` 定義)

4. 目前此 k8s cluster 支援的認證方式為 `[Node](https://kubernetes.io/docs/reference/access-authn-authz/node/)` & `[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)`

5. pod 內部服務需要與 api server 互動時的認證方式，使用 `[Service Account Token](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)` 的方式，此處對應的設定是 `--service-account-key-file=/etc/kubernetes/ssl/service-account-key.pem`


Kubespray 預設設置了 `X509`, `Node` & `RBAC` 三種認證方式，但 Kubernetes 還支援了相當多其他的認證方式，例如：

- `Static Token File`

- `Bootstrap Tokens`

- `Static Password File`

- `OpenID Connect Tokens`

- `Webhook Token Authentication`

- `Keystone Password`


以下會針對比較重要的認證機制詳細說明。 



X509 Client Certs
=================

從 Kubespray 預設的安裝設定中的 API server 啟動參數中，可以看到與 X509 相關設定參數有：

- `client-ca-file`

- `tls-private-key-file`

- `tls-cert-file`

由於只有在 k8s cluster 內的 node 才會有這些憑證檔案，因此這樣的認證方式是在 cluster 內部不同 node 之間的服務相互存取時使用。

但若是與 API server 同在本地端(localhost)的服務要存取 API 呢? 就直接 pass 了.....(像是 `kube-scheduler` & `kube-controller-manager` 都屬於這一類的服務)

可看到 `--insecure-bind-address=127.0.0.1` & `--insecure-port=8080` 兩個設定，就可以看出在本地端內部服務存取 API server 時，使用的是 **http://localhost:8080**



Service Account
===============

Service Account 本身在 k8s 中是屬於 resource 的一種。我們可以透過以下指令取得 Service Account 清單：

```bash
#
# 每一個 namespace 都有一個名稱為 "default" 的 service account
#
root@kube-master0:~# kubectl get serviceaccount --all-namespaces
NAMESPACE     NAME                                 SECRETS   AGE
default       default                              1         4d
kube-public   default                              1         4d
kube-system   attachdetach-controller              1         4d
kube-system   cronjob-controller                   1         4d
kube-system   daemon-set-controller                1         4d
kube-system   default                              1         4d
kube-system   deployment-controller                1         4d
.....(略)
```
從上面清單可看出，每個 namespace 中都會有一個名稱為 `default` 的 service account，接著可以來拆解這個 `default` service account 有什麼內容：

```bash
#
# 名稱為 "default" 的 service account 其實就是帶著用來存取 API server 時認證用的 token
#
root@kube-master0:~# kubectl describe serviceaccount/default -n kube-system
Name:                default
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-nkl69
Tokens:              default-token-nkl69
Events:              <none>
```

最後來看看上面 service account token 所帶的內容：

```bash
#
# 此 token 內容載明了 token 本身之外，還有 namespace, type ... 等資訊
#
root@kube-master0:~# kubectl describe secret/default-token-nkl69 -n kube-system
Name:         default-token-nkl69
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=default
              kubernetes.io/service-account.uid=fd2e5d64-6f9c-11e8-b6b3-065296fdbf18

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLW5rbDY5Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmZDJlNWQ2NC02ZjljLTExZTgtYjZiMy0wNjUyOTZmZGJmMTgiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.cMC6jKcj1vmDhQnxlsHzop1t6W3qeG-xhEAZyBuzagUgb-2f06ZxI0UvIQ-qb3mkXyjtCdhw-Hn4PpvJ56HtvCBMYwfaP-ii4ord0aZfhqIRynlFuj-cc2qvhewaGwC84Yj7awMj6rv9yQGMgFBEL0roaLgoVAyYqpbJ6B2ig4cBlQQnTbiewYXdGoWGzb3wtl2Ii6E9nZ6ANPxLI4dhwbAxVNVAR4tojRiukQldSnI0ItX-Iwx1Djd5FK3FCgxY1soo682sLE-_NJeF8KfVPAWtg1049dSXe1iNQ3k-AO3pfEGDBaAq4WACuYhMtfL2iYlZRxS5SEt8Mwz3aVW1ng
ca.crt:     1090 bytes
```

> Service Account 的認證方式主要是由 k8s 自行管理，當我們建立一個 namespace 時，就自動會產生一個名稱為 `default` 的 service account 並帶有新建立的 token；而未來在此 namespace 中所產生的 pod，都會自動使用此 token 與 API server 進行認證。



References
==========

- [Authenticating - Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

- [Kubernetes-- 漫谈kubernetes 中的认证 & 授权 & 准入机制 | Solar](https://zhangchenchen.github.io/2017/08/17/kubernetes-authentication-authorization-admission-control/)

- [https://kairen.github.io/2018/04/15/kubernetes/k8s-integration-ldap/](整合 OpenLDAP 進行 Kubernetes 身份認證 | KaiRen's Blog)