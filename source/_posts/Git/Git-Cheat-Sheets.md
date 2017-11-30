---
layout: post
title:  "[Git] Cheat Sheet"
description: "Git Cheat Sheet"
date: 2016-11-08 16:20:00
published: true
comments: true
categories: [Git]
tags: [Git]
---


The Basics
==========

```bash
# Create a Git repository in the current folder.
$ git init

# View the status of each file in a repository.
$ git status

# Stage a file for the next commit.
$ git add <file>

# Commit the staged files with a descriptive message.
$ git commit

# View a repository’s commit history.
$ git log

# 以單行的方式顯示 git log (commit log)
$ git log --oneline

# 以單行的方式顯示與指定檔案相關的 git log
$ git log --oneline <file_name>

# 詳細列出所有 commit 的歷史紀錄
$ git log --graph --abbrev-commit --decorate --date=relative --all
    
# Define the author name to be used in all repositories.
$ git config --global user.name "<name>"
    
# Define the author email to be used in all repositories. 
$ git config --global user.email <email>
```


----------------------------------------------


Undoing Changes
===============

```bash
# 將 HEAD 指標移到指定的 commit or tag(sha1_checksum_prefix 可由 "git log --oneline" 取得)
# 若 git log 沒有加上 "--all" 參數，就會看不到比 HEAD 更新的 commit snapshot
$ git checkout <commit-id | tag-name>

# 回到最新的 commit 紀錄("master" 會固定指向最新一筆 commit snapshot)
$ git checkout master

# 為目前最新的 commit 加上一個 annotated tag (通常 tag 用來作為正式 release 的標記用途)
$ git tag -a <tag-name> -m "<description>"

# 列出目前所有的 tag 紀錄
$ git tag

# undo 指定的 commit (重要! 這並非是回到特定 commit 的概念)
# 即使 commit 被 undo，其實歷史紀錄還是永遠保留著，這是 git 的特性
$ git revert <commit-id>

# 取消所有 tracked 的變更，回到最新的 commit (但不包含 untracked 的部分，例如：新增的檔案)
$ git reset --hard

# 移除 untracked files (與上一個指令搭配使用，此時應該回復到一個乾淨的 working directory，亦即為最新的 commit snapshot)
$ git clean -f
```

> 需要注意 `git reset --hard` + `git clean -f` 的使用，是在 working directory 上生效，並不是在 commit snapshot 上，因此一旦 undo 之後，所有尚未 commit 的變更都會完全消失且無法追溯，請確定真的不要這些變更之後，再執行這兩個指令


----------------------------------------------


Branches
========

```bash
# 顯示所有的 branch (星號的部分表示目前所在的 branch)
$ git branch

# 使用目前的 working directory 作為基礎，新增 branch
$ git branch <branch-name>

# 將 HEAD 指標以及 working directory 移到指定的 branch
$ git checkout <branch-name>

# 將指定的 branch 與目前所在的 branch(checked-out) 做合併
$ git merge <branch-name>

# 刪除指定的 branch
$ git branch -d <branch-name>

# 強制刪除尚未合併的 branch (將會永遠遺失所有的檔案變更)
$ git branch -D <branch-name>

# 從 working directory 中移除 & 停止追蹤指定的檔案
$ git rm <file>

# 修改檔案名稱
$ git mv <old-filename> <new-filename>

# 使用一行指令直接 commit 所有狀態為 tracked 的檔案，並指定 commit 內容
$ git commit -a -m "<message>"
```

以下 **master** branch 合併 **css** branch 的狀況稱為 `fast-forward merge`：

![fast-forward merge](http://rypress.com/tutorials/git/media/3-10.png)

> 因為 css branch 是從 master branch 中最新的 commit snapshot 所延伸出來，所以 master branch 合併 css branch 是不需要做什麼額外的判斷處理

使用 branch 的基本原則：

1. 為每一個主要的新增功能，都使用 branch 的方式來完成

2. 若無法為 branch 取一個實際的名稱(無法定義修改的內容為何)，就不要使用 branch

!3-way merge](http://rypress.com/tutorials/git/media/4-1.png)

> 3-way merge 在當要合併兩個擁有不同 commit snapshot 時會發生，此時 Git 會額外建立一個 merge commit snapshot，並同時指向兩個不同的 branch(如上圖中的 **After**，紅色圈圈表示這個 commit snapshot 同時來自 crazy & master 兩個 branch)

> fast-forward merge 不會在 project history 中看到，這是與 3-way merge 不同的地方

----------------------------------------------


Rebasing
========

當整個 git 專案越來越多 branch 時，可以透過 rebase 的方式來整理 branch，避免過於凌亂；此外，rebase 也有在不進行 merge 的情況下，取得最新版 master 的效果。

透過 rebase 可以把 branch 指向指定 branch 的最新 commit snapshot；舉例來說，可將多餘的 branch 重新 rebase 後指向 master 最新的 commit snapshot，如此一來後續進行 merge 時就會變成 **fast-forward merge**，相關的 commit snapshot 都變成了 linear history，這樣在後續檢視上會更為直覺。

> rebase 雖然可以讓整體專案的 branch 更為簡潔易讀，但在某些 rebase 操作上會移除(or 修改)某些 commit snapshot，因此後續就會無法還原當初 commit snapshot 的全貌，這也是 rebase 功能有所爭議的地方

```bash
# 檢視 branch-feature 是否有落後 branch-dev，有顯示 commit snapshot 則表示落後，可考慮進行 rebase
$ git log <branch-feature>..<branch-dev>

# 將目前的 branch 的 root commit snapshot 指向 new-base 最新的 commit snapshot
$ git rebase <new-base>

# 以互動的方式進行 rebse 的設定，並可選擇對每個 commit snapshot 所要執行的動作
$ git rebase -i <new-base>

# 修改已經存在的 commit snapshot，不產生新的 (搭配上面 rebase 編輯 commit snapshop list 時使用 edit/squash 等關鍵是)
$ git commit --amend

# 完成修改了特定的 commit snapshot 資訊後，繼續進行 rebase 工作
$ git rebase --continue

# 放棄目前的 rebase 結果並回到原先的狀態
git rebase --abort

# 強制 git 不以 fast-forward 的方式進行 merge，藉此保留 branch 的相關資訊
git merge --no-ff <branch-name>
```

以下用圖說明執行 `git merge` 指令時有無加上 `--no-ff` 參數所產生的效果：

![git merge](https://i0.wp.com/farm6.static.flickr.com/5054/5488984566_359f74ecc2.jpg?resize=463%2C414&ssl=1)


----------------------------------------------


Rewriting History
=================

```bash
# 顯示本地端所有的 commit snapshot (以時間排序，包含已經被 reset 的 commit snapshot)
$ git reflog

# 將 HEAD 指標往前移到第 n 的 commit snapshot，但不變更 working directory 中的內容
# 比 HEAD~<n> 還新的 commit snapshot 中的變更會保留
$ git reset --mixed HEAD~<n>

# 將 HEAD 指標往前移到第 n 的 commit snapshot，但會變更 working directory 中的內容
# 比 HEAD~<n> 還新的 commit snapshot 中的變更會被移除
$ git reset --hard HEAD~<n>

# 顯示從 <since> & <until> 兩個 branch 之間的 commit snapshot 紀錄
$ git log <since>..<until>

# 在 git log 顯示訊息中額外包含被修改的檔案資訊
$ git log --stat 
```


----------------------------------------------


Remotes
=======

```bash
# 從遠端複製指定的 Git repository 回來
$ git clone <remote-path>

# 列出 remote repository 列表(若加上 -v 參數會顯示每個 remote repository 的詳細位址)
$ git remote

# 新增一筆 remote repository 記錄
$ git remote add <remote-name> <remote-path>

# 取得 remote repository 的指定 branch (不進行 merge)
$ git fetch <remote-name>

# merge 指定的 remote repository branch 到目前所在的 branch
$ git merge <remote-name>/<branch-name>

# 列出 remote repository branch
$ git branch -r

# 將 local repository branch 推送到 remote repository branch 進行更新
$ git push <remote-name> <branch-name>

# 為 remote repository 加上一個 tag
$ git push <remote-name> <tag-name>
```


----------------------------------------------


Centralized Workflows
=====================

```bash
# 建立 git repository，但沒有 working directory (只是用來存放檔案用)
$ git init --bare <repository-name>

# 移除 remote connection
$ git remote rm <remote-name>
```


----------------------------------------------


Patch Workflows
===============

![patch workflows](http://rypress.com/tutorials/git/media/10-2.png)

```bash
# 建立 patch，包含了目前所在的 branch 有的，卻沒有在 <branch-name> branch 出現的 commit snapshot
$ git format-patch <branch-name>
    Create a patch for each commit contained in the current branch but not in <branch-name>. You can also specify a commit ID instead of <branch-name>.

# 套用  patch 到目前的 branch
$ git am < <patch-file>
```


----------------------------------------------


Tips & Tricks
=============

```bash
# 備份目前的 branch，但只留下最新的一筆 commit snapshot
$ git archive <branch-name> --format=zip --output=<file>

# 完整匯出指定的 branch 到指定檔案(會包含所有 commit snapshot 記錄)
$ git bundle create <file> <branch-name>

# 透過 bundled repository 重新建立一個 repository 並 checkout 指定的 branch
$ git clone repo.bundle <repo-dir> -b <branch-name>

# 暫時將變更隱藏，將目前的目錄變為乾淨的 working directory
$ git stash

# 將隱藏的變更套用到 working directory
$ git stash apply

# 檢視兩個 commit snapshot 之間的差異
$ git diff <commit-id>..<commit-id>

# 檢視目前 working directory 與 staging area 的差異
$ git diff

# 檢視目前 staging area 與最新一筆 commit snapshot 之間的差異
$ git diff --cached

# 將指令檔案從 commit snapshot 中移到 staged snapshot 中
$ git reset HEAD <file>
    Unstage a file, but don’t alter the working directory or move the current branch.

# 從指定的 commit snapshot 中取得指定檔案
$ git checkout <commit-id> <file>

# 設定 git command alias
$ git config --global alias.<alias-name> <git-command>
```

**git reset**：
![git reset](http://rypress.com/tutorials/git/media/11-1.png)

**git checkout**：
![git checkout](http://rypress.com/tutorials/git/media/11-2.png)


----------------------------------------------


Plumbing
========

```bash
# 顯示指定的 object 內容(其中 <type> 可以是 commit / tree / blob / tag)
$ git cat-file <type> <object-id>
```

![commit and tree objects](http://rypress.com/tutorials/git/media/12-1.png)


```bash
# 查詢指定的 object id 是屬於哪種 type
$ git cat-file -t <object-id>

# 列出指定 tree 物件的內容
$ git ls-tree <tree-id>
040000 tree 5aa02e7f90df11621262d7fe91a9357bb44494aa	about
100644 blob 4838a99c7bb4cc75941ff0a5e3a05fe4889570f9	blue.html
.......
100644 blob c9d942d8aadb84617d78455f8e2da25866c079a2	style.css
100644 blob e9d1781fd949fd41d2439ae3824a293531bc38a5	yellow.html

# 顯示指定 object id(這裡指的是 blob 物件) 的內容
$ git cat-file blob e9d178
```

![commit, tree, and blob objects](http://rypress.com/tutorials/git/media/12-2.png)


```bash
# 對 git object database 進行 garbage collection 
$ git gc

# 將指定的檔案從 working directory 移到 staged area (新增的檔案要額外加上 --add 參數)
$ git update-index [--add] <file>
    Stage the specified file, using the optional --add flag to denote a new untracked file.

# 從目前的 index 中產生一個 tree 物件，並存入 object database 中
$ git write-tree

# 從指定的 tree object & parent commit 中產生一個新的 commit 物件
$ git commit-tree <tree-id> -p <parent-id> 
```


----------------------------------------------


References
==========

- [Ry’s Git Tutorial - RyPress](http://rypress.com/tutorials/git/index)

- [Git-rebase 小筆記 - Yu-Cheng Chuang’s Blog](https://blog.yorkxin.org/2011/07/29/git-rebase)

- [Git 版本控制 branch model 分支模組基本介紹 \| 小惡魔 - 電腦技術 - 工作筆記 - AppleBOY](https://blog.wu-boy.com/2011/03/git-%E7%89%88%E6%9C%AC%E6%8E%A7%E5%88%B6-branch-model-%E5%88%86%E6%94%AF%E6%A8%A1%E7%B5%84%E5%9F%BA%E6%9C%AC%E4%BB%8B%E7%B4%B9/)

- [洁癖者用 Git：pull --rebase 和 merge --no-ff](http://hungyuhei.github.io/2012/08/07/better-git-commit-graph-using-pull---rebase-and-merge---no-ff.html)

- [Laradebut #03 從 git 入門到團隊合作開發 // Speaker Deck](https://speakerdeck.com/mouson/laradebut-number-03-cong-git-ru-men-dao-tuan-dui-he-zuo-kai-fa)

- [Tag 列表頁(git) - iT 邦幫忙::一起幫忙解決難題，拯救 IT 人的一天](http://ithelp.ithome.com.tw/tags/articles/git?page=2)