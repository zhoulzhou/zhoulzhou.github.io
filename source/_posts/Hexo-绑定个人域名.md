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

{% asset_img he.PNG %}

#### Github上仓库的pages设置
在项目的Settings中，添加Custom domain到自己的域名

{% asset_img he1.PNG %}

#### 添加CNAME文件
在根目录下的source文件夹下新建CNAME文件，没有后缀。打开CNAME文件，在里边添加你的域名信息


#### HEXO部署
>hexo g
>hexo d

#### 解决TaskCanceledException
git提交
git remote add origin https://github.com/hellofriday/hellofriday.github.com.git
git add .
git commit -m 'add blog'
git push --set-upstream origin hexo

hexo clean
hexo g -d

