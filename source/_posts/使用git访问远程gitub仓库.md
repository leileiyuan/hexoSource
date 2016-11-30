---
title: 使用git访问远程gitub仓库
date: 2016-10-22 21:39:43
tags:
  - git 
  - github
categories: 
  - git
---

在windows下安装`msysgit`，使用`git`命令操作仓库，并推送到远程`github`仓库。
 安装过以后，打开 `Git Bash`,会显示类型以下信息
 
	leileiyuan@08-201508200156 MINGW64 ~
	$

输入`git --version`命令，显示版本号。
<!--more-->

	$ git --version
	git version 2.9.3.windows.2

	 
 本地建立工作环境，提交到远程仓库中。最简步骤：

	makdir blog                         -- 创建一个项目blog  
	$ cd blog                           -- 打开这个项目
	$ git init                          -- 初始化
	$ touch README.md
	$ git add README.md                 -- 更新README文件
	$ git commit -m 'first commit'      -- 提交更新，并注释信息“first commit”
	$ git remote add origin https://github.com/leileiyuan/hexoSource.git     -- 连接远程github项目  
	$ git push -u origin master         -- 将本地项目推动到远程仓库

目录文件比较多时，可以一次性`add`多个文件及目录

	git add --all

 或者
 
	 git add -A
  
 
 从远程仓库中下载工程环境

	git clone https://github.com/leileiyuan/hexoSource.git blog
	git add .
	git commit -m "master commit"
	git push origin 
	
clone下来的分支是master，在本地建立分支dev，切换到远程分支dev上(origin/dev)

	git checkout -b dev origin/dev

**更新本地工程,合并工程**

	git pull <远程主机名> <远程分支名>:<本地分支名>
	git pull origin master:master

合并分支，将远程master合并到本地dev分支

	git pull origin master:dev

将远程分支与当前分支合并

	git pull origin dev

相当于
git fetch 和 git merge
先抓取远程数据，对比下有什么不同之处，再合并。这样操作相对安全一些。



**合并分支**
把branchName合并到当前分支

	git merge branchName

> 合并时，可能会有冲突。手动把有冲突的文件编辑以后，添加索引，再commit
> 有冲突(conflicts)的文件会保存在索引中，除非你解决了问题了并且更新了索引，否则执行 git commit都会失败



创建分支

	git branch branchName


切换分支

	git checkout brancnName


创建并切换到分支

	git checkout -b branchName


查看分支列表

	git branch 不带参数：列出所有分支
	gir branch -r 列出远程分支
	gir branch -a 列出本地和远程分支



提交到远程仓库

	git push <远程主机名> <本地分支名>:<远程分支名>
	git push origin dev

如果当前分支与远程分支之间存在追踪关系，则本地分支和远程分支都可以省略。

	$ git push origin


