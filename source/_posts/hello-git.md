---
title: Hello Git
tag: 工具
---

Git 入门笔记，带你快速使用 Git Bash 命令

# Git 入门

git 分布式版本控制

记录版本号 ，每个版本都有（高效的压缩解压算法）

## 本地库

本地结构

    1. 本地库
    1. 暂存区
    1. 工作区

#### git init

初始化本地库

#### git add **git commit**

工作区 -》**git add** 【】暂存区 -》**git commit** 【】 本地库

工作区 下（.git 同级）未 git add

缓存区 已经 git add 未 git commit

#### git status

查看工作状态

#### git log

提交日志（由近到远）

每一条内容 有一个 key 索引唯一对应

不同展示样式：

##### git log --pretty=oneline

##### git log --oneline

##### git reflog

#### git reset --【】索引

1.使用 hard（使用最多）

​ 本地库的指针移动的同时 同步工作区、暂存区、本地库

2.使用 mixed

​ 本地库的指针移动的同时 同步暂存区、本地库

2.使用 soft

​ 只会让本地移动

#### git diff

比较工作区和暂存区的差异 （带文件比文件 ，不带比所有）

### 分支

新建分支会先将主分支的最新版本然后加到分支

_各自分支相互独立_

#### git branch

##### git branch -v

​ 查看当前所有分支

##### git branch 【】

创建分支

##### git checkout 【】

切换分支

主分支合并 其他分支

1. 切换到主分支
2. 使用 git merge 【分支】（当主分支和其他分支 都修改了同一文件的同一位置就会冲突）
3. 解决冲突方法：直接文件选择性删除 （再添加 提交）

## 远程库

**github**

**gitee**

**gitlab**

#### git remote add 【name】【https://.....】

本地库起远程库别名

#### git remote -v

查看别名

**git push 【name】【分支】**

向远程仓库（别名）推送 本地仓库 的指定分支

**git clone 【https://.....】**

初始化本地库 从远程库克隆到本地 起了别名 origin

**库的拥有者拉取**

1. fetch +merge 操作

##### git fetch 【name】 【分支】

从远程库抓取到本地库，工作区不变 这时候本地有个分支是 **name/分支**

切换到本地 master 分支 调用 git merge name/分支 就可以合并

2. pull 操作

##### git pull【name】 【分支】

冲突产生 需要到本地 解决再 push 到 远程仓库

## 免密操作

$ ssh-keygen -t rsa -C【email】

### 常用操作

idea 远程 pull 前提准备

git pull origin master --allow-unrelated-histories

推送到上游其它分支

git push --set-upstream origin myblog（分支）
