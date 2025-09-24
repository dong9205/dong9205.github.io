---
weight: 0
title: "Powershell Completion"
subtitle: ""
date: 2023-11-30T16:30:48+08:00
lastmod: 2023-11-30T16:30:48+08:00
draft: false
author: "Derrick"
authorLink: "https://www.p-pp.cn/"
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




---

# powershell设置自动补全
```powershell
notepad.exe $PROFILE
hugo.exe completion powershell | Out-String | Invoke-Expression
```
