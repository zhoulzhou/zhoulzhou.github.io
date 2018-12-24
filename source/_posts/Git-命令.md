---
title: Git 命令
date: 2018-12-01 21:36:44
tags:
- Git
categories:
- 开发工具
---
1.查看最新的commit
git show
2.查看指定commit hashID的所有修改：
git show commitId
3.查看某次commit中具体某个文件的修改：
git show commitId fileName

#### git checkout	
文件层面	舍弃工作目录中的更改
git checkout -- filename
git checkout .  舍弃当前目录中的所有的更改
如果上面不行 执行下面：
git reset HEAD -- filename
git checkout -- filename
git reset HEAD 来取消缓存区的所有的修改
git rm –cached “文件路径”，不删除物理文件，仅将该文件从缓存中删除；
git rm –f “文件路径”，不仅将该文件从缓存中删除，还会将物理文件删除（不会回收到垃圾桶）。

#### 提交代码：
>git stash//保存修改
>git pull --rebase origin branch
>git stash pop//恢复修改
>修改的目录下：git add .
>git commit 
>git push origin HEAD:refs/for/branch

#### 增补提交
1、git commit -C HEAD --amend  (不用经过提交模块过程)
   git commit --amend
2、git push origin HEAD:refs/for/branch

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


#### git的format-patch和am命令进行生成patch和打patch
推荐大家使用git的format-patch和am命令进行生成patch和打patch，用此方法获得的patch其实就是commit里提交的code修改以及commit信息。有如下好处：

1. 对于git这种以project为单位的修改，尤其是涉及到多个文件夹下的多个文件的改动时，非常方便，能够记录所有的改动（添加，修改，删除文件等）

2. 可以保存commit信息。

3. 能够灵活的获取patch。可以获取任意两个commit之间的patch集。

 

使用方法（直接给一些examples）：

git format-patch

$ git format-patch HEAD^ 　　　　　　　　　　　　　   #生成最近的1次commit的patch

$ git format-patch HEAD^^　　　　　　　　　　　　　  #生成最近的2次commit的patch

$ git format-patch HEAD^^^ 　　　　　　　　　　　　　#生成最近的3次commit的patch

$ git format-patch HEAD^^^^ 　　　　　　　　　　　      #生成最近的4次commit的patch

$ git format-patch <r1>..<r2>                                              #生成两个commit间的修改的patch（包含两个commit. <r1>和<r2>都是具体的commit号)
$ git format-patch -1 <r1>                                                   #生成单个commit的patch
$ git format-patch <r1>                                                       #生成某commit以来的修改patch（不包含该commit）
$ git format-patch --root <r1>　　　　　　　　　　　　   #生成从根到r1提交的所有patch
 
git am
$ git apply --stat 0001-limit-log-function.patch   　　　　  # 查看patch的情况
$ git apply --check 0001-limit-log-function.patch   　　　  # 检查patch是否能够打上，如果没有任何输出，则说明无冲突，可以打上
(注：git apply是另外一种打patch的命令，其与git am的区别是，git apply并不会将commit message等打上去，打完patch后需要重新git add和git commit，而git am会直接将patch的所有信息打上去，而且不用重新git add和git commit,author也是patch的author而不是打patch的人)
$ git am 0001-limit-log-function.patch                                # 将名字为0001-limit-log-function.patch的patch打上
$ git am --signoff 0001-limit-log-function.patch                  # 添加-s或者--signoff，还可以把自己的名字添加为signed off by信息，作用是注明打patch的人是谁，因为有时打patch的人并不是patch的作者
$ git am ~/patch-set/*.patch　　　　　　　　　　　　　# 将路径~/patch-set/*.patch 按照先后顺序打上
$ git am --abort                                                                   # 当git am失败时，用以将已经在am过程中打上的patch废弃掉(比如有三个patch，打到第三个patch时有冲突，那么这条命令会把打上的前两个patch丢弃掉，返回没有打patch的状态)
$ git am --resolved                                                             #当git am失败，解决完冲突后，这条命令会接着打patch
 
如果打Patch的过程中发生了冲突（conflicts），怎么办？
解决patch冲突的过程是：
如果不想打这一系列patch了，直接：git am --abort。
如果还想打, 有两种解决方案：
方案一（个人推荐）：
(1) 根据git am失败的信息，找到发生冲突的具体patch文件，然后用命令git apply --reject <patch_name>，强行打这个patch，发生冲突的部分会保存为.rej文件（例如发生冲突的文件是a.txt，那么运行完这个命令后，发生conflict的部分会保存为a.txt.rej），未发生冲突的部分会成功打上patch
(2) 根据.rej文件，通过编辑该patch文件的方式解决冲突。
(3) 废弃上一条am命令已经打了的patch：git am --abort
(4) 重新打patch：git am ~/patch-set/*.patchpatch