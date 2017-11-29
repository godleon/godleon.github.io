---
layout: post
title: "[OpenShift] How-to customize built-in Jenkins image"
description: "本文章介紹如何在 Jenkins pipeline 中計算工作執行的時間"
date: 2017-11-29 13:50:00
comments: true
published: true
tags: 
  - OpenShift
  - Jenkins
categories: 
  - OpenShift
---

最近的工作都是使用 OpenShift + Jenkins 在進行 CI/CD 的相關工作，在時間的顯示上遇到了兩個問題：

1. Jenkins 系統上使用的是 UTC 時間，希望改成 Taiwan 時間(+8:00)

2. 想要知道整個 pipeline 的工作完成後一共花了多少時間

上網找了很多資料，拼拼湊湊寫出了以下的 groovy script 來完成這件事情：

```groovy
import java.text.SimpleDateFormat
import java.util.Calendar
import groovy.time.*

// human-readable format
def dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")

// convert time from UTC to Taiwan Time(+8:00)
def startTime = new Date((Calendar.getInstance()).getTimeInMillis() + (480 * 60000))
def strStartTime = dateFormat.format(startTime)

sleep 5

openshift.withCluster() {
    stage('Calculate Time Duration') {
        node {

            def endTime = new Date((Calendar.getInstance()).getTimeInMillis() + (480 * 60000))
            strEndTime = dateFormat.format(endTime)
            
            // calculate time duration
            TimeDuration duration = TimeCategory.minus(endTime, startTime)
            def strDuration = String.format("%02d", duration.getHours()) + ":" + String.format("%02d", duration.getMinutes()) + ":" + String.format("%02d", duration.getSeconds())
            // it's necessary to set it to null for avoiding "not serializable exception"
            duration = null

            echo "Start Time = ${strStartTime}"
            echo "End Time = ${strEndTime}"
            echo "Time Duration = ${strDuration}"
        } 
    } // ---------- End of stage('Configure IAAS')
}
```

透過以下步驟，可以直接使用我放在 GitHub 上面的範例，直接建立一個 build job 來測試：

```bash
$ git clone https://github.com/godleon/learning_openshift.git

$ oc create -f learning_openshift/examples/pipeline-calculate-time-duration/bc-calculate-time-duration.yml

$ oc start-build calculate-time-duration
```

接著到 Jenkins 系統內部就可以看到執行結果囉!


--------------------------


關於 Script Approval 的處理
=========================

預設 Jenkins pipeline 會在 sandbox 的環境中執行，因此很多 grovvy or java method 都會無法使用，因此在 OpenShift 中需要對 Jenkins 進行客製化，預先設定 script whitelist，讓某些 method 可以在 pipeline 中使用，如何客製化 Jenkins 詳情可以參考之前寫過的文章。