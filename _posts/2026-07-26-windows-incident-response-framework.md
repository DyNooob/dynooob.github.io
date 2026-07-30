---
layout: post
title: "Windows 应急响应排查框架：从接报到定位入侵"
date: 2026-07-26 19:00:00 +0800
categories: [安全]
tags: [Windows, 应急响应, 入侵排查, 安全运维]
---

做应急响应多了就会发现，大多数入侵不是多高级的手法，而是没按流程排查，漏了关键节点。

## 通用排查思路

别上来就敲命令。第一步永远是问清楚：

- 谁发现的异常？什么时间？
- 异常现象是什么（CPU 爆了？文件被加密？有陌生人登录？）
- 受害方做了什么处置（有没有关机、杀进程、删文件——这些操作可能破坏现场）

问清楚后，先跑一条 `systeminfo` 把机器基本信息捞出来：主机名、OS 版本、安装时间、启动时间、补丁列表。有条件的话全程录屏。

从现象反推可能的入侵路径，再从路径反查对应的 Windows 痕迹点。盲查效率极低，而且容易破坏残留证据。

## 查账号

弱口令加 3389 对公网是 Windows 被拿下的最常见入口。先问管理员：RDP 是否对公网开放，密码复杂度如何。

命令行跑 `lusrmgr.msc`，进本地用户和组。重点看 Administrators 组里有没有不认识的账号。留意用户名带 `$` 结尾的隐藏账号和克隆账号。注册表路径 `HKLM\SAM\SAM\Domains\Account\Users` 需要提权才能查看。

查登录日志：`eventvwr.msc`，筛选安全日志，事件 ID 4624 是成功登录。重点关注非工作时间、异地 IP 的登录记录。

## 查端口和进程

定位 ESTABLISHED 状态的连接，发现可疑外部 IP 扔威胁情报平台查归属。拿到 PID 后用 `tasklist | findstr "PID"` 定位进程。

重点关注这几类进程：

- 没有数字签名的
- 没有描述信息的
- 属主不是 SYSTEM 或正常用户的
- 路径不合法（跑在 Temp、下载目录的）
- CPU/内存长期高占用的

工具方面，Process Explorer、火绒剑、PC Hunter、Process Hacker 轮流上。拿到可疑文件的 MD5 扔微步或 VirusTotal 查。

## 查启动项、计划任务、服务

启动项重点看注册表这几个位置：

- `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`
- `HKLM\Software\Microsoft\Windows\CurrentVersion\Run`
- `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\WinLogon`

用 Autoruns 工具扫更省事，启动项、服务、驱动、浏览器插件一网打尽。

计划任务：`taskschd.msc` 打开任务计划程序，逐个看属性和触发条件。重点看执行路径是否指向可疑目录。计划任务的 XML 配置文件落在 `C:\Windows\System32\Tasks` 等目录，有些马会藏在这里，不通过 GUI 看不出来。

服务自启动：`services.msc`，看启动类型为「自动」的服务，注意那些没描述、没签发者的。

## 排查流程总结

```
接到告警 → 问清情况 → systeminfo 快照
  → 查账号（弱口令、新增账号、登录日志）
  → 查端口进程（ESTABLISHED 连接、可疑进程）
  → 查持久化（启动项、计划任务、服务）
  → 后续（系统信息、病毒查杀、日志分析）
```

上篇覆盖从接报到定位持久化的前半段。把「人怎么进来的」和「怎么赖着不走」搞清楚，才能进行后续的清理和加固。