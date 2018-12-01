---
title: Git 命令
date: 2018-12-01 21:36:44
tags:
categories:
- 开发工具
---
ssh-keygen
一路默认  然后在隐藏文件夹.ssh 找到key文件
>说明如下
>ssh-keygen  -t rsa -C 注释
>ssh-keygen用于为生成、管理和转换认证密钥，包括 RSA 和 DSA 两种密钥。
>密钥类型可以用-t选项指定。如果没有指定则默认生成用于SSH-2的RSA密钥。 
>-C  comment 提供一个新注释  
>-f  filename  指定密钥文件名。