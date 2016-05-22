---
layout: post
title: 上传文件时忽略某些文件或文件夹或特定类型的文件
date: 2016-02-08
tags: Git
categories: Be My Hero
---

在.gitignore中编辑
example:
```
#ignore database file, sln file, config file
*.mdb
*.ldb
*.sln
*.config

#ignore fileFolder
node_modules
Debug

在具体的.gitignore文件中，也有详细的说明
