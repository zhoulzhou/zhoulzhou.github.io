---
title: Linux 命令
date: 2018-12-01 21:43:17
tags:
categories:
- 开发工具
---
 rm -rf 目录名    删除文件夹
-r 就是向下递归，不管多少目录文件都一并删除
-f 就是直接强行删除文件，不做任何提示

ls -al
显示所有目录文件包括隐藏的

cat  filename 
一次显示整个文件

cat  > filename
创建新文件  不能编辑已有文件

cat  file1  file2  >  file
将几个文件合并为一个文件

cat  -n  file1 > file2
把file1的内容加上行号后输入到file2上