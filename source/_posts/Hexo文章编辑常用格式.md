---
title: Hexo文章编辑常用格式
date: 2018-12-02 15:19:31
tags:
- hexo
- 编辑
categories:
- HEXO
---
#### 添加网络图片

```
![avatar](https://github.com/zhoulzhou/MarkDownPhotos/raw/master/android/nei1.png)
```

#### 添加本地图片
修改_config.yml配置文件post_asset_folder项为true
把需要显示的图片放到对应博文的文件夹下

```
{% asset_img XXX.jpg %}
```

#### 添加标题
使用“#### 标题名”

#### 文字加粗
```
使用“**加粗的文字**”
```

#### 引用内容

```
>这是引用的内容
```


#### 代码格式
>代码上下各加三个 `