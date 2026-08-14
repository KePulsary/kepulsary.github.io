---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
updated: {{ .Date }}
tags: []
description: ""
draft: false
# 标题与 description 记得改成规范写法（见 docs/workflow.md「命名与标签规范」）：
#   标题：writeup 类 <赛事> <年份> 题解，学习笔记类直接写主题，中英文混排加空格
#   description：一句话准确概括内容，禁"笔记/刷题/比赛wp"等含糊词
# 想用自己的封面？把图片放进本目录，取消下一行注释：
# image: cover.jpg
---

<!-- 图片直接放本目录，正文里用 ![](文件名.png) 引用；不写 image 则自动生成封面 -->
