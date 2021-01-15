---
layout: post
title:  "[Debug] SSH Tunnel 無法正常使用"
description: "此篇文章紀錄了使用 SSH tunnel 遇到了異常(open failed: administratively prohibited: open failed)，排除的過程 & 方法"
date: 2021-01-15 11:45:00
published: true
comments: true
categories:
  - Debug
tags:
  - Debug
  - SSH
---


今天一如往常的使用類似下面的指令設定了 SSH Tunnel：
> ssh ubuntu@my.bastion.com -N -L 9200:mydomain.com.tw:443

結果在本機開啟 9200 port 時，出現了以下的錯誤訊息：

```bash
channel 2: open failed: administratively prohibited: open failed
```

上網查了一下，總結一下以後，一共要確認以下兩點：

1. 在 `/etc/ssh/sshd_config` 要有 `AllowTCPForwarding yes` 的設定

2. Bastion node 要有能力對上面範例中的 `mydomain.com.tw` 進行域名解析

關於 `AllowTCPForwarding` 的部份，基本上預設值就是 `yes`，所以只要確認沒有 `AllowTCPForwarding no` 存在即可。

至於域名解析的部份就比較雷，今天查了一兩個小時才發現這個問題，原來機器上的 DNS server 設定跑掉了，導致於 SSH tunnel 一直不正常，因此下次遇到這一類的問題，記得確認一下 DNS 解析的部份是正常的。


References
==========

- [amazon ec2 - Trying to make a SSH Tunel - Stack Overflow](https://stackoverflow.com/questions/34402682/trying-to-make-a-ssh-tunel)

- [Gentoo Forums :: View topic - SSH service no work.. [SOLVED]](https://forums.gentoo.org/viewtopic-t-659363-start-0.html)