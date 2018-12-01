---
title: ADB 命令
date: 2018-12-01 21:46:52
tags:
categories:
- 开发工具
---
查看连接设备
adb devices


安装一个apk
adb install <apkfile>


保留数据和缓存文件，重新安装apk：
adb install -r <apkfile>


启动/停止 Server
adb start-server
adb kill-server


重启
adb reboot


查看屏幕分辨率
adb shell wm size


查看屏幕密度
adb shell wm density