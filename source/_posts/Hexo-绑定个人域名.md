---
title: Hexo 绑定个人域名
date: 2018-12-01 19:38:12
tags:
categories:
- HEXO
---
#### 购买域名
在阿里云上购买域名很方便

#### 解析域名
添加如下的解析：

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/huxiu/he.PNG)

#### Github上仓库的pages设置
在项目的Settings中，添加Custom domain到自己的域名

![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/huxiu/he1.PNG)

#### 添加CNAME文件
在根目录下的source文件夹下新建CNAME文件，没有后缀。打开CNAME文件，在里边添加你的域名信息


#### HEXO部署
>hexo g
>hexo d