---
title: Git 命令
date: 2018-12-01 21:36:44
tags:
categories:
- 开发工具
---
#### 提交代码：
>git stash//保存修改
>git pull --rebase origin branch
>git stash pop//恢复修改
>修改的目录下：git add .
>git commit 
>git push origin branch

#### ssh-keygen
一路默认  然后在隐藏文件夹.ssh 找到key文件
>ssh-keygen  -t rsa -C 注释
>ssh-keygen用于为生成、管理和转换认证密钥，包括 RSA 和 DSA 两种密钥。
>密钥类型可以用-t选项指定。如果没有指定则默认生成用于SSH-2的RSA密钥。 
>-C  comment 提供一个新注释  
>-f  filename  指定密钥文件名。

clone时出现，unable to negotiate with 10.0.0.8: no matching key exchange methodfound. Their offer: diffie-hellman-group1-sha1
解决方法：在执行git clone之前，在终端输入：
>export GIT_SSH_COMMAND='ssh -o KexAlgorithms=+diffie-hellman-group1-sha1'
打开.bashrc文件，在终端输入：$ vim ~/.bashrc  ，然后向.bashrc文件写入上面的命令
