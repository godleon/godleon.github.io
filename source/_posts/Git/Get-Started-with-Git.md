---
layout: post
title:  "[Git] Get Started with Git"
description: "This article introduces the informations for getting started with Git"
date: 2016-10-29 23:50:00
published: true
comments: true
categories: [git]
tags: [Git]
---

介紹開始使用 Git 前調教 config 的相關資訊


初始化 Git Repository
=====================

```bash
# 初始化 git repository
[root@centos7 git-lab]# git init
Initialized empty Git repository in /root/git/git-lab/.git/
```

----------------------------------------------

Git config
==========

## 常用 Git config (~/.gitconfig)

```ini
[user]
    email = godleon@gmail.com
    name = godleon
[core]
    editor = vim
    
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
[color]
    ui = true
[alias]
    tree = log --all --graph --decorate=short --color --format=format:'%C(bold blue)%h%C(reset) %C(auto)%d%C(reset)\n         %C(black)[%cr]%C(reset)  %x09%C(black)%an: %s %C(reset)'

    logs = log --all --graph --decorate=short --color --format=format:'%C(bold blue)%h%C(reset)+%C(dim black)(%cr)%C(reset)+%C(auto)%d%C(reset)++\n+++       %C(bold black)%an%C(reset)%C(black): %s%C(reset)'

    stree = !bash -c '"                                                                             \
    while IFS=+ read -r hash time branch message; do                                            \
        timelength=$(echo \"$time\" | sed -r \"s:[^ ][[]([0-9]{1,2}(;[0-9]{1,2})?)?m::g\");     \
        timelength=$(echo \"16+${#time}-${#timelength}\" | bc);                                 \
        printf \"%${timelength}s    %s %s %s\n\" \"$time\" \"$hash\" \"$branch\" \"\";          \
    done < <(git logs && echo);"'

    logv = log --all --graph --decorate=short --color --format=format:'%C(bold blue)%h%C(reset)+%C(dim black)(%cr)%C(reset)+%C(auto)%d%C(reset)++\n+++       %C(bold black)%an%C(reset)%C(black): %s%C(reset)'

    vtree = !bash -c '"                                                                             \
    while IFS=+ read -r hash time branch message; do                                            \
        timelength=$(echo \"$time\" | sed -r \"s:[^ ][[]([0-9]{1,2}(;[0-9]{1,2})?)?m::g\");     \
        timelength=$(echo \"16+${#time}-${#timelength}\" | bc);                                 \
        printf \"%${timelength}s    %s %s %s\n\" \"$time\" \"$hash\" \"$branch\" \"$message\";  \
    done < <(git logv && echo);"'
```

## Git config 優先權

Git config 的優先權如下：

1. `.git/config`

2. `~/.gitconfig`

3. `/etc/gitconfig`

```bash
# 檢視 ".git/config" 的內容
[root@centos7 git-lab]# cat .git/config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true

# 顯示目前有效的 git config
[root@centos7 git-lab]# git config -l
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true

# 在 '/etc/gitconfig' 中的設定
[root@centos7 git-lab]# git config --system -l
fatal: unable to read config file '/etc/gitconfig': No such file or directory

# 在 ~/.gitconfig 中的設定
[root@centos7 git-lab]# git config --global -l
fatal: unable to read config file '/root/.gitconfig': No such file or directory
```

## 設定 & 移除 git config

```bash
# 以下的設定將會存在 ~/.gitconfig 中
# 若改成 "--system" 參數，則設定會存在 '/etc/gitconfig'
[root@centos7 git-lab]# git config --global user.name 'godleon'
[root@centos7 git-lab]# git config --global user.email 'godleon@gmail.com'
[root@centos7 git-lab]# git config --list
user.name=godleon
user.email=godleon@gmail.com
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true

# 移除 git config 設定
[root@centos7 git-lab]# git config --global --unset user.name
[root@centos7 git-lab]# git config --list
user.email=godleon@gmail.com
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true
```

## 設定 Git alias

跟 Linux 的 alias 很像，可以自訂簡單的命令來替代複雜的指令：

```bash
[root@centos7 git-lab]# git config alias.con 'config -l'
# 此命令等同於 "git config -l" 的效果
[root@centos7 git-lab]# git con
user.email=godleon@gmail.com
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true
alias.con=config -l
```

## 設定 editor & diff tool

- `git config --global color.ui true`：加上顏色

- `git config --global core.editor vim`：把預設的文字編輯器改為 vim

- `git config --global diff.tool meld`：把預設的 diff tool 改為 meld (要先安裝 meld，且這是 GUI tool，不適用 text mode)

- `git difftool`：查詢檔案差異

----------------------------------------------

將檔案存入 Git Repository
========================

## .gitignore

Git 追蹤檔案的方式，會將檔案 & 資料夾分成三類：

1. untracked
> 一開始所有檔案都是 untracked，執行 `git status` 可以看到 untracked 的檔案清單

2. tracked
> 當透過 `git add` 處理之後的檔案，狀態會變成 tracked，等待使用者進一步的 commit or cancel

3. ignored
> 會被 git 忽略處理的檔案，需要新增 `.gitignore` 檔案，並逐行宣告要忽略的檔案 (被設定忽略的檔案狀態不會變成 untracked)

以下為 `.gitignore` 設定範例：

```bash
# 可用萬用字元(# 開頭為註解)
*.txt
# 不要忽略 note.txt 檔案
!note.txt
# folder 資料夾 & 資料夾內的檔案通通忽略
folder
```

## git commit

### 1、從 tracked 復原到 untracked

```bash
# 加入 file1.txt，狀態變成 tracked
[root@centos7 git-lab]# git add file1.txt

# 從 git index 中移除，狀態變回 untracked
[root@centos7 git-lab]# git rm --cache file1.txt
```

### 2、從已經 commit 後的狀態回復原狀

首先先來看看如何顯示目前 commit 的狀態

```bash
# 此為自訂的 alias，請參考上方設定
[root@centos7 git-lab]# git tree
* 72f32d4  (HEAD, master)
|          [25 minutes ago]     root: commit of file1
* fbaedbb

[root@centos7 git-lab]# git log --graph --oneline --decorate --date=relative --all
* 72f32d4 (HEAD, master) commit of file1
* fbaedbb First commit

[root@centos7 git-lab]# git log --graph --abbrev-commit --decorate --date=relative --all
* commit 72f32d4 (HEAD, master)
| Author: root <godleon@gmail.com>
| Date:   23 minutes ago
|
|     commit of file1
|
* commit fbaedbb
  Author: root <godleon@gmail.com>
  Date:   9 hours ago

      First commit

# 取消剛剛 commit 的狀態，將 file1.txt 狀態回復到 untracked 的狀態
[root@centos7 git-lab]# git reset HEAD^
[root@centos7 git-lab]# git status
# On branch master
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#       .gitignore
#       file1.txt
#       file2.txt
nothing added to commit but untracked files present (use "git add" to track)

# 目前 HEAD 已經回到第一次  commit 的地方
[root@centos7 git-lab]# git tree
* fbaedbb  (HEAD, master)
```

----------------------------------------------

比較檔案差異 & 從 Git Repository 取回檔案
=======================================

## 比較檔案差異

git 提供了很多方式來檢視不同版本間的檔案差異，主要就是使用 `git diff` & `git difftool` 來完成此功能，以下用個簡單範例來說明：

```bash
# 列出目前的 commit 紀錄
[root@centos7 git-lab]# git tree
* 81618ad  (HEAD, master)
|          [2 hours ago]        godleon: 2nd commit for file1.txt
* fbaedbb
           [2 days ago]         root: First commit

# 目前 file1.txt 已經有修改
[root@centos7 git-lab]# git status
# On branch master
# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#   (use "git checkout -- <file>..." to discard changes in working directory)
#
#       modified:   file1.txt
#
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#       .gitignore
#       file2.txt
no changes added to commit (use "git add" and/or "git commit -a")

# 比較本地端檔案 & git repository 的差異
[root@centos7 git-lab]# git diff
diff --git a/file1.txt b/file1.txt
index 7cca94a..3aff906 100644
--- a/file1.txt
+++ b/file1.txt
@@ -1,2 +1,3 @@
 file1, line 1
 file1, line2
+file1, line3

# 將檔案 file1.txt 移到 staging 中
[root@centos7 git-lab]# git add file1.txt

# 比較 staging 與 HEAD 的版本差異
[root@centos7 git-lab]# git diff --cached file1.txt
diff --git a/file1.txt b/file1.txt
index 7cca94a..3aff906 100644
--- a/file1.txt
+++ b/file1.txt
@@ -1,2 +1,3 @@
 file1, line 1
 file1, line2
+file1, line3

# 比較不同 commit 之間指定檔案的差異
[root@centos7 git-lab]# git diff HEAD HEAD~1 file1.txt
diff --git a/file1.txt b/file1.txt
deleted file mode 100644
index 7cca94a..0000000
--- a/file1.txt
+++ /dev/null
@@ -1,2 +0,0 @@
-file1, line 1
-file1, line2
```

----------------------------------------------

References
==========

- [git-recipes](https://github.com/geeeeeeeeek/git-recipes/wiki)

- [Git Magic](http://www-cs-students.stanford.edu/%7Eblynn/gitmagic/intl/zh_tw/index.html)

- [我所记录的git命令（非常实用） - 糖糖果 - 博客园](http://www.cnblogs.com/fanfan259/p/4810517.html)

- [Git 版本控制系統(3) 還沒 push 前可以做的事 | ihower { blogging }](https://ihower.tw/blog/archives/2622)
