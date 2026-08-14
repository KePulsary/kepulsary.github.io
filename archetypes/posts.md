---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
updated: {{ .Date }}
tags: []
draft: false
# 想用自己的封面？把图片放进本目录，取消下一行注释：
# image: cover.jpg
---

<!-- 图片直接放本目录，正文里用 ![](文件名.png) 引用；不写 image 则自动生成封面 -->
