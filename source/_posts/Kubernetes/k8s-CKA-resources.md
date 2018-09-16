---
layout: post
title:  "CKA(Certified Kubernetes Administrator) 學習資源整理"
description: "本篇文章蒐集與 CKA(Certified Kubernetes Administrator) 考試範圍的相關學習資源"
date: 2018-08-26 17:40:00
published: true
comments: true
categories:
  - Kubernetes
tags:
  - Kubernetes
  - CKA
---

本篇文章蒐集與 CKA(Certified Kubernetes Administrator) 考試範圍的相關學習資源整理....會根據學習進度陸續補上


# Installation, Configuration & Validation

## 內容範圍

- Design a Kubernetes cluster.

- Install Kubernetes masters and nodes, including the use of TLS bootstrapping.

- Configure secure cluster communications.

- Configure a Highly-Available Kubernetes cluster.

- Know where to get the Kubernetes release binaries.

- Provision underlying infrastructure to deploy a Kubernetes cluster.

- Choose a network solution.

- Choose your Kubernetes infrastructure configuration.

- Run end-to-end tests on your cluster.

- Analyse end-to-end tests results.

- Run Node end-to-end tests

## 官網資料

- [Creating a Custom Cluster from Scratch - Kubernetes](https://kubernetes.io/docs/setup/scratch/)

## 其他參考資料


---

# Core Concept

## 內容範圍

- Understand the Kubernetes API primitives.

- Understand the Kubernetes cluster architecture.

- Understand Services and other network primitives.

## 官網資料

- [Nodes - Kubernetes](https://kubernetes.io/docs/concepts/architecture/nodes/)

- [Master-Node communication - Kubernetes](https://kubernetes.io/docs/concepts/architecture/master-node-communication/)

- [Concepts Underlying the Cloud Controller Manager - Kubernetes](https://kubernetes.io/docs/concepts/architecture/cloud-controller/)

- [Kubernetes Design and Architecture](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md)


## 其他參考資料

- [Kubernetes架构 · Kubernetes Handbook - Kubernetes中文指南/云原生应用架构实践手册 by Jimmy Song(宋净超)](https://jimmysong.io/kubernetes-handbook/concepts/)

- [架构原理 · Kubernetes Handbook](https://kubernetes.feisky.xyz/zh/architecture/architecture.html)

- [Kubernetes指南 - 架构原理 · Kubernetes Handbook](https://kubernetes.feisky.xyz/zh/architecture/architecture.html)

---


# Networking

## 內容範圍

- Understand the networking configuration on the cluster nodes.

- Understand Pod networking concepts.

- Understand service networking.

- Deploy and configure network load balancer.

- Know how to use Ingress rules.

- Know how to configure and use the cluster DNS.

- Understand CNI

## 官網資料

## 其他參考資料

- [[netfilter] Introduction to ebtables | Hwchiu Learning Note](https://www.hwchiu.com/netfilter-eiptables-i.html)

- [[netfilter] Introduction to iptables | Hwchiu Learning Note](https://www.hwchiu.com/netfilter-eiptables-iai.html)

- [[Kubernetes] How To Implemete Kubernetes Service - ClusterIP | Hwchiu Learning Note](https://www.hwchiu.com/kubernetes-service-ii.html)

- [[Kubernetes] How To Implemete Kubernetes Service - NodePort | Hwchiu Learning Note](https://www.hwchiu.com/kubernetes-service-iii.html)

- [[Kubernetes] How Implemete Kubernetes Service - SessionAffinity | Hwchiu Learning Note](https://www.hwchiu.com/kubernetes-service-iiii.html)


---


# Storage

## 內容範圍

- Understand persistent volumes and know how to create them.

- Understand access modes for volumes.

- Understand persistent volume claims primitive.

- Understand Kubernetes storage objects.

- Know how to configure applications with persistent storage.

## 官網資料

## 其他參考資料


---

# Scheduling

## 內容範圍

- Use label selectors to schedule Pods.

- Understand the role of DaemonSets.

- Understand how resource limits can affect Pod scheduling.

- Understand how to run multiple schedulers and how to configure Pods to use them.

- Manually schedule a pod without a scheduler.

- Display scheduler events.

- Know how to configure the Kubernetes scheduler

## 官網資料

## 其他參考資料

---

# Logging & Monitoring

## 內容範圍

- Understand how to monitor all cluster components.

- Understand how to monitor applications.

- Manage cluster component logs.

- Manage application logs.

## 官網資料

## 其他參考資料

---

# Application Lifecycle Management

## 內容範圍

- Understand Deployments and how to perform rolling updates and rollbacks.

- Know various ways to configure applications.

- Know how to scale applications.

- Understand the primitives necessary to create a self-healing application.

## 官網資料

## 其他參考資料

---

# Cluster Maintenance

## 內容範圍

- Understand Kubernetes cluster upgrade process.

- Facilitate operating system upgrades.

- Implement backup and restore methodologies

## 官網資料

## 其他參考資料

---

# Security

## 內容範圍

- Know how to configure authentication and authorization.

- Understand Kubernetes security primitives.

- Know to configure network policies.

- Create and manage TLS certificates for cluster components.

- Work with images securely.

- Define security contexts.

- Secure persistent key value store.

- Work with role-based access control.

## 官網資料

## 其他參考資料

---

# Troubleshooting

## 內容範圍

- Troubleshoot application failure.

- Troubleshoot control plane failure.

- Troubleshoot worker node failure.

- Troubleshoot networking.

## 官網資料

## 其他參考資料

---

# 完整學習資料整理

- [Kubernetes Handbook——Kubernetes中文指南/云原生应用架构实践手册](https://jimmysong.io/kubernetes-handbook/)

- [walidshaari/Kubernetes-Certified-Administrator](https://github.com/walidshaari/Kubernetes-Certified-Administrator)

> 裏面有熱心網友整理了相當多寶貴的資料....

- [LiNUX Courses - Certified Kubernetes Administrator (CKA)](https://linuxacademy.com/linux/training/course/name/certified-kubernetes-administrator-preparation-course)

> 沒付費可以有七天的無限制存取權限，用來快速的了解整體內容 or 做複習衝刺應該會有幫助

- [nikovirtala/Certified-Kubernetes-Administrator-CKA](https://github.com/nikovirtala/Certified-Kubernetes-Administrator-CKA)



References
==========

- [K8S CKA Study Prep](https://www.youtube.com/watch?v=tqr581_bBM0&list=PLrKlqis8aRHbMYWrNTsbx_MD9zJxjRLIc)

- [CKA Study](https://www.youtube.com/watch?v=zeS6OyDoy78&list=PLX-REfIYjgkzkWPRlLxSTuGn743cKD1GH)