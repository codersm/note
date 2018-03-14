---
title: linux
date: 2016-10-08 11:32:39
tags:  linux
---

常用linux命令行记录
<!-- more -->


#### grep查找某个关键字的前后几行
```shell
$grep -5 'parttern' inputfile //打印匹配行的前后5行

$grep -C 5 'parttern' inputfile //打印匹配行的前后5行

$grep -A 5 'parttern' inputfile //打印匹配行的后5行

$grep -B 5 'parttern' inputfile //打印匹配行的前5行
```
> 线上查找问题的时候，可以使用下列命令，查找关键字，显示关键字出现行的后5行，并且给关键字着色。  

```shell
$grep -A -15  --color 1010061938 *
```


linux书籍
- W3School Linux 教程 https://wizardforcel.gitbooks.io/w3school-linux/content/61.html
