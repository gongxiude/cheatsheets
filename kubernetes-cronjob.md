---
title: CronJob
layout: 2017/sheet
prism_languages: [go, bash]
weight: -3
tags: [Featured]
category: Kubernetes
updated: 2021-06-10
---

## kuberetes Cronjob 
{: .-one-column}

CronJob 可以帮我们周期性的运行脚本或者进行其它方面的工作,  其时间的表现方式和linux系统cronjob的格式相同,  分、时、日、月、周

```
Cron 时间表语法
# ┌───────────── 分钟 (0 - 59)
# │ ┌───────────── 小时 (0 - 23)
# │ │ ┌───────────── 月的某天 (1 - 31)
# │ │ │ ┌───────────── 月份 (1 - 12)
# │ │ │ │ ┌───────────── 周的某天 (0 - 6) （周日到周一；在某些系统上，7 也是星期日）
# │ │ │ │ │                                   
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```
