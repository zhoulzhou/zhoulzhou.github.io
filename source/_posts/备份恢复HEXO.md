---
title: 备份恢复HEXO
date: 2018-11-24 17:32:55
tags:
categories:
- HEXO
---
### 备份hexo博客
//假设hexo文件夹是已经生成的hexo博客目录
//如果themes/next(风格名字目录)下面有.git，请删除这个.git文件夹。
``` bash
cd hexo
git init  //初始化本地仓库
git add source themes scaffolds _config.yml package.json package-lock.json  //将必要的文件依次添加
git commit -m "blog hexo"
git branch hexo  //新建hexo分支
git checkout hexo  //切换到hexo分支上
git remote add origin git@github.com:user/user.github.io.git  //将本地与Github项目对接
git push origin hexo  //push到Github项目的hexo分支上
```

### 在其他终端克隆和更新hexo博客
克隆hexo博客环境
前提:nodejs,git,hexo已经安装好，并配置好环境变量。
``` bash
git clone -b hexo git@github.com:user/user.github.io.git  //将Github中hexo分支clone到本地
cd user.github.io
npm install
```
此时，这个文件夹就是hexo博客的本地副本了。

### 写新文章并备份和部署
//进入user.github.io文件夹,应是hexo分支
``` bash
git pull origin hexo //本地和远端的融合
hexo new post "new post name"  //写新文章
git add source
git commit -m "xxx"
git push origin hexo  //备份
hexo d -g  //部署
```
---------------------
