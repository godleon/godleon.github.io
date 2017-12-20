---
layout: post
title:  "[RHCE7] RH134 Chapter 3 Creating and Editing Text Files with vim 學習筆記"
description: "此文章記錄學習 RHCE7 RH134 Chapter 3 Creating and Editing Text Files with vim 留下的內容"
date: 2016-05-05 04:20:00
published: true
comments: true
categories: [RHCE]
tags: [Linux, RHCE, RH134]
---

3.3 Basic vim Workflow
======================

## Editor basics

`i`：進入 insert mode

`I`：進入 insert mode，從游標所在行首插入新字元(`i` + `HOME`)

`A`：`i` + `END`

`R`：replace mode，所輸入的會取代原本的內容

`o`：在游標上方插入新的一行，並進入 insert mode

`O`：在游標下方插入新的一行，並進入 insert mode

`:n`：移到第 n 行

`:$`：移到最後一行

`u`：undo

`Ctrl + r`： redo

`w`(往後移動一個 word) & `b`(往前移動一個 word)：可用 Ctrl + 左右鍵 取代

`DEL` or `x`：刪除一個字元

`20dd`：刪除 20 行

`5yy`：複製 5 行

------------------------------------------------------------------------

3.4 Editing with Vim
====================

## Search and Replace

簡單來說，就是在 vim 中使用 sed 的功能

`:1,$s/the/*****/g`：將檔案中的 then 全部換成星號
