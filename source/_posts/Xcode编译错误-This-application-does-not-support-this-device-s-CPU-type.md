title: 'Xcode编译错误:This application does not support this device''s CPU type'
author: Cyrus
tags:
  - Xcode问题
categories:
  - iOS
date: 2018-10-31 22:24:00
---
最近运行一个旧项目代码，出现错误提示：This application does not support this device's CPU type
原因：Xcode设置了32-bit architecture，而手机是64-bit的CPU。
解决方法：修改build Setting->Architectures 为 **architectrues - $(ARCHS_STANDARD)**
![](buildSetting.png)