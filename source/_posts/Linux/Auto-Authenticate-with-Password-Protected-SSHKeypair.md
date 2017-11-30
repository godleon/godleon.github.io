---
layout: post
title:  "[Linux] 解決使用帶有密碼的 SSH keypair 時需要重複輸入密碼的問題"
description: "This article introduces the how to solve the problem of repeating input password when you use a SSH keypair with a password"
date: 2016-07-01 10:00:00
published: true
comments: true
categories: [Linux]
tags: [Linux]
---

> use ssh agent and keychain to input the secret of password-protected ssh key

最近被 GitHub 騙了去產生個帶有密碼的 SSH keypair 來用

發現怎麼每次使用都要我輸入密碼呢....? 於是上網找了一下答案......

要解決這方式，需要 **ssh agent** 搭配 **keychain** 來將密碼安全的儲存起來

假設以下情況已經完成：

1. SSK keypair 已經存在於 `~/.ssh/id_rsa*`

2. keychain 套件已經安裝

接著只要執行以下指令：

```bash
$ tee --append ~/.bash_profile <<-'EOF'
### START-Keychain ###
# Let  re-use ssh-agent and/or gpg-agent between logins
/usr/bin/keychain $HOME/.ssh/id_rsa
source $HOME/.keychain/$HOSTNAME-sh
### End-Keychain ###
EOF
```

接著再重新登入，輸入一次密碼後，後續使用到 SSH keypair 時，就不用一直重複輸入了!



References
==========

- [Generating an SSH key - User Documentation](https://help.github.com/articles/generating-an-ssh-key/)

- [ssh-agent: How to set it up so my CentOS server will only ask for passphrase once? - Unix & Linux Stack Exchange](http://unix.stackexchange.com/questions/83608/ssh-agent-how-to-set-it-up-so-my-centos-server-will-only-ask-for-passphrase-onc)

- [keychain: Set Up Secure Passwordless SSH Access For Backup Scripts](http://www.cyberciti.biz/faq/ssh-passwordless-login-with-keychain-for-scripts/)
