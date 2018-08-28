---
title:  git常用命令
date: 2018-06-24 12:59:58
tags:  git
categories: [Linux-Service ]
comments: true
copyright: true
---

## git常用命令



以下命令整理自廖雪峰的git笔记

[git笔记](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)

---

#### 一.基础命令.查看命令

git init  -------创建版本库

git add file  ---------添加git追踪文件

git status    ---------查看缓存区的文件

git stash    -----------保存当前缓存区修改文件(方便临时切换到其他分支,后续再切换回来)

git stash list  ---------查看保存的缓存区文件

<!--more-->

**恢复保存的缓存区文件有两条命令:**

- git stash apply -----------恢复过后stash保存的记录和内容并不删除,需要git stash drop手动删除
- git stash pop   -----------恢复stash保存的文件,同时自动删除保存记录(推荐)

git stash apply stash@{0} -------------指定恢复哪个缓存记录

git commit -m '更新说明'  --------提交修改到仓库

git diff        ----------查看文件差别

git log       -----------查看提交历史记录

git log -p -2  --------查看最近2次提交记录

git log --stat  -------查看提交历史记录简洁信息

git log --pretty=oneline ------查看提交历史记录---以每行显示

有关git log 显示格式,查询条件等其他更多用法,可以参考 pro git 中文版这本书

----

#### 二.回退版本

git reset --hard HEAD^   ------------回退到上一个版本,前提是没有提交到远程仓库(此时仓库里的文件内容会恢复到上个版本,相当于系统还原)

git reset --hard 版本号前7位  --------回退到具体某个版本号

回退到某次之前的提交..可以用git log 查看提交历史...使用 git reset --hard commit_id(版本号)跳转到某个版本

返回到之后的某次提交,可以用 git reflog查看提交历史.使用 git reset --hard commit_id(版本号)跳转到某个版本

----



#### 三.撤销文件修改

git checkout -- file    --------撤销对文件的修改,如果删除了工作区文件,则用这个命令同样可以恢复(如果文件修改后已经提交到缓存区,则不起作用)

git reset HEAD file   ---------撤销暂存区内即将要提交的文件 (对文件内容无影响)

git rm file  ------删除仓库里某个文件(当本地删除某个文件时,,可能你也需要删除仓库里这个文件)

------

#### 四.远程仓库:

git remote add origin git@github.com:path/github仓库名.git

比如:

git remote add origin git@github.come:jessehuang408/learngit.git

github.com----git远程服务器域名

jessehuang408/learngit -------仓库的路径,这里是指我的github账户下的learngit仓库

.git -----仓库后缀名

git remote -v   -----------查看远程仓库信息

git remote rm origin -------------删除本地仓库和远程仓库的关联

关联多个远程仓库时..由于默认的git给远程仓库起码是origin.所以如果有多个远程仓库,则需要用不同的标识来表示不同的远程仓库.例如:

git remote add github  git@github.com:path   ----关联github远程仓库.远程仓库名称是github

git remote add gitee   git@gitee.com:path    ----关联gitee远程仓库,远程仓库名称是gitee

git push github master ---表示推送到github远程仓库

git push gitee master  ----表示推送到gitee远程仓库

git pull origin master ----拉取远程数据到本地仓库

git push -u origin master 第一次提交master分支所有内容...后续提交只需要git push origin master

git clone  [git@github.com](mailto:git@github.com):path/github仓库名.git -------克隆远程仓库

此命令会在你的当前目录下产生一个仓库名的文件夹,这个文件夹就是你git的本地工作仓库

> 还可以用[https://](https://github.com/michaelliao/gitskills.git)[github.com](mailto:git@github.com):path/github仓库名.git这样的地址。实际上，Git支持多种协议，默认的git://使用ssh，但也可以使用https等其他协议。

使用https除了速度慢以外，还有个最大的麻烦是每次推送都必须输入口令，但是在某些只开放http端口的公司内部就无法使用ssh协议而只能用https

------

#### 五.分支命令 

git branch 分支名 ----创建分支

git checkout 分支名   ---切换分支

git checkout -b 分支名  ------创建切换分支...相当于上面2条命令合并

git branch   ------查看分支

git merge 分支名  ----合并某分支到当前分支

git branch -d 分支名  ------删除分支

git branch -D 分支名 -------强行删除一个未合并的分支

------

#### 六.合并命令

git merge 分支名   -----在当前分支下与某分支合并

git merge --no-ff -m '提交信息' 分支名 ----禁用fast forward模式.合并时候生成一个新的commit

------

#### 七.多人写作

git remote ---查看远程仓库信息

git remote -v ----查看远程仓库详细信息

git push origin master -----提交本地仓库到远程仓库

git push origin dev -------提交其他分支到远程仓库

git clone git@github.com:path -----克隆远程仓库到本地当前目录下

git branch  --------查看克隆过来的远程仓库的分支

git checkout -b 分支名 origin/分支名  ---------创建远程origin的某分支到本地.

git push origin 分支名  -------提交本地分支到远程分支

比如 git checkout -b dev origin/dev  创建远程dev分支到本地

git push origin dev -----提交本地dev的修改到远程dev

------

#### 八.标签命令

git tag 标签名                                                  ------在当前分支下.打伤标签,那么这次提交就会带上这个标签

git tag                                                             ------查看所有标签

git tag 标签名 commit_ID                             ------给之前某个提交记录打上标签

git show 标签名                                               -------查看某个标签信息

git tat -a 标签名 -m 说明文字  [commit_ID] --------给某次提交打上标签,并且附带说明信息

git tag -d 标签名                                              --------删除标签

git push origin 标签名                                   ---------推送标签到远程仓库

git push  origin --tags                                 -----------推送所有尚未推送的标签到远程仓库

**删除远程仓库标签的方法:** 

第一步:先本地删除

第二步:git push origin :refs/tags/标签名