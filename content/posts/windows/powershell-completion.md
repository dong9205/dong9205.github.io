---
title: "Powershell Completion"
subtitle: ""
date: 2023-11-30T16:30:48+08:00
lastmod: 2023-11-30T16:30:48+08:00
draft: false
author: ""
authorLink: ""
license: ""

tags: 
- "windows"

categories: 
- "documentation"
- "windows"

featuredImage: ""
featuredImagePreview: ""

summary: ""

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
  auto: true

mapbox:
share:
  enable: true
comment:
  enable: true
---

# powershell设置自动补全
```powershell
notepad.exe $PROFILE
hugo.exe completion powershell | Out-String | Invoke-Expression
```
