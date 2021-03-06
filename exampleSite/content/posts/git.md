---
title: 【团队协作】GitHub WorkFlow
description:
toc: true
authors:
  - Example Author
tags: 
  - Git
  - Workflow
categories: 团队协作
series:
date: '2019-11-20T22:52:56+08:00'
lastmod: '2019-11-20T22:52:56+08:00'
featuredImage:
draft: false
---

# Git WorkFlow

## Step 1 : Fork

将目的项目fork一份到自己仓库中

``` shell
http://git.code.oa.com/mesh/ankyra
```

## Step 2：clone

将fork后的仓库clone下来

``` shell
git clone http://git.code.oa.com/bitliu/ankyra.git
```

## Step 3：添加上游

将项目地址设为upstream

``` shell
git remote add upstream http://git.code.oa.com/mesh/ankyra.git
```

更新上游（上游有更新时）

```shell
git fetch upstream
```

合并上游更新

``` shell
git merge upstream/指定分支
```

## Step 4：切换到工作分支

``` shell
git branch 
git checkout 工作分支
```

## Step 5：基于工作分支创建开发开发

```shell
git checkout -b dev-branch 工作分支
```

## Step 6：创建远程开发分支并ODP测试（可省略）

``` shell
git push origin/远程开发分支：本地开发分支名
```

## Step 7：工作分支合并开发分支并push到远端

```shell
git merge 开发分支 //开发分支add、commit之后合并
git push
```

## Step 8: 删除开发分支

``` shell
git branch -d 开发分支
```

