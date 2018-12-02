---
title: Hexo 首页不显示全文只显示预览
date: 2018-12-02 15:29:10
tags:
categories:
- HEXO
---
#### 问题
next是Hexo的一个博客主题，这个主题 ，首页显示的文章列表，每一遍都是全文，不方便概览。

#### 希望达到的效果
首页显示文章列表，列表里的每一篇文章只显示预览，不显示全文。

#### 解决方法

进入hexo博客项目的themes/next目录
用文本编辑器打开_config.yml文件
搜索"auto_excerpt",找到如下部分：

```
# Automatically Excerpt. Not recommand.
# Please use <!-- more --> in the post to control excerpt accurately.
auto_excerpt:
  enable: false
  length: 150
```

把enable改为对应的false改为true，然后hexo d -g，再进主页，问题就解决了！