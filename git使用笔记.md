---
title: git使用笔记
date: 2016-08-19 11:05:54
tags: Git
---

<!-- Git 清理无效的远程追踪分支
在使用git进行版本控制时，经常会创建一些特性分支方便产品功能的开发和迭代，
在远程版本库创建了一个分支后，在本地可以使用
```
$ git remote update
```
可以在本地创建远程追踪分支，

但是，如果在远程版本库上删除了某一分支，该命令并不会删除本地的远程追踪分支，

这时候，有另一个命令
```
$ git remote prune
```
该命令可以删除本地版本库上那些失效的远程追踪分支，具体用法是，假如你的远程版本库名是 origin,则使用如下命令先查看哪些分支需要清理：
```
$ git remote prune origin --dry-run
```
可以看到， origin/a 和 origin/patch-1 两个远程分支已经失效，将会被清理，执行
```
$ git remote prune origin
```
这样，就完成了无效的远程追踪分支的清理工作。

需要注意，这里远程追踪分支批位于.git/refs/remote/origin 下的分支，如果有本地分支作为下游存在的话，还需要手动清理，从下图的命令可以看出，无效的远程追踪分支会以gone来标识

删除无效的本地分支命令：
```java

$ git branch -d bug fix feature master4 master5 master6 origin/master7 master 8
```

#### Download a specific tag with Git
$ git clone will give you the whole repository.
After the clone, you can list the tags with $ git tag -l and then checkout a specific tag:
```
$ git checkout tags/<tag_name>
```
Even better, checkout and create a branch (otherwise you will be on a branch named after the revision number of tag):
```
$ git checkout tags/<tag_name> -b <branch_name>
``` -->
